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
                echo ===== SHOW composer-install.cmd =====
                type composer-install.cmd
                echo ===== RUN composer-install.cmd =====
                call composer-install.cmd
                if errorlevel 1 exit /b 1
                '''
            }
        }

        //  SAST (Semgrep) - non bloquant + Python forcé + semgrep.exe + FIX UTF-8
        stage('SAST (Semgrep)') {
            steps {
                bat """
                @echo on

                REM ===== Force UTF-8 (évite UnicodeEncodeError cp1252) =====
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
                  REM Non-bloquant
                  exit /b 0
                )

                echo ===== RUN SEMGREP (non-bloquant) =====
                "${env.SEMGREP}" --version

                REM Rapport JSON
                "${env.SEMGREP}" --config=auto --json --output semgrep-report.json

                REM Rapport TXT
                "${env.SEMGREP}" --config=auto --output semgrep-report.txt

                REM Toujours réussir ce stage
                exit /b 0
                """
            }
        }

        // ✅ SCA Composer (AVANCÉ) : txt + json + CVE list (non-bloquant)
        stage('Securite (SCA)') {
            steps {
                bat '''
                @echo on
                cd /d "%WORKSPACE%"

                echo ===== COMPOSER AUDIT (txt + json) =====
                call composer audit --format=json > composer-audit.json
                set AUDIT_JSON_EXIT=%ERRORLEVEL%

                call composer audit > composer-audit.txt
                set AUDIT_TXT_EXIT=%ERRORLEVEL%

                echo ===== RESUME SCA =====
                echo AUDIT_JSON_EXIT=%AUDIT_JSON_EXIT%
                echo AUDIT_TXT_EXIT=%AUDIT_TXT_EXIT%

                echo ===== CVE LIST (si existe) =====
                findstr /I /R "CVE-[0-9][0-9][0-9][0-9]-[0-9]" composer-audit.txt > composer-cves.txt
                if exist composer-cves.txt (
                  for /f %%A in ('type composer-cves.txt ^| find /c /v ""') do set CVE_COUNT=%%A
                ) else (
                  set CVE_COUNT=0
                )
                echo CVE_COUNT=%CVE_COUNT%

                echo ===== PREUVE (contenu audit) =====
                type composer-audit.txt

                REM non-bloquant pour la presentation
                exit /b 0
                '''
            }
        }

        // ==========================
        // ✅ NOUVEAU : npm install
        // ==========================
        stage('npm - Install deps') {
            when { expression { fileExists('package.json') } }
            steps {
                bat '''
                @echo on
                cd /d "%WORKSPACE%"

                echo ===== NODE / NPM =====
                node -v
                npm -v

                echo ===== INSTALL DEPS =====
                if exist "package-lock.json" (
                  echo package-lock.json present => npm ci
                  call npm ci --no-fund --no-audit > npm-ci.txt
                  set NPM_INSTALL_EXIT=%ERRORLEVEL%
                ) else (
                  echo package-lock.json absent => npm install
                  call npm install --no-fund --no-audit > npm-install.txt
                  set NPM_INSTALL_EXIT=%ERRORLEVEL%
                )

                echo NPM_INSTALL_EXIT=%NPM_INSTALL_EXIT%
                if not "%NPM_INSTALL_EXIT%"=="0" (
                  echo ERROR: npm install/ci failed
                  exit /b %NPM_INSTALL_EXIT%
                )

                echo ===== node_modules ready =====
                if not exist "node_modules" (
                  echo ERROR: node_modules introuvable apres installation
                  dir
                  exit /b 1
                )
                '''
            }
        }

        // ==========================
        // ✅ NOUVEAU : npm test/build
        // ==========================
        stage('npm - Frontend tests/build') {
            when { expression { fileExists('package.json') } }
            steps {
                bat '''
                @echo on
                cd /d "%WORKSPACE%"

                echo ===== RUN TEST =====
                call npm run test --silent
                set TEST_EXIT=%ERRORLEVEL%
                echo TEST_EXIT=%TEST_EXIT%

                echo ===== RUN BUILD =====
                call npm run build --silent
                set BUILD_EXIT=%ERRORLEVEL%
                echo BUILD_EXIT=%BUILD_EXIT%

                if not "%TEST_EXIT%"=="0" exit /b %TEST_EXIT%
                if not "%BUILD_EXIT%"=="0" exit /b %BUILD_EXIT%
                '''
            }
        }

        // ✅ npm (AVANCÉ) : audit seulement (non-bloquant), réutilise node_modules
        stage('Securite (npm)') {
            when { expression { fileExists('package.json') } }
            steps {
                bat '''
                @echo on
                cd /d "%WORKSPACE%"

                echo ===== CHECK node_modules =====
                if not exist "node_modules" (
                  echo WARNING: node_modules absent => npm install stage not executed or failed
                  REM non-bloquant
                  exit /b 0
                )

                echo ===== NODE / NPM =====
                node -v
                npm -v

                echo ===== NPM AUDIT (txt + json) =====
                call npm audit --audit-level=high > npm-audit.txt
                set NPM_AUDIT_TXT_EXIT=%ERRORLEVEL%

                call npm audit --json > npm-audit.json
                set NPM_AUDIT_JSON_EXIT=%ERRORLEVEL%

                echo ===== RESUME NPM =====
                echo NPM_AUDIT_TXT_EXIT=%NPM_AUDIT_TXT_EXIT%
                echo NPM_AUDIT_JSON_EXIT=%NPM_AUDIT_JSON_EXIT%

                type npm-audit.txt

                REM non-bloquant pour la presentation
                exit /b 0
                '''
            }
        }

        // ✅ DAST (OWASP ZAP) - STABLE + port dédié (18080) + kill ZAP avant scan + rapports HTML
        stage('DAST (OWASP ZAP)') {
            steps {
                bat """
                @echo on 
                setlocal EnableExtensions
                cd /d "%WORKSPACE%"

                set "ZAP=${env.ZAP_HOME}\\zap.bat"
                set "ZAP_WORK=%WORKSPACE%\\zap-work"

                REM Nettoyage: fermer ZAP si déjà lancé (évite Address already in use)
                taskkill /F /IM ZAP.exe >nul 2>&1
                taskkill /F /IM zap.exe >nul 2>&1

                if not exist "%ZAP_WORK%" mkdir "%ZAP_WORK%"

                cd /d "${env.ZAP_HOME}"

                echo ===== DAST BACKEND (quick scan -> HTML report) =====
                call "%ZAP%" -cmd -dir "%ZAP_WORK%" -port ${env.ZAP_PROXY_PORT} ^
                  -quickurl "${env.BACKEND_URL}" ^
                  -quickout "%WORKSPACE%\\zap-backend.html"
                echo ZAP_BACKEND_EXIT=%ERRORLEVEL%

                echo ===== DAST FRONTEND (quick scan -> HTML report) =====
                call "%ZAP%" -cmd -dir "%ZAP_WORK%" -port ${env.ZAP_PROXY_PORT} ^
                  -quickurl "${env.FRONTEND_URL}" ^
                  -quickout "%WORKSPACE%\\zap-frontend.html"
                echo ZAP_FRONTEND_EXIT=%ERRORLEVEL%

                echo ===== LIST REPORTS (preuve) =====
                dir "%WORKSPACE%\\zap-*.html"

                REM non-bloquant pour la presentation
                exit /b 0
                """
            }
        }

        stage('Tests (Laravel + SQLite)') {
            steps {
                writeFile file: 'jenkins-tests.ps1', text: '''
$ErrorActionPreference = "Stop"

if (!(Test-Path "vendor\\autoload.php")) {
  Write-Host "ERROR: vendor\\autoload.php introuvable. Composer install n'a pas ete fait ou a echoue."
  exit 1
}

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

php artisan config:clear
if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }

php artisan cache:clear
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

            // Tests + rapports
            archiveArtifacts artifacts: 'build/reports/**', allowEmptyArchive: true

            // ✅ npm build artifacts (si exist)
            archiveArtifacts artifacts: 'dist/**', allowEmptyArchive: true

            //  Security reports (PHP + JS)
            archiveArtifacts artifacts: 'composer-audit.txt, composer-audit.json, composer-cves.txt, npm-ci.txt, npm-install.txt, npm-audit.txt, npm-audit.json', allowEmptyArchive: true

            //  Semgrep reports
            archiveArtifacts artifacts: 'semgrep-report.json, semgrep-report.txt', allowEmptyArchive: true

            //  DAST reports
            archiveArtifacts artifacts: 'zap-backend.html, zap-frontend.html', allowEmptyArchive: true

            archiveArtifacts artifacts: 'storage/logs/*.log', allowEmptyArchive: true
            cleanWs()
        }
    }
}
