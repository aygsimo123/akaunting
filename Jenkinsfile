    pipeline {
    agent any

    options {
        // Evite le double checkout ("Declarative: Checkout SCM" + ton stage Checkout)
        skipDefaultCheckout(true)
    }

    environment {
        COMPOSER_NO_INTERACTION = "1"
        COMPOSER_CACHE_DIR      = "${WORKSPACE}\\.composer-cache"
        APP_ENV                 = "testing"
        APP_DEBUG               = "false"
    }

    stages {
        stage('Checkout') {
        steps {
            checkout scm
        }
        }

        stage('Dependances PHP (Composer)') {
        steps {
            bat """
            php -v
            composer -V
            if not exist "%COMPOSER_CACHE_DIR%" mkdir "%COMPOSER_CACHE_DIR%"
            composer install --no-interaction --prefer-dist --no-progress
            """
        }
        }

        stage('Securite (SCA)') {
        steps {
            bat "composer audit"
        }
        }

        stage('Securite (npm) - optionnel') {
        when {
            expression { fileExists('package-lock.json') }
        }
        steps {
            bat """
            node -v
            npm -v
            npm ci
            npm audit --audit-level=high
            """
        }
        }

        stage('Tests (Laravel + SQLite)') {
        steps {
            bat """
            "C:\\Windows\\System32\\WindowsPowerShell\\v1.0\\powershell.exe" -NoProfile -ExecutionPolicy Bypass -Command ^
            "$ErrorActionPreference='Stop'; ^
            if (!(Test-Path '.env')) { if (Test-Path '.env.example') { Copy-Item '.env.example' '.env' } else { throw '.env.example introuvable' } }; ^
            if (!(Test-Path 'database')) { New-Item -ItemType Directory -Path 'database' | Out-Null }; ^
            if (!(Test-Path 'database\\database.sqlite')) { New-Item -ItemType File -Path 'database\\database.sqlite' | Out-Null }; ^
            Add-Content -Path '.env' -Value \"`nAPP_ENV=testing\"; ^
            Add-Content -Path '.env' -Value \"DB_CONNECTION=sqlite\"; ^
            Add-Content -Path '.env' -Value (\"DB_DATABASE=\" + $env:WORKSPACE + \"\\\\database\\\\database.sqlite\"); ^
            Add-Content -Path '.env' -Value \"CACHE_DRIVER=array\"; ^
            Add-Content -Path '.env' -Value \"SESSION_DRIVER=array\"; ^
            Add-Content -Path '.env' -Value \"QUEUE_CONNECTION=sync\"; ^
            Add-Content -Path '.env' -Value \"MAIL_MAILER=log\"; ^
            php artisan key:generate --force; ^
            php artisan config:clear; ^
            php artisan cache:clear; ^
            php artisan migrate --force --no-interaction; ^
            if (!(Test-Path 'build\\reports')) { New-Item -ItemType Directory -Path 'build\\reports' | Out-Null }; ^
            if (Test-Path 'vendor\\bin\\phpunit.bat') { cmd /c \"vendor\\bin\\phpunit.bat --log-junit build\\reports\\junit.xml\" } ^
            else { php artisan test --log-junit build\\reports\\junit.xml }"
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
        archiveArtifacts artifacts: 'build/reports/**, storage/logs/*.log', allowEmptyArchive: true
        cleanWs()
        }
    }
    }
