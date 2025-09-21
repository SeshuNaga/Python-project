pipeline {
    agent any

    environment {
        DOCKERHUB_REPO = "seshubommineni/python-project"
        PATH           = "/usr/local/bin:/usr/bin:/bin"
        FLUX_REPO      = "https://github.com/SeshuNaga/fluxrepo.git"
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/SeshuNaga/Python-project.git'
                script {
                    IMAGE_TAG = "${env.BUILD_NUMBER}" // Unique build ID per Jenkins build
                }
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub-creds', 
                                                 passwordVariable: 'DOCKER_PASS', 
                                                 usernameVariable: 'DOCKER_USER')]) {
                    sh """
                        echo "Building Docker image"
                        docker build -t ${DOCKERHUB_REPO}:${IMAGE_TAG} .
                        echo "Logging into Docker Hub"
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
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
                        # Clone flux repo if it doesn't exist
                        if [ ! -d fluxrepo ]; then
                            git clone ${FLUX_REPO}
                        fi
                        cd fluxrepo

                        git fetch origin
                        git checkout main
                        git pull origin main

                        # Delete local release branch if exists
                        if git show-ref --verify --quiet refs/heads/release; then
                            git branch -D release
                        fi

                        # Create fresh release branch
                        git checkout -b release

                        # Update fastapi.yaml image
                        sed -i.bak 's|image: '\''seshubommineni/python-project:.*'\''|image: '\''seshubommineni/python-project:${IMAGE_TAG}'\''|g' manifests/fastapi.yaml
                        rm -f manifests/fastapi.yaml.bak

                        # Commit only if there are changes
                        if git diff --quiet; then
                            echo "No changes to commit, skipping PR creation"
                            PR_URL="NO_CHANGES"
                        else
                            git add manifests/fastapi.yaml
                            git commit -m "Update FastAPI image to ${IMAGE_TAG}"
                            git push origin release --force

                            # Create PR via GitHub API
                            PR_URL=\$(curl -s -X POST \
                                -H "Authorization: token ${GITHUB_TOKEN}" \
                                -H "Accept: application/vnd.github+json" \
                                https://api.github.com/repos/SeshuNaga/fluxrepo/pulls \
                                -d "{\"title\":\"Update FastAPI image to ${IMAGE_TAG}\",\"head\":\"SeshuNaga:release\",\"base\":\"main\",\"body\":\"Automatic image update from Jenkins build ${BUILD_NUMBER}\"}" \
                                | python3 -c "import sys,json; resp=json.load(sys.stdin); print(resp.get('html_url','ERROR_CREATING_PR'))")
                        fi

                        echo "PR_URL=${PR_URL}" > pr_url.env
                        """
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
