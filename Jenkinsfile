pipeline {
    agent any

    environment {
        DOCKERHUB_REPO = "seshubommineni/python-project"
        PATH           = "/usr/local/bin:/usr/bin:/bin"
        FLUX_REPO      = "https://github.com/SeshuNaga/fluxrepo.git"
    }

    stages {

        stage('Checkout Source Code') {
            steps {
                git branch: 'main', url: 'https://github.com/SeshuNaga/Python-project.git'
                script {
                    IMAGE_TAG = "${env.BUILD_NUMBER}" // unique build id
                }
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub-creds', 
                                                 usernameVariable: 'DOCKER_USER', 
                                                 passwordVariable: 'DOCKER_PASS')]) {
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
                        # Clone or fetch flux repo
                        if [ ! -d fluxrepo ]; then
                            git clone ${FLUX_REPO}
                        fi
                        cd fluxrepo
                        git fetch origin
                        git checkout main
                        git pull origin main

                        # Delete old release branch and create new
                        if git show-ref --verify --quiet refs/heads/release; then
                            git branch -D release
                        fi
                        git checkout -b release

                        # Update fastapi.yaml image
                        sed -i.bak "s|image: 'seshubommineni/python-project:.*'|image: 'seshubommineni/python-project:${IMAGE_TAG}'|g" manifests/fastapi.yaml
                        rm -f manifests/fastapi.yaml.bak

                        git add manifests/fastapi.yaml
                        git commit -m "Update FastAPI image to ${IMAGE_TAG}" || echo "No changes to commit"
                        git push origin release --force

                        # Check for existing PR
                        EXISTING_PR=\$(curl -s -H "Authorization: token ${GITHUB_TOKEN}" \
                            -H "Accept: application/vnd.github+json" \
                            "https://api.github.com/repos/SeshuNaga/fluxrepo/pulls?head=SeshuNaga:release&base=main" \
                            | python3 -c "import sys,json; prs=json.load(sys.stdin); print(prs[0]['html_url'] if prs else '')")

                        # Create PR if not exists
                        if [ -z "\$EXISTING_PR" ]; then
                            PR_URL=\$(curl -s -X POST -H "Authorization: token ${GITHUB_TOKEN}" \
                                -H "Accept: application/vnd.github+json" \
                                https://api.github.com/repos/SeshuNaga/fluxrepo/pulls \
                                -d "{\\"title\\":\\"Update FastAPI image to ${IMAGE_TAG}\\",\\"head\\":\\"release\\",\\"base\\":\\"main\\",\\"body\\":\\"Automatic image update from Jenkins build ${BUILD_NUMBER}\\"}" \
                                | python3 -c "import sys,json; resp=json.load(sys.stdin); print(resp.get('html_url','ERROR_CREATING_PR'))")
                        else
                            PR_URL=\$EXISTING_PR
                        fi

                        echo "PR_URL=\$PR_URL" > pr_url.env
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
            echo "Pipeline completed successfully."
        }
        failure {
            echo "Pipeline failed."
        }
    }
}
