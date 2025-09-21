pipeline {
    agent any

    environment {
        DOCKERHUB_REPO = "seshubommineni/python-project"
        IMAGE_TAG      = "latest"
        FLUX_REPO      = "https://github.com/SeshuNaga/fluxrepo.git"
        FLUX_DIR       = "fluxrepo"
        GITHUB_TOKEN   = credentials('github-token')  // GitHub token stored in Jenkins
    }

    stages {
        stage('Checkout Source Code') {
            steps {
                git branch: 'main', url: 'https://github.com/SeshuNaga/Python-project.git'
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                script {
                    echo "🚀 Building Docker image..."
                    sh "docker build -t ${DOCKERHUB_REPO}:${IMAGE_TAG} ."

                    echo "🔑 Logging into Docker Hub..."
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-creds', 
                                                     passwordVariable: 'DOCKER_PASS', 
                                                     usernameVariable: 'DOCKER_USER')]) {
                        sh "echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin"
                    }

                    echo "📦 Pushing image to Docker Hub..."
                    sh "docker push ${DOCKERHUB_REPO}:${IMAGE_TAG}"
                }
            }
        }

        stage('Update Flux Repo & Create PR') {
            steps {
                script {
                    // Clone or fetch flux repo
                    sh """
                    if [ ! -d ${FLUX_DIR} ]; then
                        git clone ${FLUX_REPO} ${FLUX_DIR}
                    fi
                    cd ${FLUX_DIR}
                    git fetch origin
                    git checkout main
                    git pull origin main
                    git branch -D release || true
                    git checkout -b release
                    """

                    // Update FastAPI image in YAML
                    sh """
                    sed -i.bak 's|image: \".*\"|image: \"${DOCKERHUB_REPO}:${IMAGE_TAG}\"|' ${FLUX_DIR}/manifests/fastapi.yaml
                    rm -f ${FLUX_DIR}/manifests/fastapi.yaml.bak
                    """

                    // Commit & push changes
                    sh """
                    cd ${FLUX_DIR}
                    git add manifests/fastapi.yaml
                    git commit -m "Update FastAPI image to ${IMAGE_TAG}" || echo "No changes to commit"
                    git push origin release --force
                    """

                    // Create PR using gh CLI
                    sh """
                    cd ${FLUX_DIR}
                    EXISTING_PR=\$(gh pr list --head release --base main --json url -q '.[0].url')
                    if [ -z "\$EXISTING_PR" ]; then
                        PR_URL=\$(gh pr create --title "Update FastAPI image to ${IMAGE_TAG}" \
                                               --body "Automatic image update from Jenkins build ${IMAGE_TAG}" \
                                               --head release --base main \
                                               --json url -q '.url')
                    else
                        PR_URL=\$EXISTING_PR
                    fi
                    echo "✅ Pull Request URL: \$PR_URL"
                    """
                }
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline completed successfully!"
        }
        failure {
            echo "❌ Pipeline failed. Check logs."
        }
    }
}
