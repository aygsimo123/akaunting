    pipeline {
        agent any

        options { skipDefaultCheckout(true) }

        environment {
            COMPOSER_NO_INTERACTION = "1"
            COMPOSER_CACHE_DIR      = "${WORKSPACE}\\.composer-cache"

            // ✅ Forcer Python Windows (éviter MSYS2)
            PY312   = "C:\\Program Files\\Python312\\python.exe"

            // ✅ Semgrep exe (installé par pip dans Scripts)
            SEMGREP = "C:\\Program Files\\Python312\\Scripts\\semgrep.exe"
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

            // ✅ SAST (Semgrep) - non bloquant + Python forcé + semgrep.exe + FIX UTF-8
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

            // ✅ npm (AVANCÉ) : déclenché par package.json + auto install + audit txt/json (non-bloquant)
            stage('Securite (npm)') {
                when { expression { fileExists('package.json') } }
                steps {
                    bat '''
                    @echo on
                    cd /d "%WORKSPACE%"

                    echo ===== NODE / NPM =====
                    node -v
                    npm -v

                    echo ===== INSTALL DEPS (auto) =====
                    if exist "package-lock.json" (
                    echo package-lock.json present => npm ci
                    call npm ci > npm-ci.txt
                    set NPM_INSTALL_EXIT=%ERRORLEVEL%
                    ) else (
                    echo package-lock.json absent => npm install (genere lockfile)
                    call npm install --no-fund --no-audit > npm-install.txt
                    set NPM_INSTALL_EXIT=%ERRORLEVEL%
                    )
                    echo NPM_INSTALL_EXIT=%NPM_INSTALL_EXIT%

                    if not "%NPM_INSTALL_EXIT%"=="0" (
                    echo WARNING: npm install/ci failed (non-bloquant)
                    exit /b 0
                    )

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

                    bat '''
                    @echo on
                    "C:\\Windows\\System32\\WindowsPowerShell\\v1.0\\powershell.exe" -NoProfile -ExecutionPolicy Bypass -File jenkins-tests.ps1
                    if errorlevel 1 exit /b 1
                    '''
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

                // ✅ Security reports (PHP + JS)
                archiveArtifacts artifacts: 'composer-audit.txt, composer-audit.json, composer-cves.txt, npm-ci.txt, npm-install.txt, npm-audit.txt, npm-audit.json', allowEmptyArchive: true

                // ✅ Semgrep reports
                archiveArtifacts artifacts: 'semgrep-report.json, semgrep-report.txt', allowEmptyArchive: true

                archiveArtifacts artifacts: 'storage/logs/*.log', allowEmptyArchive: true
                cleanWs()
            }
        }
    }
