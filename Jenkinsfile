    pipeline {
    agent any

    options { skipDefaultCheckout(true) }

    environment {
        COMPOSER_NO_INTERACTION = "1"
        COMPOSER_CACHE_DIR      = "${WORKSPACE}\\.composer-cache"
    }

    stages {
        stage('Checkout') {
        steps { checkout scm }
        }

        stage('Dependances PHP (Composer)') {
        steps {
            // ✅ Ultra fiable: écrire un .cmd puis l'exécuter (avec CALL pour composer)
            writeFile file: 'composer-install.cmd', text: (
            "@echo on\r\n" +
            "setlocal EnableExtensions\r\n" +
            "cd /d \"%WORKSPACE%\"\r\n" +
            "\r\n" +
            "echo ===== WHERE PHP/COMPOSER =====\r\n" +
            "where php\r\n" +
            "where composer\r\n" +
            "php -v\r\n" +
            "call composer -V\r\n" +   // ✅ IMPORTANT: CALL
            "\r\n" +
            "echo ===== COMPOSER CACHE DIR =====\r\n" +
            "if not exist \"%COMPOSER_CACHE_DIR%\" mkdir \"%COMPOSER_CACHE_DIR%\"\r\n" +
            "\r\n" +
            "echo ===== RUN COMPOSER INSTALL =====\r\n" +
            "call composer install --no-interaction --prefer-dist --no-progress\r\n" + // ✅ CALL
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

        stage('Securite (SCA)') {
        steps {
            // ✅ CALL aussi ici (par sécurité)
            bat '''
    @echo on
    call composer audit
    if errorlevel 1 exit /b 1
    '''
        }
        }

        stage('Securite (npm) - optionnel') {
        when { expression { fileExists('package-lock.json') } }
        steps {
            // (optionnel) call npm pour éviter le même souci
            bat '''
    @echo on
    node -v
    npm -v

    echo ===== NPM CI =====
    call npm ci
    if errorlevel 1 exit /b 1

    echo ===== NPM AUDIT (high+) =====
    call npm audit --audit-level=high
    if errorlevel 1 exit /b 1
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
        archiveArtifacts artifacts: 'build/reports/**', allowEmptyArchive: true
        archiveArtifacts artifacts: 'storage/logs/*.log', allowEmptyArchive: true
        cleanWs()
        }
    }
    }
