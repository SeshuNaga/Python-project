pipeline {
    agent any

    environment {
        DOCKERHUB_REPO = "seshubommineni/python-project"
        K8S_DIR        = "k8s"
        FLUX_REPO      = "https://github.com/SeshuNaga/fluxrepo.git"
        IMAGE_TAG      = "${BUILD_NUMBER}" // Jenkins build number
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
                    sh """
                    docker build -t ${DOCKERHUB_REPO}:${IMAGE_TAG} .
                    docker login -u $DOCKER_USER -p $DOCKER_PASS
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
                        # Clone or update Flux repo
                        if [ ! -d fluxrepo ]; then
                            git clone ${FLUX_REPO}
                        fi
                        cd fluxrepo
                        git fetch origin
                        git checkout main
                        git pull origin main

                        # Delete local release branch if exists
                        git branch -D release || true
                        git checkout -b release

                        # Update fastapi.yaml image using sed
                        sed -i.bak "s|image:.*|image: ${DOCKERHUB_REPO}:${IMAGE_TAG}|g" manifests/fastapi.yaml

                        git add manifests/fastapi.yaml
                        git commit -m "Update FastAPI image to ${IMAGE_TAG}"
                        git push origin release --force

                        # Create PR via GitHub API
                        PR_URL=\$(curl -s -X POST -H "Authorization: token ${GITHUB_TOKEN}" \
                            -H "Accept: application/vnd.github+json" \
                            https://api.github.com/repos/SeshuNaga/fluxrepo/pulls \
                            -d "{\\"title\\":\\"Update FastAPI image to ${IMAGE_TAG}\\",\\"head\\":\\"release\\",\\"base\\":\\"main\\",\\"body\\":\\"Automatic image update from Jenkins build ${BUILD_NUMBER}\\"}" \
                            | python3 -c "import sys,json; resp=json.load(sys.stdin); print(resp.get('html_url','ERROR_CREATING_PR'))")

                        echo "PR_URL=\${PR_URL}" > pr_url.env
                        """
                    }
                }
            }
        }

        stage('Notify') {
            steps {
                script {
                    def prUrl = readFile('fluxrepo/pr_url.env').trim().split('=')[1]
                    echo "PR created: ${prUrl}"
                    // Email step (replace with your config)
                    emailext(
                        subject: "New PR created for FastAPI image ${IMAGE_TAG}",
                        body: "PR URL: ${prUrl}",
                        to: "team@example.com"
                    )
                }
            }
        }
    }

    post {
        success { echo "✅ Build, push, PR creation successful!" }
        failure { echo "❌ Pipeline failed." }
    }
}
