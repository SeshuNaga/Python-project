pipeline {
    agent any

    environment {
        DOCKERHUB_REPO = "seshubommineni/python-project"
        IMAGE_TAG      = "latest"
        FLUX_REPO      = "https://github.com/SeshuNaga/fluxrepo.git"
        FLUX_DIR       = "fluxrepo"
        PATH           = "/usr/local/bin:/usr/bin:/bin:/opt/homebrew/bin"  // Include gh and docker
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
                    sh "docker build -t ${DOCKERHUB_REPO}:${IMAGE_TAG} ."
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

        stage('Update Flux Repo & Create PR') {
            steps {
                withCredentials([string(credentialsId: 'GITHUB_TOKEN', variable: 'GH_TOKEN')]) {
                    script {
                        sh """
                            set -e
                            # Ensure Homebrew binaries are in PATH
                            export PATH=/opt/homebrew/bin:\$PATH

                            # Clone or update flux repo
                            if [ -d "${FLUX_DIR}" ]; then
                                cd ${FLUX_DIR}
                                git fetch origin
                                git checkout main
                                git reset --hard origin/main
                                cd ..
                            fi

                            git clone ${FLUX_REPO} ${FLUX_DIR} || true
                            cd ${FLUX_DIR}

                            # Create or reset release branch
                            git checkout -B release

                            # Update fastapi.yaml image
                            sed -i.bak 's|image: .*|image: ${DOCKERHUB_REPO}:${IMAGE_TAG}|' manifests/fastapi.yaml
                            rm -f manifests/fastapi.yaml.bak

                            # Commit changes if any
                            git add manifests/fastapi.yaml || true
                            if git diff-index --quiet HEAD; then
                                echo "No changes to commit"
                            else
                                git commit -m "Update FastAPI image to ${IMAGE_TAG}"
                            fi

                            # Push release branch
                            git push origin release --force

                            # Create PR if not exists
                            export GH_TOKEN=$GH_TOKEN
                            EXISTING_PR=\$(gh pr list --head release --base main --json url -q '.[0].url')
                            if [ -z "\$EXISTING_PR" ]; then
                                PR_URL=\$(gh pr create \
                                    -t "Update FastAPI image to ${IMAGE_TAG}" \
                                    -b "Automatic image update from Jenkins build ${IMAGE_TAG}" \
                                    -B main -H release --json url -q '.url')
                            else
                                PR_URL=\$EXISTING_PR
                            fi

                            echo "✅ Pull Request URL: \$PR_URL"
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ Docker image pushed and PR created successfully!"
        }
        failure {
            echo "❌ Pipeline failed. Check logs."
        }
    }
}
