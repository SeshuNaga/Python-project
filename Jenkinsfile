pipeline {
    agent any

    environment {
        DOCKERHUB_REPO = "seshubommineni/python-project"
        PATH           = "/usr/local/bin:/usr/bin:/bin"
        K8S_DIR        = "k8s"
        FLUX_REPO      = "https://github.com/SeshuNaga/fluxrepo.git"
        IMAGE_TAG      = "${BUILD_NUMBER}" // Jenkins build number as tag
    }

    stages {

        stage('Checkout Source Code') {
            steps {
                git branch: 'main', url: 'https://github.com/SeshuNaga/Python-project.git'
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub-creds', 
                                                 usernameVariable: 'DOCKER_USER', 
                                                 passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                    docker login -u $DOCKER_USER -p $DOCKER_PASS
                    docker build -t ${DOCKERHUB_REPO}:${IMAGE_TAG} .
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
                        # Clone or fetch Flux repo
                        if [ ! -d fluxrepo ]; then
                            git clone ${FLUX_REPO}
                        fi
                        cd fluxrepo
                        git fetch origin
                        git checkout main
                        git pull origin main

                        # Reset local release branch
                        if git show-ref --verify --quiet refs/heads/release; then
                            git branch -D release
                        fi
                        git checkout -b release

                        # Update fastapi.yaml with new image tag
                        python3 - <<EOF
import yaml
file_path = 'manifests/fastapi.yaml'
image_tag = '${IMAGE_TAG}'

with open(file_path, 'r') as f:
    docs = list(yaml.safe_load_all(f))

for doc in docs:
    if doc.get('kind') == 'Deployment' and doc.get('metadata', {}).get('name') == 'fastapi':
        containers = doc['spec']['template']['spec']['containers']
        for c in containers:
            if c.get('name') == 'fastapi':
                c['image'] = f'seshubommineni/python-project:{image_tag}'

with open(file_path, 'w') as f:
    yaml.dump_all(docs, f, default_flow_style=False)
EOF

                        git add manifests/fastapi.yaml
                        git commit -m "Update FastAPI image to ${IMAGE_TAG}" || echo "No changes to commit"
                        git push origin release --force

                        # Create PR via GitHub API if it doesn't exist
                        EXISTING_PR=$(curl -s -H "Authorization: token ${GITHUB_TOKEN}" \
                            -H "Accept: application/vnd.github+json" \
                            "https://api.github.com/repos/SeshuNaga/fluxrepo/pulls?head=SeshuNaga:release&base=main" \
                            | python3 -c "import sys,json; prs=json.load(sys.stdin); print(prs[0]['html_url'] if prs else '')")

                        if [ -z "$EXISTING_PR" ]; then
                            PR_URL=$(curl -s -X POST -H "Authorization: token ${GITHUB_TOKEN}" \
                                -H "Accept: application/vnd.github+json" \
                                https://api.github.com/repos/SeshuNaga/fluxrepo/pulls \
                                -d "{\"title\":\"Update FastAPI image to ${IMAGE_TAG}\",\"head\":\"release\",\"base\":\"main\",\"body\":\"Automatic image update from Jenkins build ${BUILD_NUMBER}\"}" \
                                | python3 -c "import sys,json; resp=json.load(sys.stdin); print(resp.get('html_url','ERROR_CREATING_PR'))")
                        else
                            PR_URL=$EXISTING_PR
                        fi

                        echo "PR_URL=${PR_URL}"
                        '''
                    }
                }
            }
        }

        stage('Show PR URL') {
            steps {
                script {
                    echo "✅ Pull Request URL for this build:"
                    sh 'cat fluxrepo/PR_URL || echo ${PR_URL}'
                }
            }
        }
    }

    post {
        success {
            echo "✅ Docker image pushed and Flux repo updated successfully!"
        }
        failure {
            echo "❌ Pipeline failed. Check logs."
        }
    }
}
