pipeline {
    agent any

    environment {
        DOCKERHUB_REPO = "seshubommineni/python-project"
        IMAGE_TAG      = "latest"
        FLUX_REPO      = "https://github.com/SeshuNaga/fluxrepo.git"
        RELEASE_BRANCH = "release"
        PATH           = "/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin" // ensure Jenkins sees gh & docker
    }

    stages {
        stage('Checkout Python Code') {
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

        stage('Push Docker Image') {
            steps {
                script {
                    echo "🔑 Logging in to Docker Hub..."
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-creds',
                                                     passwordVariable: 'DOCKER_PASS',
                                                     usernameVariable: 'DOCKER_USER')]) {
                        sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin"
                    }

                    echo "📦 Pushing Docker image..."
                    sh "docker push ${DOCKERHUB_REPO}:${IMAGE_TAG}"
                }
            }
        }

        stage('Update Flux Repo & Create PR') {
            steps {
                script {
                    // Clone flux repo
                    sh "git clone ${FLUX_REPO} fluxrepo"
                    dir("fluxrepo") {
                        // Delete and recreate release branch
                        sh "git fetch origin"
                        sh "git branch -D ${RELEASE_BRANCH} || true"
                        sh "git checkout -b ${RELEASE_BRANCH} origin/main"

                        // Update FastAPI image
                        sh "sed -i.bak 's|image: .*|image: ${DOCKERHUB_REPO}:${IMAGE_TAG}|g' manifests/fastapi.yaml"
                        sh "rm -f manifests/fastapi.yaml.bak"

                        // Commit changes if any
                        sh "git add manifests/fastapi.yaml"
                        sh "git diff --cached --quiet || git commit -m 'Update FastAPI image to ${IMAGE_TAG}'"

                        // Push branch
                        sh "git push origin ${RELEASE_BRANCH} --force"

                        // Create PR using GitHub CLI
                        withCredentials([string(credentialsId: 'github-token', variable: 'GH_TOKEN')]) {
                            // Set GH authentication
                            sh "gh auth login --with-token <<< $GH_TOKEN"

                            // Check if PR exists
                            def existingPR = sh(script: "/opt/homebrew/bin/gh pr list --head ${RELEASE_BRANCH} --base main --json url -q '.[0].url'", returnStdout: true).trim()
                            if (existingPR) {
                                echo "✅ PR already exists: ${existingPR}"
                            } else {
                                def prUrl = sh(script: "/opt/homebrew/bin/gh pr create --title 'Update FastAPI image to ${IMAGE_TAG}' --body 'Automatic image update from Jenkins build ${IMAGE_TAG}' --head ${RELEASE_BRANCH} --base main --json url -q '.url'", returnStdout: true).trim()
                                echo "✅ PR created: ${prUrl}"
                            }
                        }
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline completed successfully!"
        }
        failure {
            echo "❌ Pipeline failed. Check logs."
        }
    }
}
