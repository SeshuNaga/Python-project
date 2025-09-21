pipeline {
    agent any

    environment {
        DOCKERHUB_REPO = "seshubommineni/python-project"
        IMAGE_TAG      = "latest"
        PATH           = "/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin"  // Include gh & docker
        FLUX_REPO      = "https://github.com/SeshuNaga/fluxrepo.git"
        FLUX_DIR       = "fluxrepo"
    }

    stages {
        stage('Checkout Python Code') {
            steps {
                git branch: 'main', url: 'https://github.com/SeshuNaga/Python-project.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "🚀 Building Docker image..."
                sh "docker build -t ${DOCKERHUB_REPO}:${IMAGE_TAG} ."
            }
        }

        stage('Login & Push to Docker Hub') {
            steps {
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

        stage('Update Flux Repo & Create PR') {
            steps {
                // Use GH_TOKEN from Jenkins Credentials
                withCredentials([string(credentialsId: 'GITHUB_TOKEN', variable: 'GH_TOKEN')]) {
                    sh '''
                        set -e

                        # Clone flux repo if not exists
                        if [ ! -d ${FLUX_DIR} ]; then
                            git clone ${FLUX_REPO} ${FLUX_DIR}
                        fi

                        cd ${FLUX_DIR}
                        git fetch origin
                        git checkout -B release origin/main || git checkout -B release

                        # Update fastapi.yaml image
                        sed -i.bak "s|image: '.*'|image: '${DOCKERHUB_REPO}:${IMAGE_TAG}'|g" manifests/fastapi.yaml
                        rm -f manifests/fastapi.yaml.bak

                        git add manifests/fastapi.yaml
                        if git diff --cached --quiet; then
                            echo "No changes to commit"
                        else
                            git commit -m "Update FastAPI image to ${IMAGE_TAG}"
                            git push origin release --force
                        fi

                        # Check existing PR
                        EXISTING_PR=$(gh pr list --head release --base main --json url -q '.[0].url')
                        if [ -z "$EXISTING_PR" ]; then
                            echo "No existing PR, creating a new one..."
                            gh pr create --title "Update FastAPI image to ${IMAGE_TAG}" \
                                         --body "Automatic image update from Jenkins build ${IMAGE_TAG}" \
                                         --head release --base main
                        else
                            echo "Existing PR: $EXISTING_PR"
                        fi
                    '''
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
