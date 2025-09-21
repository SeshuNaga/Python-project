pipeline {
  agent any

  environment {
    DOCKERHUB_REPO = "seshubommineni/python-project"
    FLUX_REPO      = "SeshuNaga/fluxrepo"
    IMAGE_TAG      = "${BUILD_NUMBER}"
    FLUX_FILE_PATH = "manifests/fastapi.yaml"
  }

  stages {
    stage('Build & Push Docker Image') {
      steps {
        withCredentials([
          usernamePassword(credentialsId: 'docker-hub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')
        ]) {
          sh '''
            echo "Build tag = $IMAGE_TAG"
            docker build -t ${DOCKERHUB_REPO}:${IMAGE_TAG} .
            echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
            docker push ${DOCKERHUB_REPO}:${IMAGE_TAG}
          '''
        }
      }
    }

    stage('Update Flux Repo & Create PR') {
      steps {
        withCredentials([
          string(credentialsId: 'GITHUB_TOKEN', variable: 'GITHUB_TOKEN')
        ]) {
          script {
            sh '''
              set -e

              # Clone fluxrepo
              rm -rf fluxrepo
              git clone https://x-access-token:${GITHUB_TOKEN}@github.com/${FLUX_REPO}.git
              cd fluxrepo

              git checkout main
              git pull origin main

              # Delete local release branch if it exists
              if git show-ref --verify --quiet refs/heads/release; then
                git branch -D release
              fi

              git checkout -b release

              # Update the image tag in fastapi.yaml
              sed -i.bak "s|image: 'seshubommineni/python-project:.*'|image: 'seshubommineni/python-project:${IMAGE_TAG}'|g" ${FLUX_FILE_PATH}
              rm -f ${FLUX_FILE_PATH}.bak

              git add ${FLUX_FILE_PATH}
              if git diff --cached --quiet; then
                echo "No changes to commit; image already up-to-date"
                PR_URL="NO_CHANGES"
              else
                git commit -m "Update FastAPI image to ${IMAGE_TAG}"
                git push origin release --force

                # Create PR via GitHub API with correct head format
                PR_URL=$(curl -s -X POST \
                  -H "Authorization: token ${GITHUB_TOKEN}" \
                  -H "Accept: application/vnd.github+json" \
                  https://api.github.com/repos/${FLUX_REPO}/pulls \
                  -d "{\"title\":\"Update FastAPI image to ${IMAGE_TAG}\",\"head\":\"SeshuNaga:release\",\"base\":\"main\",\"body\":\"Automatic image update from build ${IMAGE_TAG}\"}" \
                  | python3 -c "import sys,json; resp=json.load(sys.stdin); print(resp.get('html_url','ERROR_CREATING_PR'))")
              fi

              echo "PR_URL=${PR_URL}" > ../pr_url.env
            '''
          }
        }
      }
    }

    stage('Print PR URL') {
      steps {
        script {
          def props = readFile('pr_url.env').trim().tokenize('=')  // expects format "PR_URL=..."
          if (props.size() == 2 && props[1] != "NO_CHANGES") {
            echo "✅ Pull Request URL: ${props[1]}"
          } else {
            echo "ℹ️ No new PR created (no changes)."
          }
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
