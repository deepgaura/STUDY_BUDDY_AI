pipeline {
  agent any

  options {
    timestamps()
    ansiColor('xterm')
    skipDefaultCheckout(true)
  }

  environment {
    // --- change these IDs only if your Jenkins credentials use different names ---
    DOCKER_HUB_REPO           = 'deep2107/studybuddy'
    DOCKER_HUB_CREDENTIALS_ID = 'dockerhub-token'   // Docker Hub username/password/token
    GITHUB_CREDENTIALS_ID     = 'github-token'      // GitHub username/password (or PAT as password)
    IMAGE_TAG                 = "v${BUILD_NUMBER}"  // e.g., v23
    ARGOCD_HOST_PORT          = '34.67.42.132:31704'// your exposed Argo CD server
  }

  stages {

    stage('Checkout GitHub') {
      steps {
        echo 'Checking out code from GitHub...'
        git branch: 'main',
            url: 'https://github.com/deepgaura/STUDY_BUDDY_AI.git',
            credentialsId: "${GITHUB_CREDENTIALS_ID}"
      }
    }

    stage('Build Docker Image') {
      steps {
        script {
          echo "Building Docker image ${DOCKER_HUB_REPO}:${IMAGE_TAG} ..."
          dockerImage = docker.build("${DOCKER_HUB_REPO}:${IMAGE_TAG}")
        }
      }
    }

    stage('Push Image to DockerHub') {
      steps {
        script {
          echo 'Pushing Docker image to DockerHub...'
          docker.withRegistry('https://registry.hub.docker.com', "${DOCKER_HUB_CREDENTIALS_ID}") {
            dockerImage.push("${IMAGE_TAG}")
          }
        }
      }
    }

    stage('Update Deployment YAML with New Tag') {
      steps {
        sh """
          set -e
          # Replace the entire image line regardless of previous tag/digest
          sed -i -E "s|^(\\s*image:\\s*).+|\\1${DOCKER_HUB_REPO}:${IMAGE_TAG}|" manifests/deployment.yaml
          echo 'Updated image line:' && grep -n 'image:' manifests/deployment.yaml
        """
      }
    }

    stage('Commit Updated YAML') {
      steps {
        script {
          withCredentials([usernamePassword(
              credentialsId: "${GITHUB_CREDENTIALS_ID}",
              usernameVariable: 'GIT_USER',
              passwordVariable: 'GIT_PASS'
          )]) {
            sh """
              set -e
              git config user.name  "deepgaura"
              git config user.email "dipmahjn@gmail.com"
              git add manifests/deployment.yaml
              git commit -m "Update image to ${DOCKER_HUB_REPO}:${IMAGE_TAG}" || echo "No changes to commit"
              git push https://${GIT_USER}:${GIT_PASS}@github.com/deepgaura/STUDY_BUDDY_AI.git HEAD:main
            """
          }
        }
      }
    }

    stage('Install kubectl & argocd CLI') {
      steps {
        sh '''
          set -e
          echo "Installing kubectl & argocd CLI..."
          curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
          chmod +x kubectl && mv kubectl /usr/local/bin/kubectl
          curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
          chmod +x /usr/local/bin/argocd
          kubectl version --client || true
          argocd version --client || true
        '''
      }
    }

    stage('Apply via kubectl & Sync Argo CD') {
      steps {
        // You uploaded this as a Secret file with ID "kubeconfig"
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KCFG')]) {
          sh """
            set -e
            export KUBECONFIG="$KCFG"

            echo 'Kube sanity:'
            kubectl get ns
            kubectl -n argocd get pods

            echo 'Logging into Argo CD and syncing app...'
            ARGO_PASS=\$(kubectl -n argocd get secret argocd-initial-admin-secret \
              -o jsonpath='{.data.password}' | base64 -d)

            /usr/local/bin/argocd login ${ARGOCD_HOST_PORT} \
              --username admin --password "\$ARGO_PASS" --insecure

            /usr/local/bin/argocd app sync study

            echo 'Waiting for rollout of llmops-app...'
            kubectl -n argocd rollout status deploy/llmops-app
          """
        }
      }
    }
  }

  post {
    always {
      echo 'Cleaning up dangling images (best effort)...'
      sh 'docker image prune -f || true'
    }
  }
}
