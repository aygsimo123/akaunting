    pipeline {
    agent any

    environment {
        APP_ENV = "testing"
    }

    stages {

        stage('Checkout') {
        steps {
            git url: 'https://github.com/aygsimo123/akaunting.git', branch: 'master'
        }
        }

        stage('Prepare Environment') {
        steps {
            sh '''
            php -v
            node -v
            composer --version
            npm -v
            '''
        }
        }

        stage('Install Backend Dependencies') {
        steps {
            sh '''
            composer install \
                --no-interaction \
                --prefer-dist \
                --optimize-autoloader
            '''
        }
        }

        stage('Install Frontend Dependencies') {
        steps {
            sh '''
            npm ci
            '''
        }
        }

        stage('Static Code Analysis') {
        steps {
            sh '''
            vendor/bin/pint --test || true
            vendor/bin/phpstan analyse --memory-limit=1G || true
            '''
        }
        }

        stage('Unit Tests') {
        steps {
            sh '''
            cp .env.example .env || true
            php artisan key:generate || true
            vendor/bin/phpunit
            '''
        }
        }

        stage('Security Scan (SCA)') {
        steps {
            sh '''
            composer audit || true
            '''
        }
        }

        stage('Build Frontend Assets') {
        steps {
            sh '''
            npm run build || npm run prod
            '''
        }
        }

        stage('Package Artifact') {
        steps {
            sh '''
            tar -czf akaunting-build.tar.gz .
            '''
        }
        }

        stage('Deploy (optional)') {
        when {
            branch 'main'
        }
        steps {
            echo "🚀 Déploiement manuel / SCP / rsync"
        }
        }
    }

    post {
        always {
        cleanWs()
        echo "✅ Pipeline terminée sans Docker"
        }

        failure {
        echo "❌ Erreur détectée – vérifier les logs Jenkins"
        }
    }
    }
