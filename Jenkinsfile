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
            // ✅ Ultra fiable: écrire un .cmd puis l'exécuter
            writeFile file: 'composer-install.cmd', text: '''
    @echo on
    php -v
    composer -V

    if not exist "%COMPOSER_CACHE_DIR%" mkdir "%COMPOSER_CACHE_DIR%"

    echo ===== RUN COMPOSER INSTALL =====
    composer install --no-interaction --prefer-dist --no-progress
    if errorlevel 1 exit /b 1

    echo ===== CHECK VENDOR =====
    if not exist "vendor\\autoload.php" (
    echo ERROR: vendor\\autoload.php introuvable apres composer install
    dir
    exit /b 1
    )

    echo ===== OK (vendor present) =====
    '''
            bat 'call composer-install.cmd'
        }
        }

        stage('Securite (SCA)') {
        steps {
            bat '''
    @echo on
    composer audit
    if errorlevel 1 exit /b 1
    '''
        }
        }

        stage('Securite (npm) - optionnel') {
        when { expression { fileExists('package-lock.json') } }
        steps {
            // ✅ Ici aussi on force l’échec si npm échoue
            bat '''
    @echo on
    node -v
    npm -v

    echo ===== NPM CI =====
    npm ci
    if errorlevel 1 exit /b 1

    echo ===== NPM AUDIT (high+) =====
    npm audit --audit-level=high
    if errorlevel 1 exit /b 1
    '''
        }
        }

        stage('Tests (Laravel + SQLite)') {
        steps {
            // ✅ Script .ps1 (évite soucis Groovy avec $ErrorActionPreference)
            writeFile file: 'jenkins-tests.ps1', text: '''
    $ErrorActionPreference = "Stop"

    # S'assurer que vendor existe (sinon artisan va planter)
    if (!(Test-Path "vendor\\autoload.php")) {
    Write-Host "ERROR: vendor\\autoload.php introuvable. Composer install n'a pas ete fait ou a echoue."
    exit 1
    }

    # .env
    if (!(Test-Path ".env")) {
    if (Test-Path ".env.example") { Copy-Item ".env.example" ".env" }
    else { throw ".env.example introuvable" }
    }

    # SQLite
    if (!(Test-Path "database")) { New-Item -ItemType Directory -Path "database" | Out-Null }
    if (!(Test-Path "database\\database.sqlite")) { New-Item -ItemType File -Path "database\\database.sqlite" | Out-Null }

    # Override env (testing)
    Add-Content -Path ".env" -Value "`nAPP_ENV=testing"
    Add-Content -Path ".env" -Value "APP_DEBUG=false"
    Add-Content -Path ".env" -Value "DB_CONNECTION=sqlite"
    Add-Content -Path ".env" -Value ("DB_DATABASE=" + $env:WORKSPACE + "\\database\\database.sqlite")
    Add-Content -Path ".env" -Value "CACHE_DRIVER=array"
    Add-Content -Path ".env" -Value "SESSION_DRIVER=array"
    Add-Content -Path ".env" -Value "QUEUE_CONNECTION=sync"
    Add-Content -Path ".env" -Value "MAIL_MAILER=log"

    # Artisan + checks exit code
    php artisan key:generate --force
    if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }

    php artisan config:clear
    if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }

    php artisan cache:clear
    if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }

    # Migrations
    php artisan migrate --force --no-interaction
    if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }

    # Reports dir
    if (!(Test-Path "build\\reports")) { New-Item -ItemType Directory -Path "build\\reports" | Out-Null }

    # PHPUnit / artisan test
    if (Test-Path "vendor\\bin\\phpunit.bat") {
    cmd /c "vendor\\bin\\phpunit.bat --log-junit build\\reports\\junit.xml"
    if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }
    } else {
    php artisan test --log-junit build\\reports\\junit.xml
    if ($LASTEXITCODE -ne 0) { exit $LASTEXITCODE }
    }
    '''

            // ✅ Appel PowerShell par chemin complet (fiable en service)
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
