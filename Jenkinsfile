
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
                    set -eu
                    test -f Dockerfile
                    test -f requirements.txt
                    echo "Project files verified."
                '''
            }
        }

        stage('Build Image') {
            steps {
                sh '''
                    set -eu
                    docker build \
                      -t "${IMAGE_NAME}:${IMAGE_TAG}" \
                      .
                '''
            }
        }

        stage('Push Image to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKERHUB_USER',
                        passwordVariable: 'DOCKERHUB_TOKEN'
                    )
                ]) {
                    sh '''
                        set -eu
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

                    withCredentials([
                        usernamePassword(
                            credentialsId: 'github-gitops-token',
                            usernameVariable: 'GIT_USERNAME',
                            passwordVariable: 'GIT_PASSWORD'
                        )
                    ]) {
                        sh '''
                            set -eu
                            set +x

                            git config user.name "Jenkins CI"
                            git config user.email "jenkins@localhost"

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

def replace_image(match):
    return (
        match.group(1)
        + image
        + "\\n"
        + match.group(2)
        + '"' + tag + '"'
    )

updated, count = re.subn(
    pattern,
    replace_image,
    text,
    count=1
)

if count != 1:
    raise SystemExit(
        "Expected image entry not found in kustomization.yaml"
    )

path.write_text(updated)
PY

                            git add apps/codeforge-agents/kustomization.yaml

                            if git diff --cached --quiet; then
                                echo "No GitOps changes to commit."
                                exit 0
                            fi

                            git commit -m \
                              "Update CodeForge Agents image to ${IMAGE_TAG}"

                            askpass="$(mktemp)"
                            trap 'rm -f "$askpass"' EXIT

                            cat > "$askpass" <<'ASKPASS'
#!/bin/sh
case "$1" in
    *Username*) printf '%s\\n' "$GIT_USERNAME" ;;
    *Password*) printf '%s\\n' "$GIT_PASSWORD" ;;
    *) exit 1 ;;
esac
ASKPASS

                            chmod 700 "$askpass"

                            GIT_ASKPASS="$askpass" \
                            GIT_TERMINAL_PROMPT=0 \
                                git push origin "HEAD:${GITOPS_BRANCH}"
                        '''
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: Image published and GitOps update completed.'
        }

        failure {
            echo 'FAILED: Check Console Output for the failing stage.'
        }

        always {
            echo 'CodeForge Agents pipeline finished.'
        }
    }
}
