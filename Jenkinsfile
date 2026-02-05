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

        stage('Check Node Version') {
            steps {
                bat 'node -v'
                bat 'npm -v'
            }
        }

        stage('Clean old npm modules') {
            steps {
                bat '''
                @echo on
                cd /d "%WORKSPACE%"

                if exist node_modules rmdir /s /q node_modules
                if exist package-lock.json del package-lock.json

                exit /b 0
                '''
            }
        }

        stage('Dependances PHP (Composer)') {
            steps {
                bat '''
                @echo on
                cd /d "%WORKSPACE%"

                if not exist "%COMPOSER_CACHE_DIR%" mkdir "%COMPOSER_CACHE_DIR%"

                call composer install --no-interaction --prefer-dist --no-progress

                if not exist "vendor\\autoload.php" exit /b 1
                '''
            }
        }

        stage('SAST (Semgrep)') {
            steps {
                bat """
                @echo on
                chcp 65001 >nul

                "${env.PY312}" -m pip install --quiet --upgrade semgrep

                if exist "${env.SEMGREP}" (
                    "${env.SEMGREP}" --config=auto --json --output semgrep-report.json
                    "${env.SEMGREP}" --config=auto --output semgrep-report.txt
                )

                exit /b 0
                """
            }
        }

        stage('Securite (SCA)') {
            steps {
                bat '''
                @echo on
                cd /d "%WORKSPACE%"

                call composer audit --format=json > composer-audit.json
                call composer audit > composer-audit.txt

                exit /b 0
                '''
            }
        }

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

cmd /c "npm install --no-fund --no-audit >> `"$log`" 2>&1"

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

        stage('npm - Frontend build') {
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

        stage('Securite (npm)') {
            steps {
                bat '''
                @echo on
                cd /d "%WORKSPACE%"

                if exist .npm_failed exit /b 0
                if not exist node_modules exit /b 0

                call npm audit > npm-audit.txt
                call npm audit --json > npm-audit.json

                exit /b 0
                '''
            }
        }

        stage('DAST (OWASP ZAP)') {
            steps {
                bat """
                @echo on
                cd /d "%WORKSPACE%"

                set "ZAP=${env.ZAP_HOME}\\zap.bat"
                set "ZAP_WORK=%WORKSPACE%\\zap-work"

                if not exist "%ZAP_WORK%" mkdir "%ZAP_WORK%"

                call "%ZAP%" -cmd -dir "%ZAP_WORK%" -port ${env.ZAP_PROXY_PORT} ^
                  -quickurl "${env.BACKEND_URL}" ^
                  -quickout "%WORKSPACE%\\zap-backend.html"

                exit /b 0
                """
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: '*.json, *.txt, *.html, npm-install.log', allowEmptyArchive: true
            cleanWs()
        }
    }
}
