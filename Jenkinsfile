pipeline {
    agent any

    environment {
        DOCKERHUB_REPO = "seshubommineni/python-project"
        IMAGE_TAG      = "latest"
        PATH           = "/usr/local/bin:/usr/bin:/bin"   // Ensure Jenkins can find Docker and gh
        FLUX_DIR       = "fluxrepo"                       // Flux repo folder
    }

    stages {
        stage('Checkout Code') {
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
                    echo "🔑 Logging in to Docker Hub..."
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-creds', 
                                                     passwordVariable: 'DOCKER_PASS', 
                                                     usernameVariable: 'DOCKER_USER')]) {
                        sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin"
                    }

                    echo "📦 Pushing image to Docker Hub..."
                    sh "docker push ${DOCKERHUB_REPO}:${IMAGE_TAG}"
                }
            }
        }

        stage('Clone Flux Repo') {
            steps {
                script {
                    echo "📥 Cloning Flux repo..."
                    sh "rm -rf ${FLUX_DIR}"  // Remove existing folder if present
                    sh "git clone https://github.com/SeshuNaga/fluxrepo.git ${FLUX_DIR}"
                }
            }
        }

        stage('Update FastAPI Image in Flux') {
            steps {
                script {
                    echo "📝 Updating FastAPI image in Flux manifests..."
                    sh """
                        cd ${FLUX_DIR}
                        git checkout -B release || git checkout -b release
                        sed -i.bak 's|image: \\'${DOCKERHUB_REPO}:.*\\'|image: \\'${DOCKERHUB_REPO}:${IMAGE_TAG}\\'|g' manifests/fastapi.yaml
                        rm -f manifests/fastapi.yaml.bak
                        git add manifests/fastapi.yaml
                        git commit -m 'Update FastAPI image to ${IMAGE_TAG}' || echo 'No changes to commit'
                        git push origin release --force
                    """
                }
            }
        }

        stage('Create Pull Request') {
            steps {
                script {
                    echo "📬 Creating Pull Request..."
                    sh """
                        cd ${FLUX_DIR}
                        EXISTING_PR=\$(gh pr list --head release --base main --json url -q '.[0].url')
                        if [ -z "\$EXISTING_PR" ]; then
                            PR_URL=\$(gh pr create --title "Update FastAPI image to ${IMAGE_TAG}" --body "Automatic image update from Jenkins build ${IMAGE_TAG}" --head release --base main --json url -q '.url')
                        else
                            PR_URL=\$EXISTING_PR
                        fi
                        echo "✅ PR URL: \$PR_URL"
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
            echo "❌ Pipeline failed. Check logs."
        }
    }
}
