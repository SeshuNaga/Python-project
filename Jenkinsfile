pipeline {
    agent any

    environment {
        DOCKERHUB_REPO = "seshubommineni/python-project"
        FLUX_REPO      = "https://github.com/SeshuNaga/fluxrepo.git"
        IMAGE_TAG      = "${BUILD_NUMBER}" // Jenkins build number
        PYTHON_REPO    = "https://github.com/SeshuNaga/Python-project.git"
    }

    stages {
        stage('Checkout Python Source') {
            steps {
                dir('python-project') {
                    git branch: 'main', url: "${PYTHON_REPO}"
                }
            }
        }
        stage('Build & Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub-creds',
                                                 passwordVariable: 'DOCKER_PASS',
                                                 usernameVariable: 'DOCKER_USER')]) {
                    dir('python-project') {
                        sh '''
                        echo "Logging into Docker Hub"
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        echo "Building Docker image"
                        docker build -t ${DOCKERHUB_REPO}:${IMAGE_TAG} .
                        echo "Pushing Docker image"
                        docker push ${DOCKERHUB_REPO}:${IMAGE_TAG}
                        '''
                    }
                }
            }
        }
        stage('Update Flux Repo & Create PR') {
            steps {
                withCredentials([string(credentialsId: 'GITHUB_TOKEN', variable: 'GITHUB_TOKEN')]) {
                    dir('fluxrepo') {
                        sh '''
                        # Clone Flux repo if not exists
                        if [ ! -d .git ]; then
                            git clone ${FLUX_REPO} .
                        fi
                        git fetch origin
                        git checkout main
                        git pull origin main

                        # Delete or create release branch
                        git branch -D release || true
                        git checkout -b release

                        # Update fastapi.yaml image using sed and remove backup
                        sed -i.bak "s|image: 'seshubommineni/python-project:.*'|image: 'seshubommineni/python-project:${IMAGE_TAG}'|" manifests/fastapi.yaml
                        rm -f manifests/fastapi.yaml.bak

                        # Commit and push changes
                        git add manifests/fastapi.yaml
                        git commit -m "Update FastAPI image to ${IMAGE_TAG}" || echo "No changes to commit"
                        git push origin release --force

                        # Check if PR already exists
                        EXISTING_PR=$(curl -s -H "Authorization: token ${GITHUB_TOKEN}" \
                            -H "Accept: application/vnd.github+json" \
                            "https://api.github.com/repos/SeshuNaga/fluxrepo/pulls?head=SeshuNaga:release&base=main" \
                            | python3 -c "import sys,json; prs=json.load(sys.stdin); print(prs[0]['html_url'] if prs else '')")

                        if [ -z "$EXISTING_PR" ]; then
                            PR_URL=$(curl -s -X POST -H "Authorization: token ${GITHUB_TOKEN}" \
                                -H "Accept: application/vnd.github+json" \
                                https://api.github.com/repos/SeshuNaga/fluxrepo/pulls \
                                -d "{\\"title\\":\\"Update FastAPI image to ${IMAGE_TAG}\\",\\"head\\":\\"release\\",\\"base\\":\\"main\\",\\"body\\":\\"Automatic image update from Jenkins build ${BUILD_NUMBER}\\"}" \
                                | python3 -c "import sys,json; resp=json.load(sys.stdin); print(resp.get('html_url','ERROR_CREATING_PR'))")
                        else
                            PR_URL="$EXISTING_PR"
                        fi

                        echo "PR created or exists: ${PR_URL}"
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
