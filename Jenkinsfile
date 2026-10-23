
pipeline {
    agent any

    options {
        timestamps()
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'git rev-parse --short HEAD'
            }
        }

        stage('Inspect Project') {
            steps {
                sh '''
                    echo "Checking project files..."
                    ls -la
                    test -f Dockerfile
                    test -f requirements.txt
                '''
            }
        }

        stage('CI/CD Setup Check') {
            steps {
                echo 'Checkout and project structure checks passed.'
                echo 'Docker build, image push, and GitOps update will be added next.'
            }
        }
    }
}
