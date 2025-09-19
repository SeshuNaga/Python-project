pipeline {
    agent any

    environment {
        REGISTRY = "myregistry.localhost:5000"
        IMAGE_NAME = "fastapi-psql-service"
        IMAGE_TAG = "latest"
        K8S_DIR = "k8s"

        PATH = "/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin"
        DOCKER_BUILDKIT = "0"
        DOCKER_CONFIG = "/tmp/docker-config"
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/SeshuNaga/Python-project.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                echo "🚀 Building Docker image..."
                docker build -t $REGISTRY/$IMAGE_NAME:$IMAGE_TAG .
                """
            }
        }

        stage('Push to Local Registry') {
            steps {
                sh """
                echo "📦 Pushing image to k3d registry..."
                docker push $REGISTRY/$IMAGE_NAME:$IMAGE_TAG
                """
            }
        }

        stage('Update K8s Manifests') {
            steps {
                sh """
                echo "🔧 Updating Deployment image reference..."
                sed -i.bak "s|image:.*|image: $REGISTRY/$IMAGE_NAME:$IMAGE_TAG|" $K8S_DIR/fastapi.yaml
                """
            }
        }

        stage('Deploy to K3D') {
            steps {
                sh """
                echo "🚀 Deploying to k3d cluster..."
                kubectl apply -f $K8S_DIR/postgres.yaml
                kubectl apply -f $K8S_DIR/fastapi.yaml
                kubectl rollout status deployment fastapi
                """
            }
        }
    }

    post {
        success {
            echo "✅ Deployment successful!"
        }
        failure {
            echo "❌ Deployment failed. Check logs."
        }
    }
}
