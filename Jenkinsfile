  pipeline {
      agent any

      options { skipDefaultCheckout(true) }

      environment {
          COMPOSER_NO_INTERACTION = "1"
          COMPOSER_CACHE_DIR      = "${WORKSPACE}\\.composer-cache"

          PY312   = "C:\\Program Files\\Python312\\python.exe"
          SEMGREP = "C:\\Program Files\\Python312\\Scripts\\semgrep.exe"
          PSH = "C:\\Windows\\System32\\WindowsPowerShell\\v1.0\\powershell.exe"

          ZAP_HOME = "C:\\Program Files\\ZAP\\Zed Attack Proxy"

          BACKEND_URL  = "http://127.0.0.1:8000"
          FRONTEND_URL = "http://127.0.0.1:5173"

          ZAP_PROXY_PORT = "18080"
      }

      stages {

          stage('Checkout') {
              steps { checkout scm }
          }

          /* =========================
            COMPOSER
          ========================== */
          stage('Dependances PHP (Composer)') {
              steps {
                  writeFile file: 'composer-install.cmd', text: (
                      "@echo on\r\n" +
                      "setlocal EnableExtensions\r\n" +
                      "cd /d \"%WORKSPACE%\"\r\n" +
                      "where php\r\n" +
                      "where composer\r\n" +
                      "php -v\r\n" +
                      "call composer -V\r\n" +
                      "if not exist \"%COMPOSER_CACHE_DIR%\" mkdir \"%COMPOSER_CACHE_DIR%\"\r\n" +
                      "call composer install --no-interaction --prefer-dist --no-progress\r\n" +
                      "set COMPOSER_EXIT=%ERRORLEVEL%\r\n" +
                      "if not \"%COMPOSER_EXIT%\"==\"0\" exit /b %COMPOSER_EXIT%\r\n" +
                      "if not exist \"vendor\\autoload.php\" exit /b 1\r\n" +
                      "endlocal\r\n"
                  )

                  bat '''
                  call composer-install.cmd
                  if errorlevel 1 exit /b 1
                  '''
              }
          }

          /* =========================
            SAST
          ========================== */
          stage('SAST (Semgrep)'){
              steps {
                  bat """
                  "${env.PY312}" -m pip install --quiet --upgrade semgrep
                  if exist "${env.SEMGREP}" (
                    "${env.SEMGREP}" --config=auto --json --output semgrep-report.json
                    "${env.SEMGREP}" --config=auto --output semgrep-report.txt
                  )
                  exit /b 0
                  """
              }
          }

          /* =========================
            SCA COMPOSER
          ========================== */
          stage('Securite (SCA)') {
              steps {
                  bat '''
                  call composer audit --format=json > composer-audit.json
                  call composer audit > composer-audit.txt
                  exit /b 0
                  '''
              }
          }

          /* =========================
            FRONTEND INSTALL (STABLE)
          ========================== */
          stage('npm - Install deps') {
              steps {
                writeFile file: 'npm-install.ps1', text: '''
  $ErrorActionPreference = "Continue"
  Set-Location $env:WORKSPACE

  if (!(Test-Path "package.json")) { exit 0 }

  cmd /c "node -v"
  cmd /c "npm -v"

  $env:PYTHON = "C:\\Program Files\\Python312\\python.exe"
  $env:npm_config_python = $env:PYTHON

  if (Test-Path ".npm_failed") { Remove-Item ".npm_failed" -Force }

  $log = Join-Path $env:WORKSPACE "npm-install.log"
  if (Test-Path $log) { Remove-Item $log -Force }

  cmd /c "npm ci --no-fund --no-audit >> `"$log`" 2>&1"

  if ($LASTEXITCODE -ne 0) {
    New-Item -ItemType File -Path ".npm_failed" -Force | Out-Null
  }

  exit 0
  '''
                bat """
                "${env.PSH}" -NoProfile -ExecutionPolicy Bypass -File npm-install.ps1 || exit /b 0
                exit /b 0
                """
              }
          }

          /* =========================
            FRONTEND BUILD
          ========================== */
          stage('npm - Frontend build (Mix production)') {
              steps {
                writeFile file: 'npm-build.ps1', text: '''
  $ErrorActionPreference = "Continue"
  Set-Location $env:WORKSPACE

  if (!(Test-Path "package.json")) { exit 0 }
  if (Test-Path ".npm_failed") { exit 0 }
  if (!(Test-Path "node_modules\\.bin")) { exit 0 }

  cmd /c "npm run production"
  exit 0
  '''
                bat """
                "${env.PSH}" -NoProfile -ExecutionPolicy Bypass -File npm-build.ps1 || exit /b 0
                exit /b 0
                """
              }
          }

          /* =========================
            NPM AUDIT
          ========================== */
          stage('Securite (npm)') {
              steps {
                bat '''
                if not exist package.json exit /b 0
                if exist .npm_failed exit /b 0
                if not exist node_modules exit /b 0

                call npm audit --audit-level=high > npm-audit.txt
                call npm audit --json > npm-audit.json

                exit /b 0
                '''
              }
          }

          /* =========================
            DAST
          ========================== */
          stage('DAST (OWASP ZAP)') {
              steps {
                  bat """
                  set "ZAP=${env.ZAP_HOME}\\zap.bat"
                  call "%ZAP%" -cmd -quickurl "${env.BACKEND_URL}" ^
                      -quickout "%WORKSPACE%\\zap-backend.html"
                  exit /b 0
                  """
              }
          }
      }

      post {
          always {
              archiveArtifacts artifacts: 'npm-install.log', allowEmptyArchive: true
              archiveArtifacts artifacts: 'npm-audit.txt, npm-audit.json', allowEmptyArchive: true
              archiveArtifacts artifacts: 'semgrep-report.json, semgrep-report.txt', allowEmptyArchive: true
              archiveArtifacts artifacts: 'zap-backend.html', allowEmptyArchive: true
              cleanWs()
          }
      }
  }
