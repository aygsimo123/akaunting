pipeline {
    agent any

    options { skipDefaultCheckout(true) }

    environment {
        COMPOSER_NO_INTERACTION = "1"
        COMPOSER_CACHE_DIR      = "${WORKSPACE}\\.composer-cache"

        // Forcer Python Windows (éviter MSYS2)
        PY312   = "C:\\Program Files\\Python312\\python.exe"

        // Semgrep exe (installé par pip dans Scripts)
        SEMGREP = "C:\\Program Files\\Python312\\Scripts\\semgrep.exe"

        // PowerShell full path (évite: 'powershell' n'est pas reconnu)
        PSH = "C:\\Windows\\System32\\WindowsPowerShell\\v1.0\\powershell.exe"

        // ZAP path (ton installation)
        ZAP_HOME = "C:\\Program Files\\ZAP\\Zed Attack Proxy"

        // Cibles DAST (adapte si besoin)
        BACKEND_URL  = "http://127.0.0.1:8000"
        FRONTEND_URL = "http://127.0.0.1:5173"

        // IMPORTANT: Jenkins utilise 8080 => ZAP doit utiliser un autre port
        ZAP_PROXY_PORT = "18080"
    }

    stages {

        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Dependances PHP (Composer)') {
            steps {
                writeFile file: 'composer-install.cmd', text: (
                    "@echo on\r\n" +
                    "setlocal EnableExtensions\r\n" +
                    "cd /d \"%WORKSPACE%\"\r\n" +
                    "\r\n" +
                    "echo ===== WHERE PHP/COMPOSER =====\r\n" +
                    "where php\r\n" +
                    "where composer\r\n" +
                    "php -v\r\n" +
                    "call composer -V\r\n" +
                    "\r\n" +
                    "echo ===== COMPOSER CACHE DIR =====\r\n" +
                    "if not exist \"%COMPOSER_CACHE_DIR%\" mkdir \"%COMPOSER_CACHE_DIR%\"\r\n" +
                    "\r\n" +
                    "echo ===== RUN COMPOSER INSTALL =====\r\n" +
                    "call composer install --no-interaction --prefer-dist --no-progress\r\n" +
                    "set COMPOSER_EXIT=%ERRORLEVEL%\r\n" +
                    "echo COMPOSER_EXIT=%COMPOSER_EXIT%\r\n" +
                    "if not \"%COMPOSER_EXIT%\"==\"0\" exit /b %COMPOSER_EXIT%\r\n" +
                    "\r\n" +
                    "echo ===== CHECK VENDOR =====\r\n" +
                    "if not exist \"vendor\\autoload.php\" (\r\n" +
                    "  echo ERROR: vendor\\autoload.php introuvable apres composer install\r\n" +
                    "  dir\r\n" +
                    "  exit /b 1\r\n" +
                    ")\r\n" +
                    "echo ===== OK (vendor present) =====\r\n" +
                    "endlocal\r\n"
                )

                bat '''
                @echo on
                echo ===== RUN composer-install.cmd =====
                call composer-install.cmd
                if errorlevel 1 exit /b 1
                '''
            }
        }

        //  SAST (Semgrep) - non bloquant
        stage('SAST (Semgrep)') {
            steps {
                bat """
                @echo on
                chcp 65001 >nul
                set PYTHONUTF8=1
                set PYTHONIOENCODING=utf-8

                echo ===== PYTHON CHECK (FORCED PATH) =====
                "${env.PY312}" --version
                "${env.PY312}" -m pip --version

                echo ===== INSTALL/UPDATE SEMGREP =====
                "${env.PY312}" -m pip install --quiet --upgrade semgrep

                echo ===== CHECK SEMGREP EXE =====
                if not exist "${env.SEMGREP}" (
                  echo ERROR: semgrep.exe introuvable: ${env.SEMGREP}
                  dir "C:\\Program Files\\Python312\\Scripts"
                  exit /b 0
                )

                echo ===== RUN SEMGREP (non-bloquant) =====
                "${env.SEMGREP}" --config=auto --json --output semgrep-report.json
                "${env.SEMGREP}" --config=auto --output semgrep-report.txt

                exit /b 0
                """
            }
        }

        // ✅ SCA Composer (non-bloquant)
        stage('Securite (SCA)') {
            steps {
                bat '''
                @echo on
                cd /d "%WORKSPACE%"

                call composer audit --format=json > composer-audit.json
                call composer audit > composer-audit.txt

                findstr /I /R "CVE-[0-9][0-9][0-9][0-9]-[0-9]" composer-audit.txt > composer-cves.txt

                exit /b 0
                '''
            }
        }

        // ✅ npm install (non-bloquant, avec retry + gestion EPERM + logs)
        stage('npm - Install deps') {
            steps {
                writeFile file: 'npm-install.ps1', text: '''
$ErrorActionPreference = "Continue"
Set-Location $env:WORKSPACE

Write-Host "===== NODE / NPM ====="
node -v
npm -v

# Nettoyage flag précédent
if (Test-Path ".npm_failed") { Remove-Item ".npm_failed" -Force }

function Run-NpmInstall {
  param([int]$attempt)

  Write-Host "===== ATTEMPT $attempt ====="

  # Cleanup windows locks (EPERM)
  if (Test-Path "node_modules") {
    Write-Host "Cleaning node_modules (best effort)..."
    cmd /c "rmdir /s /q node_modules" | Out-Null
    Start-Sleep -Seconds 2
  }

  Write-Host "===== INSTALL DEPS ====="
  if (Test-Path "package-lock.json") {
    Write-Host "Running: npm ci"
    npm ci --no-fund --no-audit 2>&1 | Tee-Object -FilePath "npm-install.log" -Append
    return $LASTEXITCODE
  } else {
    Write-Host "Running: npm install"
    npm install --no-fund --no-audit 2>&1 | Tee-Object -FilePath "npm-install.log" -Append
    return $LASTEXITCODE
  }
}

$exit = Run-NpmInstall -attempt 1
if ($exit -ne 0) {
  Write-Host "WARNING: npm install failed (attempt 1). retry in 5s..."
  Start-Sleep -Seconds 5
  $exit = Run-NpmInstall -attempt 2
}

Write-Host "NPM_INSTALL_EXIT=$exit"

if ($exit -ne 0) {
  Write-Host "WARNING: npm install failed (non-bloquant). Frontend build + npm audit will be skipped."
  New-Item -ItemType File -Path ".npm_failed" -Force | Out-Null
  exit 0
}

if (!(Test-Path "node_modules\\.bin")) {
  Write-Host "WARNING: node_modules\\.bin missing (non-bloquant). Frontend build + npm audit will be skipped."
  New-Item -ItemType File -Path ".npm_failed" -Force | Out-Null
  exit 0
}

Write-Host "===== PROOF: node_modules\\.bin ====="
Get-ChildItem "node_modules\\.bin" | Select-Object -First 30 Name
exit 0
'''
                bat """
                @echo on
                "${env.PSH}" -NoProfile -ExecutionPolicy Bypass -File npm-install.ps1
                """
            }
        }

        // ✅ build mix (skippé si npm fail)
        stage('npm - Frontend build (Mix production)') {
            steps {
                writeFile file: 'npm-build.ps1', text: '''
$ErrorActionPreference = "Stop"
Set-Location $env:WORKSPACE

if (Test-Path ".npm_failed") {
  Write-Host "SKIP: npm failed earlier => skipping frontend build"
  exit 0
}

if (!(Test-Path "node_modules\\.bin")) {
  Write-Host "SKIP: node_modules\\.bin missing => skipping frontend build"
  exit 0
}

Write-Host "===== RUN PRODUCTION ====="
npm run production
if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }
exit 0
'''
                bat """
                @echo on
                "${env.PSH}" -NoProfile -ExecutionPolicy Bypass -File npm-build.ps1
                """
            }
        }

        // ✅ npm audit (skippé si npm fail) + non-bloquant
        stage('Securite (npm)') {
            steps {
                bat '''
                @echo on
                cd /d "%WORKSPACE%"

                if exist .npm_failed (
                  echo SKIP: npm install failed => skipping npm audit
                  exit /b 0
                )

                if not exist node_modules (
                  echo SKIP: node_modules missing => skipping npm audit
                  exit /b 0
                )

                call npm audit --audit-level=high > npm-audit.txt
                call npm audit --json > npm-audit.json

                exit /b 0
                '''
            }
        }

        // ✅ DAST (OWASP ZAP) - non-bloquant
        stage('DAST (OWASP ZAP)') {
            steps {
                bat """
                @echo on 
                setlocal EnableExtensions
                cd /d "%WORKSPACE%"

                set "ZAP=${env.ZAP_HOME}\\zap.bat"
                set "ZAP_WORK=%WORKSPACE%\\zap-work"

                taskkill /F /IM ZAP.exe >nul 2>&1
                taskkill /F /IM zap.exe >nul 2>&1

                if not exist "%ZAP_WORK%" mkdir "%ZAP_WORK%"
                cd /d "${env.ZAP_HOME}"

                call "%ZAP%" -cmd -dir "%ZAP_WORK%" -port ${env.ZAP_PROXY_PORT} ^
                  -quickurl "${env.BACKEND_URL}" ^
                  -quickout "%WORKSPACE%\\zap-backend.html"

                call "%ZAP%" -cmd -dir "%ZAP_WORK%" -port ${env.ZAP_PROXY_PORT} ^
                  -quickurl "${env.FRONTEND_URL}" ^
                  -quickout "%WORKSPACE%\\zap-frontend.html"

                dir "%WORKSPACE%\\zap-*.html"
                exit /b 0
                """
            }
        }

        stage('Tests (Laravel + SQLite)') {
            steps {
                writeFile file: 'jenkins-tests.ps1', text: '''
$ErrorActionPreference = "Stop"
Set-Location $env:WORKSPACE

if (!(Test-Path "vendor\\autoload.php")) { exit 1 }

if (!(Test-Path ".env")) {
  if (Test-Path ".env.example") { Copy-Item ".env.example" ".env" }
  else { throw ".env.example introuvable" }
}

if (!(Test-Path "database")) { New-Item -ItemType Directory -Path "database" | Out-Null }
if (!(Test-Path "database\\database.sqlite")) { New-Item -ItemType File -Path "database\\database.sqlite" | Out-Null }

Add-Content -Path ".env" -Value "`nAPP_ENV=testing"
Add-Content -Path ".env" -Value "APP_DEBUG=false"
Add-Content -Path ".env" -Value "DB_CONNECTION=sqlite"
Add-Content -Path ".env" -Value ("DB_DATABASE=" + $env:WORKSPACE + "\\database\\database.sqlite")
Add-Content -Path ".env" -Value "CACHE_DRIVER=array"
Add-Content -Path ".env" -Value "SESSION_DRIVER=array"
Add-Content -Path ".env" -Value "QUEUE_CONNECTION=sync"
Add-Content -Path ".env" -Value "MAIL_MAILER=log"

php artisan key:generate --force
if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }

php artisan migrate --force --no-interaction
if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }

if (!(Test-Path "build\\reports")) { New-Item -ItemType Directory -Path "build\\reports" | Out-Null }

if (Test-Path "vendor\\bin\\phpunit.bat") {
  cmd /c "vendor\\bin\\phpunit.bat --log-junit build\\reports\\junit.xml"
  if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }
} else {
  php artisan test --log-junit build\\reports\\junit.xml
  if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }
}
'''
                bat """
                @echo on
                "${env.PSH}" -NoProfile -ExecutionPolicy Bypass -File jenkins-tests.ps1
                if errorlevel 1 exit /b 1
                """
            }
        }
    }

    post {
        always {
            script {
                if (fileExists('build/reports/junit.xml')) {
                    junit testResults: 'build/reports/junit.xml', allowEmptyResults: true
                }
            }

            archiveArtifacts artifacts: 'build/reports/**', allowEmptyArchive: true
            archiveArtifacts artifacts: 'composer-audit.txt, composer-audit.json, composer-cves.txt', allowEmptyArchive: true
            archiveArtifacts artifacts: 'npm-audit.txt, npm-audit.json', allowEmptyArchive: true
            archiveArtifacts artifacts: 'npm-install.log', allowEmptyArchive: true
            archiveArtifacts artifacts: 'semgrep-report.json, semgrep-report.txt', allowEmptyArchive: true
            archiveArtifacts artifacts: 'zap-backend.html, zap-frontend.html', allowEmptyArchive: true
            archiveArtifacts artifacts: 'storage/logs/*.log', allowEmptyArchive: true

            cleanWs()
        }
    }
}
