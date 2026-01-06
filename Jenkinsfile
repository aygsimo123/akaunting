    pipeline {
    agent any

    environment {
        // Pour stabiliser les chemins/cache (optionnel mais utile)
        COMPOSER_NO_INTERACTION = "1"
        COMPOSER_CACHE_DIR      = "${WORKSPACE}\\.composer-cache"
        // Laravel test env
        APP_ENV = "testing"
        APP_DEBUG = "false"
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
            // Echoue si vulnérabilités (exit code non-zero)
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
            powershell """
            \$ErrorActionPreference = 'Stop'

            # 1) .env
            if (-not (Test-Path '.env')) {
                if (Test-Path '.env.example') {
                Copy-Item '.env.example' '.env'
                } else {
                throw '.env.example introuvable (impossible de créer .env)'
                }
            }

            # 2) SQLite file
            if (-not (Test-Path 'database')) { New-Item -ItemType Directory -Path 'database' | Out-Null }
            if (-not (Test-Path 'database\\database.sqlite')) { New-Item -ItemType File -Path 'database\\database.sqlite' | Out-Null }

            # 3) Forcer config test minimale via .env (on ajoute à la fin => override)
            Add-Content -Path '.env' -Value \"`nAPP_ENV=testing\"
            Add-Content -Path '.env' -Value \"APP_DEBUG=false\"
            Add-Content -Path '.env' -Value \"DB_CONNECTION=sqlite\"
            Add-Content -Path '.env' -Value \"DB_DATABASE=${env:WORKSPACE}\\\\database\\\\database.sqlite\"
            Add-Content -Path '.env' -Value \"CACHE_DRIVER=array\"
            Add-Content -Path '.env' -Value \"SESSION_DRIVER=array\"
            Add-Content -Path '.env' -Value \"QUEUE_CONNECTION=sync\"
            Add-Content -Path '.env' -Value \"MAIL_MAILER=log\"

            # 4) Key + clear
            php artisan key:generate --force
            php artisan config:clear
            php artisan cache:clear

            # 5) Migrations (si votre suite en a besoin)
            php artisan migrate --force --no-interaction

            # 6) PHPUnit + rapport JUnit si possible
            if (-not (Test-Path 'build\\reports')) { New-Item -ItemType Directory -Path 'build\\reports' | Out-Null }

            if (Test-Path 'vendor\\bin\\phpunit.bat') {
                cmd /c \"vendor\\bin\\phpunit.bat --log-junit build\\reports\\junit.xml\"
            } elseif (Test-Path 'vendor\\bin\\phpunit') {
                php vendor\\bin\\phpunit --log-junit build\\reports\\junit.xml
            } else {
                # fallback Laravel
                php artisan test --log-junit build\\reports\\junit.xml
            }
            """
        }
        }
    }

    post {
        always {
        // JUnit (si le fichier existe)
        script {
            if (fileExists('build/reports/junit.xml')) {
            junit testResults: 'build/reports/junit.xml', allowEmptyResults: true
            }
        }

        // Archiver ce qui existe (logs + rapports)
        archiveArtifacts artifacts: 'build/reports/**, storage/logs/*.log', allowEmptyArchive: true

        // Nettoyage demandé
        cleanWs()
        }
    }
    }
