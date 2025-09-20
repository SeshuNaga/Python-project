pipeline {
    agent any

    environment {
        DOCKERHUB_REPO = "dockerhub-username/fastapi-psql-service"
        IMAGE_TAG      = "latest"
        K8S_DIR        = "k8s"
        PATH           = "/usr/local/bin:/usr/bin:/bin"
        DOCKER_BUILDKIT = "0"
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/SeshuNaga/Python-project.git'
            }
        }

        stage('Build and Push Docker Image') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'docker-hub-creds') {
                        def img = docker.build("${DOCKERHUB_REPO}:${IMAGE_TAG}")
                        img.push()   
                    }
                }
            }
        }

        stage('Update K8s Manifests') {
            steps {
                sh """
                echo "🔧 Updating Deployment image reference..."
                sed -i.bak "s|image:.*|image: $DOCKERHUB_REPO:$IMAGE_TAG|" $K8S_DIR/fastapi.yaml
                """
            }
        }

        stage('Deploy to K3D') {
            steps {
                sh """
                echo "🚀 Deploying to k3d cluster..."
                kubectl apply -f $K8S_DIR/postgres.yaml
                kubectl apply -f $K8S_DIR/fastapi.yaml
                kubectl rollout status deployment/fastapi
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
