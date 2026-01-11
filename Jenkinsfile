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

            // ✅ NEW: Installer + builder le frontend (Akaunting assets)
            stage('Dependances Frontend (npm + build)') {
                when { expression { fileExists('package.json') } }
                steps {
                    bat '''
                    @echo on
                    cd /d "%WORKSPACE%"

                    echo ===== NODE / NPM =====
                    node -v
                    npm -v

                    echo ===== INSTALL FRONTEND DEPS =====
                    if exist "package-lock.json" (
                    echo package-lock.json present => npm ci
                    call npm ci
                    ) else (
                    echo package-lock.json absent => npm install
                    call npm install --no-fund
                    )
                    if errorlevel 1 exit /b 1

                    echo ===== CHECK NPM SCRIPTS =====
                    call npm run | findstr /I "build"
                    if errorlevel 1 (
                    echo INFO: script "build" not found -> skip build
                    exit /b 0
                    )

                    echo ===== RUN npm run build =====
                    call npm run build
                    if errorlevel 1 exit /b 1

                    echo ===== OK FRONTEND BUILD =====
                    '''
                }
            }

            stage('SAST (Semgrep)') {
                steps {
                    bat """
                    @echo on
                    chcp 65001 >nul
                    set PYTHONUTF8=1
                    set PYTHONIOENCODING=utf-8

                    "${env.PY312}" --version
                    "${env.PY312}" -m pip --version
                    "${env.PY312}" -m pip install --quiet --upgrade semgrep

                    if not exist "${env.SEMGREP}" (
                    echo ERROR: semgrep.exe introuvable: ${env.SEMGREP}
                    dir "C:\\Program Files\\Python312\\Scripts"
                    exit /b 0
                    )

                    "${env.SEMGREP}" --config=auto --json --output semgrep-report.json
                    "${env.SEMGREP}" --config=auto --output semgrep-report.txt
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

                    findstr /I /R "CVE-[0-9][0-9][0-9][0-9]-[0-9]" composer-audit.txt > composer-cves.txt

                    type composer-audit.txt
                    exit /b 0
                    '''
                }
            }

            stage('Securite (npm)') {
                when { expression { fileExists('package.json') } }
                steps {
                    bat '''
                    @echo on
                    cd /d "%WORKSPACE%"

                    node -v
                    npm -v

                    if exist "package-lock.json" (
                    call npm ci > npm-ci.txt
                    ) else (
                    call npm install --no-fund --no-audit > npm-install.txt
                    )

                    call npm audit --audit-level=high > npm-audit.txt
                    call npm audit --json > npm-audit.json

                    type npm-audit.txt
                    exit /b 0
                    '''
                }
            }

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
                    echo ZAP_BACKEND_EXIT=%ERRORLEVEL%

                    call "%ZAP%" -cmd -dir "%ZAP_WORK%" -port ${env.ZAP_PROXY_PORT} ^
                    -quickurl "${env.FRONTEND_URL}" ^
                    -quickout "%WORKSPACE%\\zap-frontend.html"
                    echo ZAP_FRONTEND_EXIT=%ERRORLEVEL%

                    dir "%WORKSPACE%\\zap-*.html"
                    exit /b 0
                    """
                }
            }

            stage('Tests (Laravel + SQLite)') {
                steps {
                    writeFile file: 'jenkins-tests.ps1', text: '''
    $ErrorActionPreference = "Stop"

    if (!(Test-Path "vendor\\autoload.php")) { Write-Host "ERROR: vendor\\autoload.php missing"; exit 1 }

    if (!(Test-Path ".env")) {
    if (Test-Path ".env.example") { Copy-Item ".env.example" ".env" } else { throw ".env.example missing" }
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

                archiveArtifacts artifacts: 'build/reports/**', allowEmptyArchive: true
                archiveArtifacts artifacts: 'composer-audit.txt, composer-audit.json, composer-cves.txt, npm-ci.txt, npm-install.txt, npm-audit.txt, npm-audit.json', allowEmptyArchive: true
                archiveArtifacts artifacts: 'semgrep-report.json, semgrep-report.txt', allowEmptyArchive: true
                archiveArtifacts artifacts: 'zap-backend.html, zap-frontend.html', allowEmptyArchive: true
                archiveArtifacts artifacts: 'storage/logs/*.log', allowEmptyArchive: true
                cleanWs()
            }
        }
    }
