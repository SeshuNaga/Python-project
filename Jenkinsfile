pipeline {
    agent any

    environment {
        DOCKERHUB_REPO = "seshubommineni/python-project"
        K8S_DIR        = "k8s"
        FLUX_REPO      = "https://github.com/SeshuNaga/fluxrepo.git"
        IMAGE_TAG      = "${BUILD_NUMBER}" // Jenkins build number
        PATH           = "/usr/local/bin:/usr/bin:/bin:$PATH"
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/SeshuNaga/Python-project.git'
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub-creds',
                                                 passwordVariable: 'DOCKER_PASS',
                                                 usernameVariable: 'DOCKER_USER')]) {
                    sh '''
                    echo "Logging into Docker Hub"
                    docker login -u "$DOCKER_USER" -p "$DOCKER_PASS"
                    echo "Building Docker image"
                    docker build -t ${DOCKERHUB_REPO}:${IMAGE_TAG} .
                    echo "Pushing Docker image"
                    docker push ${DOCKERHUB_REPO}:${IMAGE_TAG}
                    '''
                }
            }
        }

        stage('Update Flux Repo & Create PR') {
            steps {
                withCredentials([string(credentialsId: 'GITHUB_TOKEN', variable: 'GITHUB_TOKEN')]) {
                    script {
                        sh '''
                        # Clone Flux repo if not exists
                        if [ ! -d fluxrepo ]; then
                            git clone ${FLUX_REPO}
                        fi
                        cd fluxrepo
                        git fetch origin
                        git checkout main
                        git pull origin main

                        # Delete or create release branch
                        git branch -D release || true
                        git checkout -b release

                        # Update fastapi.yaml image using sed
                        sed -i.bak "s|image: 'seshubommineni/python-project:.*'|image: 'seshubommineni/python-project:${IMAGE_TAG}'|" manifests/fastapi.yaml

                        # Commit and push changes
                        git add manifests/fastapi.yaml
                        git commit -m "Update FastAPI image to ${IMAGE_TAG}" || echo "No changes to commit"
                        git push origin release --force

                        # Create PR via GitHub API
                        PR_URL=$(curl -s -X POST -H "Authorization: token ${GITHUB_TOKEN}" \\
                            -H "Accept: application/vnd.github+json" \\
                            https://api.github.com/repos/SeshuNaga/fluxrepo/pulls \\
                            -d "{\\"title\\":\\"Update FastAPI image to ${IMAGE_TAG}\\",\\"head\\":\\"release\\",\\"base\\":\\"main\\",\\"body\\":\\"Automatic image update from Jenkins build ${BUILD_NUMBER}\\"}" \\
                            | python3 -c "import sys,json; resp=json.load(sys.stdin); print(resp.get('html_url','ERROR_CREATING_PR'))")

                        echo "PR created: ${PR_URL}"
                        '''
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
