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

          //  SAST (Semgrep) - non bloquant + Python forcé + semgrep.exe + FIX UTF-8
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

          // ==========================
          // ✅ npm install (FORCÉ, sans vérification package.json) + preuve node_modules\.bin
          // ==========================
          stage('npm - Install deps') {
              steps {
                  writeFile file: 'npm-install.cmd', text: (
                      "@echo on\r\n" +
                      "setlocal EnableExtensions\r\n" +
                      "cd /d \"%WORKSPACE%\"\r\n" +
                      "\r\n" +
                      "echo ===== NODE / NPM =====\r\n" +
                      "node -v\r\n" +
                      "npm -v\r\n" +
                      "\r\n" +
                      "echo ===== LIST ROOT (preuve) =====\r\n" +
                      "dir\r\n" +
                      "\r\n" +
                      "echo ===== CLEAN node_modules =====\r\n" +
                      "if exist node_modules rmdir /s /q node_modules\r\n" +
                      "\r\n" +
                      "echo ===== INSTALL DEPS =====\r\n" +
                      "if exist package-lock.json (\r\n" +
                      "  echo Running: npm ci\r\n" +
                      "  call npm ci --no-fund --no-audit\r\n" +
                      ") else (\r\n" +
                      "  echo Running: npm install\r\n" +
                      "  call npm install --no-fund --no-audit\r\n" +
                      ")\r\n" +
                      "set NPM_INSTALL_EXIT=%ERRORLEVEL%\r\n" +
                      "echo NPM_INSTALL_EXIT=%NPM_INSTALL_EXIT%\r\n" +
                      "if not \"%NPM_INSTALL_EXIT%\"==\"0\" exit /b %NPM_INSTALL_EXIT%\r\n" +
                      "\r\n" +
                      "echo ===== PROOF: node_modules\\\\.bin =====\r\n" +
                      "if not exist \"node_modules\\\\.bin\" (\r\n" +
                      "  echo ERROR: node_modules\\\\.bin missing\r\n" +
                      "  dir\r\n" +
                      "  exit /b 1\r\n" +
                      ")\r\n" +
                      "dir node_modules\\\\.bin\r\n" +
                      "\r\n" +
                      "endlocal\r\n"
                  )

                  bat '''
                  @echo on
                  call npm-install.cmd
                  if errorlevel 1 exit /b 1
                  '''
              }
          }

          // ==========================
          // ✅ Frontend build (Akaunting = Laravel Mix) : npm run production
          // ==========================
          stage('npm - Frontend build (Mix production)') {
              steps {
                  writeFile file: 'npm-build.cmd', text: (
                      "@echo on\r\n" +
                      "setlocal EnableExtensions\r\n" +
                      "cd /d \"%WORKSPACE%\"\r\n" +
                      "\r\n" +
                      "echo ===== CHECK node_modules\\\\.bin =====\r\n" +
                      "if not exist \"node_modules\\\\.bin\" (\r\n" +
                      "  echo ERROR: node_modules\\\\.bin missing => npm install failed\r\n" +
                      "  exit /b 1\r\n" +
                      ")\r\n" +
                      "\r\n" +
                      "echo ===== RUN PRODUCTION (mix --production) =====\r\n" +
                      "call npm run production\r\n" +
                      "set BUILD_EXIT=%ERRORLEVEL%\r\n" +
                      "echo BUILD_EXIT=%BUILD_EXIT%\r\n" +
                      "if not \"%BUILD_EXIT%\"==\"0\" exit /b %BUILD_EXIT%\r\n" +
                      "\r\n" +
                      "endlocal\r\n"
                  )

                  bat '''
                  @echo on
                  call npm-build.cmd
                  if errorlevel 1 exit /b 1
                  '''
              }
          }

          // ✅ npm audit (non-bloquant)
          stage('Securite (npm)') {
              steps {
                  bat '''
                  @echo on
                  cd /d "%WORKSPACE%"

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
              archiveArtifacts artifacts: 'composer-audit.txt, composer-audit.json, composer-cves.txt, npm-audit.txt, npm-audit.json', allowEmptyArchive: true
              archiveArtifacts artifacts: 'semgrep-report.json, semgrep-report.txt', allowEmptyArchive: true
              archiveArtifacts artifacts: 'zap-backend.html, zap-frontend.html', allowEmptyArchive: true
              archiveArtifacts artifacts: 'storage/logs/*.log', allowEmptyArchive: true

              cleanWs()
          }
      }
  }
