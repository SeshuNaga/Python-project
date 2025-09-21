pipeline {
    agent any

    environment {
        DOCKERHUB_REPO = "seshubommineni/python-project"
        IMAGE_TAG      = "latest"
        PATH           = "/usr/local/bin:/usr/bin:/bin"  // Ensure Jenkins finds docker & gh
        FLUX_DIR       = "manifests"                    // Your Flux YAML folder
        GITHUB_REPO    = "SeshuNaga/fluxrepo"          // Flux repo
        BRANCH_NAME    = "release"
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
                    sh "docker --version"
                    sh "docker build -t ${DOCKERHUB_REPO}:${IMAGE_TAG} ."
                }
            }
        }

        stage('Login & Push to Docker Hub') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-creds', 
                                                     usernameVariable: 'DOCKER_USER', 
                                                     passwordVariable: 'DOCKER_PASS')]) {
                        sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin"
                    }
                    sh "docker push ${DOCKERHUB_REPO}:${IMAGE_TAG}"
                }
            }
        }

        stage('Update Flux Repo & Create PR') {
            steps {
                script {
                    // Clone Flux repo
                    sh "rm -rf fluxrepo"
                    sh "git clone https://github.com/${GITHUB_REPO}.git fluxrepo"
                    dir('fluxrepo') {
                        // Checkout release branch
                        sh """
                        git fetch origin
                        git checkout -B ${BRANCH_NAME} origin/main
                        """

                        // Update image in manifest
                        sh """
                        sed -i.bak 's|image:.*|image: ${DOCKERHUB_REPO}:${IMAGE_TAG}|g' ${FLUX_DIR}/fastapi.yaml
                        rm -f ${FLUX_DIR}/fastapi.yaml.bak
                        """

                        // Commit changes
                        sh """
                        git add ${FLUX_DIR}/fastapi.yaml
                        git diff --cached --quiet || git commit -m 'Update FastAPI image to ${IMAGE_TAG}'
                        git push origin ${BRANCH_NAME} --force
                        """

                        // Create PR using GitHub CLI
                        def pr_url = sh(
                            script: "gh pr list --head ${BRANCH_NAME} --base main --json url -q '.[0].url'",
                            returnStdout: true
                        ).trim()

                        if (pr_url == "") {
                            pr_url = sh(
                                script: "gh pr create --title 'Update FastAPI image to ${IMAGE_TAG}' --body 'Automatic image update from Jenkins build ${IMAGE_TAG}' --head ${BRANCH_NAME} --base main --json url -q '.url'",
                                returnStdout: true
                            ).trim()
                        }

                        echo "✅ Pull Request URL: ${pr_url}"
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ Docker image pushed and Flux PR created successfully!"
        }
        failure {
            echo "❌ Pipeline failed. Check logs."
        }
    }
}
