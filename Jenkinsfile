pipeline {
    agent any

    environment {
        DOCKERHUB_REPO = "seshubommineni/python-project"
        PATH           = "/usr/local/bin:/usr/bin:/bin"
        K8S_DIR        = "k8s"
        FLUX_REPO      = "https://github.com/SeshuNaga/fluxrepo.git"
        PYTHON_REPO    = "https://github.com/SeshuNaga/Python-project.git"
    }

    stages {

        stage('Checkout Python Code & Set Build Tag') {
            steps {
                git branch: 'main', url: "${PYTHON_REPO}"
                script {
                    IMAGE_TAG = "${env.BUILD_NUMBER}" // Unique Docker build ID
                    echo "Build tag set to: ${IMAGE_TAG}"
                }
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub-creds', 
                                                 passwordVariable: 'DOCKER_PASS', 
                                                 usernameVariable: 'DOCKER_USER')]) {
                    sh """
                        docker login -u $DOCKER_USER -p $DOCKER_PASS
                        docker build -t ${DOCKERHUB_REPO}:${IMAGE_TAG} .
                        docker push ${DOCKERHUB_REPO}:${IMAGE_TAG}
                    """
                }
            }
        }

        stage('Update Flux Repo & Create PR') {
            steps {
                withCredentials([string(credentialsId: 'GITHUB_TOKEN', variable: 'GITHUB_TOKEN')]) {
                    script {
                        sh """
                        # Clone Flux repo if not exists
                        if [ ! -d fluxrepo ]; then
                            git clone ${FLUX_REPO}
                        fi
                        cd fluxrepo
                        git fetch origin

                        # Switch to main and pull latest
                        git checkout main
                        git pull origin main

                        # Delete local release branch if exists
                        git branch -D release || true
                        git checkout -b release

                        # Update fastapi.yaml with new build tag
                        sed -i.bak "s|image: 'seshubommineni/python-project:.*'|image: 'seshubommineni/python-project:${IMAGE_TAG}'|g" manifests/fastapi.yaml
                        rm -f manifests/fastapi.yaml.bak

                        git add manifests/fastapi.yaml
                        git commit -m "Update FastAPI image to ${IMAGE_TAG}" || echo "No changes to commit"

                        git push origin release --force

                        # Check if a PR already exists
                        EXISTING_PR=\$(gh pr list --head release --base main --json url -q '.[0].url')
                        if [ -z "\$EXISTING_PR" ]; then
                            PR_URL=\$(gh pr create --title "Update FastAPI image to ${IMAGE_TAG}" \
                              --body "Automatic image update from Jenkins build ${BUILD_NUMBER}" \
                              --head release --base main --json url -q '.url')
                        else
                            PR_URL="\$EXISTING_PR"
                        fi

                        echo "PR created: \$PR_URL"
                        """
                    }
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
