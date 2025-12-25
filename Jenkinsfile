pipeline {
    agent any

    environment {
        REGISTRY             = 'trrkos/devportfolio'
        REGISTRY_CREDENTIALS = 'docker-hub'
        IMAGE_TAG            = "${env.BUILD_NUMBER}"
    }

    options {
        timestamps()
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
                    env.IMAGE_TAG = env.BUILD_NUMBER
                    if (shortSha) {
                        env.IMAGE_TAG = "${env.BUILD_NUMBER}-${shortSha}"
                    }
                }
            }
        }

        stage('Build') {
            steps {
                script {
                    sh "docker build -t ${REGISTRY}:${IMAGE_TAG} -t ${REGISTRY}:latest ."
                }
            }
        }

        stage('Scan') {
            steps {
                script {
                    sh """
                        trivy image \
                          --exit-code 1 \
                          --severity HIGH,CRITICAL \
                          --ignore-unfixed \
                          --no-progress \
                          --format table \
                          --output trivy-report.txt \
                          ${REGISTRY}:${IMAGE_TAG}
                        cat trivy-report.txt
                    """
                }
                archiveArtifacts artifacts: 'trivy-report.txt', fingerprint: true
            }
        }

        stage('Publish') {
            steps {
                script {
                    docker.withRegistry('', REGISTRY_CREDENTIALS) {
                        sh "docker push ${REGISTRY}:${IMAGE_TAG}"
                        sh "docker push ${REGISTRY}:latest"
                    }
                }
            }
        }

        stage('Verify') {
            steps {
                script {
                    sh """
                        set -e
                        docker pull ${REGISTRY}:latest
                        docker pull ${REGISTRY}:${IMAGE_TAG}
                        docker rm -f devportfolio_test >/dev/null 2>&1 || true
                        docker run -d -p 18080:80 --name devportfolio_test ${REGISTRY}:${IMAGE_TAG}
                        sleep 5
                        curl -fsS http://localhost:18080 | head -n 5
                        docker rm -f devportfolio_test >/dev/null 2>&1
                    """
                }
            }
        }
    }

    post {
        always {
            sh 'docker rm -f devportfolio_test >/dev/null 2>&1 || true'
            sh "docker image prune -f"
        }
    }
}

