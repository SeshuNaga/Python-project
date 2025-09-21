pipeline {
    agent any

    environment {
        DOCKERHUB_REPO = "seshubommineni/python-project"
        IMAGE_TAG      = "latest"
        PATH           = "/usr/local/bin:/usr/bin:/bin:/opt/homebrew/bin"   // Ensure Docker, git, gh are in PATH
        FLUX_DIR       = "fluxrepo"                                         // Local clone of Flux repo
        GITHUB_REPO    = "SeshuNaga/fluxrepo"
    }

    stages {
        stage('Checkout Source Code') {
            steps {
                git branch: 'main', url: 'https://github.com/SeshuNaga/Python-project.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    echo "🚀 Building Docker image..."
                    sh "docker build -t ${DOCKERHUB_REPO}:${IMAGE_TAG} ."
                }
            }
        }

        stage('Login & Push to Docker Hub') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-creds', 
                                                     passwordVariable: 'DOCKER_PASS', 
                                                     usernameVariable: 'DOCKER_USER')]) {
                        echo "🔑 Logging in to Docker Hub..."
                        sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin"
                    }

                    echo "📦 Pushing Docker image..."
                    sh "docker push ${DOCKERHUB_REPO}:${IMAGE_TAG}"
                }
            }
        }

        stage('Update FastAPI Image in Flux') {
            steps {
                script {
                    echo "📝 Updating FastAPI image in Flux manifests..."

                    // Clone or pull Flux repo
                    sh """
                        if [ -d "${FLUX_DIR}" ]; then
                            cd ${FLUX_DIR} && git checkout main && git pull origin main
                        else
                            git clone https://github.com/${GITHUB_REPO}.git ${FLUX_DIR}
                            cd ${FLUX_DIR}
                        fi
                    """

                    // Update image in YAML
                    sh """
                        cd ${FLUX_DIR}
                        git checkout -B release || git checkout -b release
                        sed -i.bak "s|image: '${DOCKERHUB_REPO}:.*'|image: '${DOCKERHUB_REPO}:${IMAGE_TAG}'|g" manifests/fastapi.yaml
                        rm -f manifests/fastapi.yaml.bak
                        git add manifests/fastapi.yaml
                        git commit -m "Update FastAPI image to ${IMAGE_TAG}" || echo 'No changes to commit'
                        git push origin release --force
                    """

                    // Create or get existing PR
                    sh """
                        cd ${FLUX_DIR}
                        EXISTING_PR=\$(gh pr list --head release --base main --json url -q '.[0].url')
                        if [ -z "\$EXISTING_PR" ]; then
                            PR_URL=\$(gh pr create -t "Update FastAPI image to ${IMAGE_TAG}" -b "Automatic image update from Jenkins build ${IMAGE_TAG}" -B main -H release --json url -q '.url')
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
            echo "✅ Docker image pushed and Flux PR created successfully!"
        }
        failure {
            echo "❌ Pipeline failed. Check logs for details."
        }
    }
}
