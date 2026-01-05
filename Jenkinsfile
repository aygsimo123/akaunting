pipeline {
  agent any

  environment {
    PHP_IMAGE = 'php:8.2-cli'
    NODE_IMAGE = 'node:18'
    COMPOSER_CACHE = "${WORKSPACE}/.composer/cache"
  }

  tools {
    nodejs 'NodeJS'
  }

  stages {

    stage('Checkout') {
      steps {
        git url: 'https://github.com/akaunting/akaunting.git', branch: 'master'
      }
    }

    stage('Prepare') {
      parallel {
        stage('Composer Install') {
          agent { docker { image "${PHP_IMAGE}" } }
          steps {
            sh 'composer install --no-interaction --prefer-dist --optimize-autoloader'
          }
        }
        stage('Node Modules') {
          agent { docker { image "${NODE_IMAGE}" } }
          steps {
            sh 'npm install'
          }
        }
      }
    }

    stage('Static Analysis & Lint') {
      agent { docker { image "${PHP_IMAGE}" } }
      steps {
        sh 'vendor/bin/pint --test || true'
        sh 'vendor/bin/phpstan analyse --memory-limit=1G || true'
      }
    }

    stage('Unit Tests') {
      agent { docker { image "${PHP_IMAGE}" } }
      steps {
        sh 'vendor/bin/phpunit --fail-on-risky --fail-on-warning'
      }
    }

    stage('Dependency Security Scan') {
      agent { docker { image "${PHP_IMAGE}" } }
      steps {
        sh 'composer audit --ignore-severity=medium'
      }
    }

    stage('Build Assets') {
      agent { docker { image "${NODE_IMAGE}" } }
      steps {
        sh 'npm run prod'
      }
    }

    stage('Docker Build (optional)') {
      steps {
        script {
          docker.build("akaunting/app:${env.BUILD_NUMBER}")
        }
      }
    }

    stage('Deploy/Push') {
      when { expression { env.BRANCH_NAME == 'main' } }
      steps {
        echo "⚡ Deploy task here (SSH / Registry / Kubernetes)"
      }
    }
  }

  post {
    always {
      cleanWs()
      echo "✅ Pipeline terminée"
    }

    failure {
      mail to: 'dev-team@example.com',
           subject: "❌ Build Failed: ${env.JOB_NAME}",
           body: 'Veuillez vérifier les logs Jenkins.'
    }
  }
}
