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
                    // fail the build on high/critical vulnerabilities
                    sh """
                        trivy image \
                          --exit-code 1 \
                          --severity HIGH,CRITICAL \
                          --ignore-unfixed \
                          ${REGISTRY}:${IMAGE_TAG}
                    """
                }
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
    }

    post {
        always {
            sh "docker image prune -f"
        }
    }
}

