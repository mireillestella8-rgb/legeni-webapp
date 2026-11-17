// Legeni WebApp - pipeline CI/CD (Lots 4 et 5)
pipeline {
    agent any

    parameters {
        booleanParam(name: 'RUN_TRIVY', defaultValue: true, description: 'Scanner l image avec Trivy')
    }

    options {
        timestamps()
        timeout(time: 45, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '15'))
        disableConcurrentBuilds()
    }

    environment {
        APP_NAME = 'legeni-webapp'
    }

    stages {
        stage('Preparation') {
            steps {
                script {
                    env.APP_VERSION = readFile('VERSION').trim()
                    env.IMAGE_TAG = "${env.APP_VERSION}-${env.BUILD_NUMBER}"
                }
                echo "Version ${env.APP_VERSION} - image ${env.APP_NAME}:${env.IMAGE_TAG} - branche ${env.BRANCH_NAME}"
            }
        }

        stage('Controles qualite') {
            parallel {
                stage('Tests, flake8, couverture') {
                    steps {
                        sh '''
                            rm -rf reports && mkdir -p reports
                            docker build --target test -t ${APP_NAME}:test-${BUILD_NUMBER} .
                            set +e
                            docker run --name cov-${BUILD_NUMBER} ${APP_NAME}:test-${BUILD_NUMBER} \
                                pytest --cov=app --cov-report=term --cov-report=xml:/app/coverage.xml \
                                --cov-fail-under=80 --junitxml=/app/junit.xml -q
                            rc=$?
                            docker cp cov-${BUILD_NUMBER}:/app/junit.xml reports/junit.xml
                            docker cp cov-${BUILD_NUMBER}:/app/coverage.xml reports/coverage.xml
                            docker rm -f cov-${BUILD_NUMBER}
                            exit $rc
                        '''
                    }
                }
                stage('hadolint') {
                    steps {
                        sh 'docker run --rm -i hadolint/hadolint:v2.12.0-alpine hadolint --failure-threshold error - < Dockerfile'
                    }
                }
                stage('Syntaxe Ansible') {
                    steps {
                        withCredentials([file(credentialsId: 'ansible-vault-pass', variable: 'VAULT_FILE')]) {
                            dir('ansible') {
                                sh '''
                                    ANSIBLE_VAULT_PASSWORD_FILE="$VAULT_FILE" ansible-playbook provision.yml --syntax-check
                                    ANSIBLE_VAULT_PASSWORD_FILE="$VAULT_FILE" ansible-playbook deploy.yml --syntax-check -e target_env=staging
                                '''
                            }
                        }
                    }
                }
            }
        }

        stage('Build image') {
            steps {
                sh 'docker build --build-arg APP_VERSION=$APP_VERSION -t ${APP_NAME}:${IMAGE_TAG} .'
            }
        }

        stage('Scan Trivy') {
            when { expression { params.RUN_TRIVY } }
            steps {
                sh '''
                    docker run --rm -v trivy-cache:/root/.cache/ \
                        -v /var/run/docker.sock:/var/run/docker.sock \
                        aquasec/trivy:0.58.1 image --exit-code 1 --severity CRITICAL \
                        --ignore-unfixed --skip-db-update --no-progress ${APP_NAME}:${IMAGE_TAG}
                '''
            }
        }

        stage('Push registre') {
            when { branch 'main' }
            steps {
                withCredentials([usernamePassword(credentialsId: 'registry-credentials',
                                                  usernameVariable: 'REG_USER',
                                                  passwordVariable: 'REG_PASS')]) {
                    sh '''
                        echo "$REG_PASS" | docker login -u "$REG_USER" --password-stdin
                        docker tag ${APP_NAME}:${IMAGE_TAG} "$REG_USER/${APP_NAME}:${IMAGE_TAG}"
                        docker push "$REG_USER/${APP_NAME}:${IMAGE_TAG}"
                        docker logout
                    '''
                    script { env.IMAGE_REF = "${REG_USER}/${env.APP_NAME}:${env.IMAGE_TAG}" }
                }
            }
        }

        stage('Deploiement staging') {
            when { branch 'main' }
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG'),
                                 file(credentialsId: 'ansible-vault-pass', variable: 'VAULT_FILE')]) {
                    dir('ansible') {
                        sh 'ANSIBLE_VAULT_PASSWORD_FILE="$VAULT_FILE" ansible-playbook deploy.yml -e target_env=staging -e app_image="$IMAGE_REF" -e app_version="$APP_VERSION"'
                    }
                }
            }
        }

        stage('Validation production') {
            when { branch 'main' }
            steps {
                timeout(time: 30, unit: 'MINUTES') {
                    input message: "Staging valide (version ${env.APP_VERSION}). Deployer en production ?", ok: 'Deployer en production'
                }
            }
        }

        stage('Deploiement production') {
            when { branch 'main' }
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG'),
                                 file(credentialsId: 'ansible-vault-pass', variable: 'VAULT_FILE')]) {
                    dir('ansible') {
                        sh 'ANSIBLE_VAULT_PASSWORD_FILE="$VAULT_FILE" ansible-playbook deploy.yml -e target_env=prod -e app_image="$IMAGE_REF" -e app_version="$APP_VERSION"'
                    }
                }
            }
        }
    }

    post {
        always {
            junit allowEmptyResults: true, testResults: 'reports/junit.xml'
            archiveArtifacts artifacts: 'reports/*', allowEmptyArchive: true
            script {
                try {
                    recordCoverage(tools: [[parser: 'COBERTURA', pattern: 'reports/coverage.xml']])
                } catch (err) {
                    echo "Rapport de couverture non publie : ${err.message}"
                }
            }
            sh 'docker rm -f cov-${BUILD_NUMBER} || true; docker rmi -f ${APP_NAME}:test-${BUILD_NUMBER} || true'
        }
        success {
            echo "Pipeline termine : ${env.APP_NAME}:${env.IMAGE_TAG}"
        }
        failure {
            script {
                def msg = "Echec ${env.JOB_NAME} #${env.BUILD_NUMBER} - ${env.BUILD_URL}"
                try {
                    withCredentials([string(credentialsId: 'notify-webhook', variable: 'HOOK')]) {
                        sh "curl -s -m 20 -X POST -H 'Content-Type: application/json' -d '{\"content\": \"${msg}\", \"text\": \"${msg}\"}' \"\$HOOK\""
                    }
                } catch (err) {
                    echo "Notification non envoyee (credential notify-webhook absent ?) : ${msg}"
                }
            }
        }
    }
}
