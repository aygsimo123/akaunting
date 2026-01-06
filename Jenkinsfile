pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '30'))
    }

    stages {
        stage('Checkout') {
        steps { checkout scm }
        }

        stage('Fail-fast security checks') {
            parallel {
                stage('Secrets - Gitleaks') {
                steps {
                    sh '''
                    mkdir -p build/gitleaks
                    gitleaks detect --source . --report-format json --report-path build/gitleaks/report.json
                    '''
                }
                }

                stage('SAST - Semgrep') {
                steps {
                    sh '''
                    mkdir -p build/semgrep
                    semgrep scan --config p/default --json -o build/semgrep/semgrep.json .
                    '''
                }
                }

                stage('SCA - Composer audit') {
                steps {
                    sh '''
                    mkdir -p build/sca
                    composer install --no-interaction --prefer-dist
                    composer audit --format=json > build/sca/composer-audit.json
                    '''
                }
                }

                stage('SCA - npm audit') {
                steps {
                    sh '''
                    mkdir -p build/sca
                    npm ci || npm install
                    npm audit --json > build/sca/npm-audit.json || true
                    '''
                }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'build/**', allowEmptyArchive: true
                    cleanWs()
                }
            }
            }
        }
        }

