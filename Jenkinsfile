pipeline {
    agent any

    environment {
        DOCKERHUB_REPO = "seshubommineni/python-project"  // Your Docker Hub repo
        IMAGE_TAG      = "latest"                          // Tag for the image
        PATH           = "/usr/local/bin:/usr/bin:/bin"
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
                    def img = docker.build("${DOCKERHUB_REPO}:${IMAGE_TAG}")
                }
            }
        }

        stage('Login & Push to Docker Hub') {
            steps {
                script {
                    // Use Docker Hub credentials stored in Jenkins
                    docker.withRegistry('https://index.docker.io/v1/', 'docker-hub-creds') {
                        echo "📦 Pushing image to Docker Hub..."
                        def img = docker.build("${DOCKERHUB_REPO}:${IMAGE_TAG}")
                        img.push()
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ Docker image successfully pushed to Docker Hub!"
        }
        failure {
            echo "❌ Docker push failed. Check the logs."
        }
    }
}
