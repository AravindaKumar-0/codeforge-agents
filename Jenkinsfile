
pipeline {
    agent any

    options {
        timestamps()
    }

    environment {
        IMAGE_NAME = 'imaravinda/codeforge-agents'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.IMAGE_TAG = sh(
                        script: 'git rev-parse --short=12 HEAD',
                        returnStdout: true
                    ).trim()
                }
                echo "Image tag: ${IMAGE_TAG}"
            }
        }

        stage('Inspect Project') {
            steps {
                sh '''
                    test -f Dockerfile
                    test -f requirements.txt
                    echo "Project files verified."
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build \
                      -t ${IMAGE_NAME}:${IMAGE_TAG} \
                      .
                '''
            }
        }
    }

    post {
        success {
            echo 'Docker image built successfully.'
        }
        failure {
            echo 'Pipeline failed. Check Console Output.'
        }
    }
}
