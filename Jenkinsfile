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
                    IMAGE_TAG = "${env.BUILD_NUMBER}" // Unique build ID per Jenkins build
                    echo "Build tag will be: ${IMAGE_TAG}"
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "Building Docker image ${DOCKERHUB_REPO}:${IMAGE_TAG}"
                sh "docker build -t ${DOCKERHUB_REPO}:${IMAGE_TAG} ."
            }
        }

        stage('Login & Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub-creds',
                                                 usernameVariable: 'DOCKER_USER',
                                                 passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        docker login -u $DOCKER_USER -p $DOCKER_PASS
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
                        # Clone or update flux repo
                        if [ ! -d fluxrepo ]; then
                            git clone ${FLUX_REPO}
                        fi
                        cd fluxrepo
                        git fetch origin
                        git checkout main
                        git pull origin main

                        # Delete old release branch if exists
                        if git show-ref --verify --quiet refs/heads/release; then
                            git branch -D release
                        fi
                        git checkout -b release

                        # Update fastapi.yaml with new image tag
                        sed -i.bak "s|image: 'seshubommineni/python-project:.*'|image: 'seshubommineni/python-project:${IMAGE_TAG}'|g" manifests/fastapi.yaml
                        rm -f manifests/fastapi.yaml.bak

                        # Commit and push changes
                        git add -A
                        git commit -m "Update FastAPI image to ${IMAGE_TAG}" || echo "No changes to commit"
                        git push origin release --force

                        # Create PR safely using jq for proper JSON
                        DATA=$(jq -n --arg title "Update FastAPI image to ${IMAGE_TAG}" \
                                      --arg head "release" \
                                      --arg base "main" \
                                      --arg body "Automatic image update from Jenkins build ${BUILD_NUMBER}" \
                                      '{title:$title, head:$head, base:$base, body:$body}')

                        PR_URL=$(curl -s -X POST -H "Authorization: token ${GITHUB_TOKEN}" \
                            -H "Accept: application/vnd.github+json" \
                            https://api.github.com/repos/SeshuNaga/fluxrepo/pulls \
                            -d "$DATA" | python3 -c "import sys,json; resp=json.load(sys.stdin); print(resp.get('html_url','ERROR_CREATING_PR'))")

                        echo "PR_URL=${PR_URL}" > pr_url.env
                        '''
                    }
                }
            }
        }

        stage('Print PR URL') {
            steps {
                script {
                    def prUrl = readFile('fluxrepo/pr_url.env').trim().split('=')[1]
                    echo "✅ Pull Request URL: ${prUrl}"
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
