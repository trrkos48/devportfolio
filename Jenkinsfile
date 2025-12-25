pipeline {
    agent any

    environment {
        REGISTRY             = 'trrkos/devportfolio'
        REGISTRY_CREDENTIALS = 'docker-hub'
        IMAGE_TAG            = "${env.BUILD_NUMBER}"   // будет переопределён в Checkout
    }

    options {
        timestamps()
        disableConcurrentBuilds()
        timeout(time: 30, unit: 'MINUTES')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    def shortSha = env.GIT_COMMIT?.take(7)
                    if (!shortSha?.trim()) {
                        shortSha = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    }
                    // Требование: если sha пустой — использовать только BUILD_NUMBER
                    env.IMAGE_TAG = env.BUILD_NUMBER
                    if (shortSha?.trim()) {
                        env.IMAGE_TAG = "${env.BUILD_NUMBER}-${shortSha}"
                    }
                }
            }
        }

        stage('Build') {
            steps {
                sh "docker build -t ${env.REGISTRY}:${env.IMAGE_TAG} -t ${env.REGISTRY}:latest ."
            }
        }

        stage('Scan') {
            steps {
                sh """
                    set -e
                    docker run --rm \
                      -v /var/run/docker.sock:/var/run/docker.sock \
                      -v "${WORKSPACE}:/work" -w /work \
                      aquasec/trivy:0.51.4 image \
                        --exit-code 1 \
                        --severity HIGH,CRITICAL \
                        --ignore-unfixed \
                        --no-progress \
                        --format table \
                        --output trivy-report.txt \
                        ${env.REGISTRY}:${env.IMAGE_TAG}

                    cat trivy-report.txt
                """
                archiveArtifacts artifacts: 'trivy-report.txt', fingerprint: true
            }
        }

        stage('Publish') {
            steps {
                script {
                    docker.withRegistry('', env.REGISTRY_CREDENTIALS) {
                        sh "docker push ${env.REGISTRY}:${env.IMAGE_TAG}"
                        sh "docker push ${env.REGISTRY}:latest"
                    }
                }
            }
        }

        stage('Verify') {
            steps {
                sh """
                    set -e
                    docker pull ${env.REGISTRY}:latest
                    docker pull ${env.REGISTRY}:${env.IMAGE_TAG}

                    docker rm -f devportfolio_test >/dev/null 2>&1 || true
                    docker run -d -p 18080:80 --name devportfolio_test ${env.REGISTRY}:${env.IMAGE_TAG}

                    sleep 5
                    curl -fsS http://localhost:18080 | head -n 5

                    docker rm -f devportfolio_test >/dev/null 2>&1
                """
            }
        }
    }

    post {
        always {
            sh 'docker rm -f devportfolio_test >/dev/null 2>&1 || true'
            sh 'docker image prune -f'
            cleanWs()
        }
    }
}
