pipeline {
    agent any

    environment {
        REGISTRY = "localhost:5001"   // Use localhost for k3d local registry
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
                docker build -t $REGISTRY/$IMAGE_NAME:$IMAGE_TAG .
                """
            }
        }

        stage('Push to Local Registry') {
            steps {
                sh """
                docker push $REGISTRY/$IMAGE_NAME:$IMAGE_TAG
                """
            }
        }

        stage('Deploy to K3D') {
            steps {
                sh """
                kubectl apply -f $K8S_DIR/postgres.yaml
                kubectl apply -f $K8S_DIR/fastapi.yaml
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
