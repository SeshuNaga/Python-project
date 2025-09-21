pipeline {
    agent any

    environment {
        DOCKERHUB_REPO = "seshubommineni/python-project"
        IMAGE_TAG      = "latest"
        K8S_DIR        = "k8s"
        FLUX_REPO      = "SeshuNaga/fluxrepo"   // Flux repo (GitHub)
    }

    stages {
        stage('Checkout App Code') {
            steps {
                git branch: 'main', url: 'https://github.com/SeshuNaga/Python-project.git'
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                script {
                    echo "🚀 Building Docker image..."
                    sh "docker build -t ${DOCKERHUB_REPO}:${IMAGE_TAG} ."

                    withCredentials([usernamePassword(credentialsId: 'docker-hub-creds',
                                                     passwordVariable: 'DOCKER_PASS',
                                                     usernameVariable: 'DOCKER_USER')]) {
                        echo "🔑 Logging in to Docker Hub..."
                        sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin"
                    }

                    echo "📦 Pushing image to Docker Hub..."
                    sh "docker push ${DOCKERHUB_REPO}:${IMAGE_TAG}"
                }
            }
        }

        stage('Update Flux Repo') {
            steps {
                script {
                    echo "📂 Cloning Flux repo..."
                    sh """
                        rm -rf fluxrepo
                        git clone https://github.com/${FLUX_REPO}.git
                        cd fluxrepo
                        git checkout -B release
                        
                        # Update FastAPI deployment manifest with new image
                        sed -i 's|image:.*|image: ${DOCKERHUB_REPO}:${IMAGE_TAG}|' ${K8S_DIR}/fastapi.yaml
                        
                        git config user.email "jenkins@pipeline.com"
                        git config user.name "Jenkins CI"
                        git add ${K8S_DIR}/fastapi.yaml
                        git commit -m "Update FastAPI image to ${IMAGE_TAG}" || echo "No changes to commit"
                        git push origin release --force
                    """
                }
            }
        }

        stage('Create/Update Pull Request') {
            steps {
                script {
                    echo "🔀 Creating or updating PR..."
                    sh """
                        gh pr create \
                          --repo ${FLUX_REPO} \
                          --head release \
                          --base main \
                          --title "Update FastAPI image to ${IMAGE_TAG}" \
                          --body "Automatic image update from Jenkins build"
                        || echo "PR already exists"
                    """
                }
            }
        }
    }

    post {
        success {
            echo "✅ Docker image built, pushed, and PR created in Flux repo!"
        }
        failure {
            echo "❌ Pipeline failed. Check logs."
        }
    }
}
