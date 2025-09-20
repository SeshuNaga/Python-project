pipeline {
    agent any

    environment {
        DOCKERHUB_REPO = "seshubommineni/python-project"  // Your Docker Hub repo
        IMAGE_TAG      = "latest"                          // Tag for the image
        PATH           = "/usr/local/bin:/usr/bin:/bin"    // Ensure Jenkins can find Docker
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
                    // Build the image locally
                    def img = sh(script: "docker build -t ${DOCKERHUB_REPO}:${IMAGE_TAG} .", returnStdout: true)
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
                        sh "docker login -u $DOCKER_USER -p $DOCKER_PASS"
                    }

                    echo "📦 Pushing image to Docker Hub..."
                    sh "docker push ${DOCKERHUB_REPO}:${IMAGE_TAG}"
                }
            }
        }
    }

    post {
        success {
            echo "✅ Docker image successfully pushed to Docker Hub!"
        }
        failure {
            echo "❌ Docker push failed. Check logs."
        }
    }
}
