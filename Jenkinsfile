pipeline {
    agent any

    environment {
        DOCKERHUB_REPO = "seshubommineni/python-project"
        IMAGE_TAG      = "latest"
        PATH           = "/usr/local/bin:/usr/bin:/bin"   // Ensure Jenkins can find Docker
        FLUX_REPO      = "https://github.com/SeshuNaga/fluxrepo.git"
        FLUX_BRANCH    = "main"
        K8S_DIR        = "manifests"
    }

    stages {
        stage('Checkout Python Code') {
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
                                                     usernameVariable: 'DOCKER_USER', 
                                                     passwordVariable: 'DOCKER_PASS')]) {
                        sh "docker login -u $DOCKER_USER -p $DOCKER_PASS"
                    }

                    echo "📦 Pushing image to Docker Hub..."
                    sh "docker push ${DOCKERHUB_REPO}:${IMAGE_TAG}"
                }
            }
        }

        stage('Update Flux Repo') {
            steps {
                script {
                    echo "📥 Cloning Flux repo..."
                    sh "rm -rf fluxrepo"
                    sh "git clone ${FLUX_REPO} fluxrepo"
                    sh "cd fluxrepo && git checkout ${FLUX_BRANCH}"

                    echo "📝 Updating FastAPI image in manifests..."
                    sh """
                        cd fluxrepo
                        sed -i.bak 's|image:.*|image: ${DOCKERHUB_REPO}:${IMAGE_TAG}|g' ${K8S_DIR}/fastapi.yaml
                        rm -f ${K8S_DIR}/fastapi.yaml.bak
                        git add ${K8S_DIR}/fastapi.yaml
                        git commit -m 'Update FastAPI image to ${IMAGE_TAG}' || echo 'No changes to commit'
                        git push origin ${FLUX_BRANCH} --force
                    """
                }
            }
        }
    }

    post {
        success {
            echo "✅ Docker image pushed and Flux repo updated on main branch!"
        }
        failure {
            echo "❌ Pipeline failed. Check logs."
        }
    }
}
