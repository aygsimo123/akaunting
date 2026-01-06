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
            // Version "verifiable" + échoue si vendor/autoload.php n'existe pas
            bat '''
            @echo on
            php -v
            composer -V

            if not exist "%COMPOSER_CACHE_DIR%" mkdir "%COMPOSER_CACHE_DIR%"

            composer install --no-interaction --prefer-dist --no-progress
            if errorlevel 1 exit /b 1

            if not exist "vendor\\autoload.php" (
                echo ERROR: vendor\\autoload.php introuvable. composer install n'a pas genere vendor/.
                dir
                exit /b 1
            )
            '''
        }
        }

        stage('Securite (SCA)') {
        steps {
            // Echoue si vulnérabilités (exit code non-zero)
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
            bat '''
            @echo on
            node -v
            npm -v
            npm ci
            if errorlevel 1 exit /b 1
            npm audit --audit-level=high
            if errorlevel 1 exit /b 1
            '''
        }
        }

        stage('Tests (Laravel + SQLite)') {
        steps {
            // Script .ps1 (évite les problèmes Groovy avec $ErrorActionPreference)
            writeFile file: 'jenkins-tests.ps1', text: '''
    $ErrorActionPreference = "Stop"

    # Sécurité: s'assurer que vendor existe (sinon artisan va planter)
    if (!(Test-Path "vendor\\autoload.php")) {
    Write-Host "ERROR: vendor\\autoload.php introuvable. Composer install n'a pas été fait ou a échoué."
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

            // Exécution PowerShell par chemin complet (fiable sous service Jenkins)
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

        // Séparer pour éviter les warnings "ne correspond à rien"
        archiveArtifacts artifacts: 'build/reports/**', allowEmptyArchive: true
        archiveArtifacts artifacts: 'storage/logs/*.log', allowEmptyArchive: true

        cleanWs()
        }
    }
    }
