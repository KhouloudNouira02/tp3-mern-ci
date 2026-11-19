
pipeline {
  agent any

  options {
    skipDefaultCheckout(true)
  }

  parameters {
    booleanParam(
      name: 'FORCE_ALL',
      defaultValue: false,
      description: 'Build all'
    )
  }

  triggers {
    pollSCM('H/5 * * * *')
  }

  environment {
    IMAGE_SERVER = 'khouloudnouira/mern-server'
    IMAGE_CLIENT = 'khouloudnouira/mern-client'
  }

  stages {
    stage('Checkout') {
      steps {
        git branch: 'main',
            url: 'https://github.com/KhouloudNouira02/tp3-mern-ci.git'
      }
    }

    stage('SERVER : build, scan, push') {
      when {
        anyOf {
          changeset 'server/**'
          changeset 'Jenkinsfile'
          expression { env.BUILD_NUMBER == '1' }
          expression { params.FORCE_ALL }
        }
      }

      steps {
        withCredentials([usernamePassword(
          credentialsId: 'dockerhub',
          usernameVariable: 'DH_USER',
          passwordVariable: 'DH_PASS'
        )]) {
          sh '''
            set -eu

            echo "=== BUILD SERVER ==="
            docker build -t "$IMAGE_SERVER:$BUILD_NUMBER" server

            echo "=== SCAN SERVER WITH TRIVY ==="
            docker run --rm \
              -v /var/run/docker.sock:/var/run/docker.sock \
              -v trivy-cache:/root/.cache/ \
              aquasec/trivy:0.76.2 image \
              --timeout 15m \
              "$IMAGE_SERVER:$BUILD_NUMBER" > trivy_server.txt

            cat trivy_server.txt

            echo "=== DOCKER HUB LOGIN ==="
            echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin

            echo "=== PUSH SERVER WITH RETRY ==="

            attempt=1
            until docker push "$IMAGE_SERVER:$BUILD_NUMBER"; do
              if [ "$attempt" -ge 3 ]; then
                echo "SERVER PUSH FAILED AFTER 3 ATTEMPTS"
                exit 1
              fi

              echo "SERVER PUSH FAILED - RETRY $attempt"
              attempt=$((attempt + 1))
              sleep 15
            done
          '''
        }
      }
    }

    stage('CLIENT : build, scan, push') {
      when {
        anyOf {
          changeset 'client/**'
          changeset 'Jenkinsfile'
          expression { env.BUILD_NUMBER == '1' }
          expression { params.FORCE_ALL }
        }
      }

      steps {
        withCredentials([usernamePassword(
          credentialsId: 'dockerhub',
          usernameVariable: 'DH_USER',
          passwordVariable: 'DH_PASS'
        )]) {
          sh '''
            set -eu

            echo "=== BUILD CLIENT ==="
            docker build -t "$IMAGE_CLIENT:$BUILD_NUMBER" client

            echo "=== SCAN CLIENT WITH TRIVY ==="
            docker run --rm \
              -v /var/run/docker.sock:/var/run/docker.sock \
              -v trivy-cache:/root/.cache/ \
              aquasec/trivy:0.76.2 image \
              --timeout 15m \
              "$IMAGE_CLIENT:$BUILD_NUMBER" > trivy_client.txt

            cat trivy_client.txt

            echo "=== DOCKER HUB LOGIN ==="
            echo "$DH_PASS" | docker login -u "$khouloudnouira" --password-stdin

            echo "=== PUSH CLIENT WITH RETRY ==="

            attempt=1
            until docker push "$IMAGE_CLIENT:$BUILD_NUMBER"; do
              if [ "$attempt" -ge 3 ]; then
                echo "CLIENT PUSH FAILED AFTER 3 ATTEMPTS"
                exit 1
              fi

              echo "CLIENT PUSH FAILED - RETRY $attempt"
              attempt=$((attempt + 1))
              sleep 15
            done
          '''
        }
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'trivy_*.txt',
                       allowEmptyArchive: true

      sh '''
        docker logout || true

        docker image rm "$IMAGE_SERVER:$BUILD_NUMBER" \
          "$IMAGE_CLIENT:$BUILD_NUMBER" 2>/dev/null || true
      '''
    }
  }
}
