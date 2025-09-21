pipeline {
    agent any

    environment {
        DOCKERHUB_REPO = "seshubommineni/python-project"
        PATH           = "/usr/local/bin:/usr/bin:/bin"
        K8S_DIR        = "k8s"
        FLUX_REPO      = "https://github.com/SeshuNaga/fluxrepo.git"
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/SeshuNaga/Python-project.git'
                script {
                    IMAGE_TAG = "${env.BUILD_NUMBER}" // Unique build ID
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${DOCKERHUB_REPO}:${IMAGE_TAG} ."
            }
        }

        stage('Login & Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub-creds', 
                                                 passwordVariable: 'DOCKER_PASS', 
                                                 usernameVariable: 'DOCKER_USER')]) {
                    sh "docker login -u $DOCKER_USER -p $DOCKER_PASS"
                    sh "docker push ${DOCKERHUB_REPO}:${IMAGE_TAG}"
                }
            }
        }

        stage('Update Flux Repo & Create PR') {
            steps {
                script {
                    sh """
                    # Clone the Flux repo if not already present
                    if [ ! -d fluxrepo ]; then
                        git clone ${FLUX_REPO}
                    fi
                    cd fluxrepo

                    # Checkout release branch (create if not exists)
                    git fetch origin
                    git checkout -B release origin/release || git checkout -b release

                    # Update the image tag in fastapi.yaml
                    sed -i 's|image: seshubommineni/python-project:.*|image: seshubommineni/python-project:${IMAGE_TAG}|' manifests/fastapi.yaml

                    # Commit and push changes to release branch
                    git add manifests/fastapi.yaml
                    git commit -m "Update FastAPI image to ${IMAGE_TAG}"
                    git push origin release

                    # Create PR to merge release into main using GitHub CLI
                    PR_URL=$(gh pr create --title "Update FastAPI image to ${IMAGE_TAG}" \
                          --body "Automatic image update from Jenkins build ${BUILD_NUMBER}" \
                          --base main \
                          --head release \
                          --json url -q .url)

                    echo "PR_URL=${PR_URL}" > pr_url.env
                    """
                }
            }
        }

        stage('Send PR URL Notification') {
            steps {
                script {
                    def prUrl = readFile('fluxrepo/pr_url.env').trim().split('=')[1]
                    echo "PR created: ${prUrl}"
                    // Example: send email
                    emailext(
                        subject: "New PR created for FastAPI image ${IMAGE_TAG}",
                        body: "A new PR has been created: ${prUrl}",
                        to: "team@example.com"
                    )
                }
            }
        }
    }

    post {
        success {
            echo "✅ Docker image pushed, Flux repo updated, PR created successfully!"
        }
        failure {
            echo "❌ Pipeline failed. Check logs."
        }
    }
}
