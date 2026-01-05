    pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        ansiColor('xterm')
    }

    environment {
        // Laravel / CI
        APP_ENV   = "testing"
        APP_DEBUG = "false"

        // Aide node-gyp à détecter VS (si installé)
        GYP_MSVS_VERSION = "2022"

        // (optionnel) réduit certains soucis npm en CI
        NPM_CONFIG_FUND  = "false"
        NPM_CONFIG_AUDIT = "false"
    }

    stages {

        stage('Checkout') {
        steps {
            checkout scm
        }
        }

        stage('Check tools') {
        steps {
            bat '''
            echo === PHP ===
            php -v

            echo === Composer ===
            composer -V

            echo === Node ===
            node -v

            echo === NPM ===
            npm -v
            '''
        }
        }

        stage('Install dependencies') {
        steps {
            // PHP deps
            bat 'composer install --no-interaction --prefer-dist --no-progress'

            // JS deps
            // npm ci est recommandé en CI (lockfile)
            // Si ça échoue souvent à cause de node-gyp (node-sass, etc.), on affiche un message explicite
            script {
            def status = bat(script: 'npm ci', returnStatus: true)
            if (status != 0) {
                echo """
    ❌ npm ci a échoué.
    Cause probable sur Windows:
    - Visual Studio Build Tools (Desktop development with C++) non installé
    - ou package natif (ex: node-sass) incompatible avec Node actuel

    ✅ Fix recommandé:
    1) Installer "Visual Studio 2022 Build Tools" + workload "Desktop development with C++"
    2) Utiliser Node LTS (ex: 20) au lieu de Node 22
    3) Remplacer node-sass par sass (si possible)

    Je stoppe le pipeline.
    """
                error("npm ci failed (node-gyp / build tools)")
            }
            }
        }
        }

        stage('SCA - Dependency audit') {
        steps {
            // Composer vuln scan (PHP) => fail si vuln trouvée
            bat 'composer audit || exit /b 1'

            // npm audit => fail si HIGH/CRITICAL
            bat 'npm audit --audit-level=high || exit /b 1'
        }
        }

        stage('Build frontend') {
        steps {
            // Essaie build, sinon dev si ton projet n'a pas build
            script {
            def status = bat(script: 'npm run build', returnStatus: true)
            if (status != 0) {
                echo '⚠️ npm run build a échoué, tentative avec npm run dev...'
                bat 'npm run dev || exit /b 1'
            }
            }
        }
        }

        stage('Prepare Laravel env (SQLite)') {
        steps {
            bat '''
            if not exist .env (copy .env.example .env)

            php artisan key:generate --force

            if not exist database\\database.sqlite (type nul > database\\database.sqlite)

            powershell -NoProfile -Command ^
                "(Get-Content .env) ^
                -replace '^DB_CONNECTION=.*','DB_CONNECTION=sqlite' ^
                -replace '^DB_DATABASE=.*','DB_DATABASE=database\\\\database.sqlite' ^
                | Set-Content .env"

            php artisan config:clear
            php artisan cache:clear
            '''

            // migrate (si ton projet en a besoin)
            bat 'php artisan migrate --force || exit /b 1'
        }
        }

        stage('Tests') {
        steps {
            // PHPUnit
            bat '''
            if exist vendor\\bin\\phpunit (
                vendor\\bin\\phpunit -c phpunit.xml || exit /b 1
            ) else (
                echo "phpunit not found" & exit /b 1
            )
            '''
        }
        }

        stage('Secrets scan (Gitleaks)') {
        steps {
            // Si gitleaks n'est pas installé, on ne casse pas le pipeline (tu peux changer ça)
            script {
            def status = bat(script: 'gitleaks version', returnStatus: true)
            if (status != 0) {
                echo '⚠️ gitleaks n’est pas installé sur l’agent Jenkins. Stage ignoré.'
            } else {
                bat 'gitleaks detect --source . --no-git --redact || exit /b 1'
            }
            }
        }
        }

        // OPTIONNEL: SonarQube (SAST)
        /*
        stage('SAST - SonarQube') {
        steps {
            withSonarQubeEnv('SONAR') {
            bat 'sonar-scanner'
            }
        }
        }
        */
    }

    post {
        always {
        cleanWs()
        echo "Pipeline terminé"
        }
    }
    }
