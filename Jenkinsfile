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
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${DOCKERHUB_REPO}:${IMAGE_TAG} ."
            }
        }

        stage('Login & Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub-creds', 
                                                 passwordVariable: 'DOCKER_PASS', 
                                                 usernameVariable: 'DOCKER_USER')]) {
                    sh "docker login -u $DOCKER_USER -p $DOCKER_PASS"
                    sh "docker push ${DOCKERHUB_REPO}:${IMAGE_TAG}"
                }
            }
        }

        stage('Ensure PyYAML Installed') {
            steps {
                sh 'pip3 install --user pyyaml || true'
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

                        # Ensure we are on main before deleting release branch
                        git checkout main
                        git pull origin main

                        # Delete local release branch if exists
                        if git show-ref --verify --quiet refs/heads/release; then
                            git branch -D release
                        fi

                        # Create fresh release branch
                        git checkout -b release

                        # Update fastapi.yaml using Python
                        python3 - <<EOF
import yaml
file_path = 'manifests/fastapi.yaml'
image_tag = '${IMAGE_TAG}'

with open(file_path, 'r') as f:
    docs = list(yaml.safe_load_all(f))

for doc in docs:
    if 'spec' in doc and 'template' in doc['spec'] and 'spec' in doc['spec']['template']:
        containers = doc['spec']['template']['spec'].get('containers', [])
        for c in containers:
            if c.get('name') == 'fastapi':
                c['image'] = f'seshubommineni/python-project:{image_tag}'

with open(file_path, 'w') as f:
    yaml.dump_all(docs, f)
EOF

                        # Commit and push changes to release branch
                        git add manifests/fastapi.yaml
                        git commit -m "Update FastAPI image to ${IMAGE_TAG}"
                        git push origin release --force

                        # Create PR via GitHub API (safe JSON for macOS & Linux)
                        PR_URL=$(curl -s -X POST -H "Authorization: token ${GITHUB_TOKEN}" \
                            -H "Accept: application/vnd.github+json" \
                            https://api.github.com/repos/SeshuNaga/fluxrepo/pulls \
                            -d "{\"title\":\"Update FastAPI image to ${IMAGE_TAG}\",\"head\":\"release\",\"base\":\"main\",\"body\":\"Automatic image update from Jenkins build ${BUILD_NUMBER}\"}" \
                            | python3 -c "import sys, json; resp=json.load(sys.stdin); print(resp.get('html_url','ERROR_CREATING_PR'))")

                        echo "PR_URL=${PR_URL}" > pr_url.env
                        '''
                    }
                }
            }
        }

        stage('Send PR URL Notification') {
            steps {
                script {
                    def prUrl = readFile('fluxrepo/pr_url.env').trim().split('=')[1]
                    echo "PR created: ${prUrl}"
                    emailext(
                        subject: "New PR created for FastAPI image ${IMAGE_TAG}",
                        body: "A new PR has been created: ${prUrl}",
                        to: "team@example.com"
                    )
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
