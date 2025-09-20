pipeline {
    agent any

    environment {
        DOCKERHUB_REPO = "seshubommineni/python-project"
        IMAGE_TAG      = "latest"
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
                    // Build once and store in 'img'
                    img = docker.build("${DOCKERHUB_REPO}:${IMAGE_TAG}")
                }
            }
        }

        stage('Login & Push to Docker Hub') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'docker-hub-creds') {
                        echo "📦 Pushing image to Docker Hub..."
                        img.push()   // Reuse the image object built in previous stage
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
