pipeline {
  agent any

    options {
    timestamps()
    disableConcurrentBuilds()
    }

    environment {
    // CI mode
    APP_ENV = "testing"
    APP_DEBUG = "false"
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Check tools') {
      steps {
        bat 'php -v'
        bat 'composer -V'
        bat 'node -v'
        bat 'npm -v'
      }
    }

    stage('Install dependencies') {
      steps {
        // PHP deps
        bat 'composer install --no-interaction --prefer-dist --no-progress ^ --ignore-platform-req=ext-intl --ignore-platform-req=ext-gd --ignore-platform-req=ext-zip'
        // JS deps
        bat 'npm ci'
      }
    }

    stage('SCA - Dependency audit') {
      steps {
        // Composer vuln scan (PHP)
        bat 'composer audit || exit /b 1'
        // npm vuln scan (JS)
        // --audit-level=high => fail si HIGH/CRITICAL
        bat 'npm audit --audit-level=high || exit /b 1'
      }
    }

    stage('Build frontend') {
      steps {
        // Selon le projet ça peut être dev/build, ici on fait build
        // Si ça échoue, remplace par: npm run dev
        bat 'npm run build'
      }
    }

    stage('Prepare Laravel env (SQLite)') {
      steps {
        // Crée .env si absent
        bat 'if not exist .env (copy .env.example .env)'

        // Génère APP_KEY
        bat 'php artisan key:generate --force'

        // Force SQLite pour CI
        bat 'if not exist database\\database.sqlite (type nul > database\\database.sqlite)'

        // Patch rapide du .env (Windows)
        bat '''
            powershell -NoProfile -Command ^
            "(Get-Content .env) ^
            -replace '^DB_CONNECTION=.*','DB_CONNECTION=sqlite' ^
            -replace '^DB_DATABASE=.*','DB_DATABASE=database\\\\database.sqlite' ^
            | Set-Content .env"
            '''
        // Migrate (si nécessaire pour tests)
        bat 'php artisan migrate --force'
    }
    }

    stage('Tests') {
        steps {
            // Akaunting a souvent phpunit.xml
            // Si vendor\\bin\\phpunit existe, lance les tests
            bat 'if exist vendor\\bin\\phpunit (vendor\\bin\\phpunit -c phpunit.xml) else (echo "phpunit not found" & exit /b 1)'
        }
        }

    stage('Secrets scan (Gitleaks)') {
        steps {
            // Tu dois installer gitleaks sur la machine Jenkins
            // https://github.com/gitleaks/gitleaks (binary)
            // On fait échouer si secret trouvé
            bat 'gitleaks version'
            bat 'gitleaks detect --source . --no-git --redact || exit /b 1'
        }
    }

    // OPTIONNEL: SonarQube (SAST)
    // Décommente si tu as SonarQube configuré dans Jenkins (Manage Jenkins > System)
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
      // Nettoyage simple
        echo "Pipeline terminé"
    }
    }
}
