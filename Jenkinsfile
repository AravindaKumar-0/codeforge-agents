
pipeline {
    agent any

    options {
        timestamps()
    }

    environment {
        IMAGE_NAME = 'imaravinda/codeforge-agents'
        GITOPS_REPO = 'https://github.com/AravindaKumar-0/codeforge-agents-gitops.git'
        GITOPS_BRANCH = 'main'
        GITOPS_CREDENTIALS = 'github-gitops-token'
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

        stage('Validate Project') {
            steps {
                sh '''
                    test -f Dockerfile
                    test -f requirements.txt
                '''
            }
        }

        stage('Build Image') {
            steps {
                sh '''
                    docker build \
                      -t "${IMAGE_NAME}:${IMAGE_TAG}" \
                      .
                '''
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKERHUB_USER',
                        passwordVariable: 'DOCKERHUB_TOKEN'
                    )
                ]) {
                    sh '''
                        set +x
                        echo "$DOCKERHUB_TOKEN" | docker login \
                          --username "$DOCKERHUB_USER" \
                          --password-stdin
                        docker push "${IMAGE_NAME}:${IMAGE_TAG}"
                        docker logout
                    '''
                }
            }
        }

        stage('Update GitOps Repository') {
            steps {
                dir('gitops') {
                    deleteDir()

                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: "*/${GITOPS_BRANCH}"]],
                        userRemoteConfigs: [[
                            url: GITOPS_REPO,
                            credentialsId: GITOPS_CREDENTIALS
                        ]]
                    ])

                    sh '''
                        set -eu

                        git config user.name "Jenkins CI"
                        git config user.email "jenkins@localhost"

                        # Update the Kustomize image tag.
                        python3 - <<'PY'
from pathlib import Path
import os
import re

path = Path("apps/codeforge-agents/kustomization.yaml")
text = path.read_text()
image = os.environ["IMAGE_NAME"]
tag = os.environ["IMAGE_TAG"]

pattern = (
    r'(?m)^([ \t]*-[ \t]*name:[ \t]*)'
    + re.escape(image)
    + r'[ \t]*\\n([ \t]*newTag:[ \t]*).*$'
)

updated, count = re.subn(
    pattern,
    lambda m: m.group(1) + image + "\\n" + m.group(2) + '"' + tag + '"',
    text,
    count=1
)

if count != 1:
    raise SystemExit(
        "Could not find the expected image entry in kustomization.yaml"
    )

path.write_text(updated)
PY

                        git add apps/codeforge-agents/kustomization.yaml

                        if git diff --cached --quiet; then
                            echo "No GitOps change to commit."
                        else
                            git commit -m "Update CodeForge Agents image to ${IMAGE_TAG}"
                            git push origin "HEAD:${GITOPS_BRANCH}"
                        fi
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Image published and GitOps repository updated.'
        }
        failure {
            echo 'Pipeline failed. Check Console Output.'
        }
    }
}
