pipeline {
  agent any

  options {
    skipDefaultCheckout(true)
  }

  environment {
    IMAGE = 'jeongbam/bookridge-be'
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build image') {
      steps {
        script {
          env.IMAGE_TAG = sh(
            script: 'git rev-parse --short=12 HEAD',
            returnStdout: true
          ).trim()

          docker.build("${env.IMAGE}:${env.IMAGE_TAG}")
        }
      }
    }

    stage('Smoke test') {
      steps {
        sh '''
          set -eu
          docker rm -f be-test >/dev/null 2>&1 || true

          docker run -d --name be-test \
            -e JWT_SECRET=ci \
            -e PASSPORT_GOOGLE_CLIENT_ID=ci \
            -e PASSPORT_GOOGLE_CLIENT_SECRET=ci \
            -e GOOGLE_CALLBACK_URL=http://ci \
            -e AWS_REGION=ap-northeast-2 \
            -e AWS_S3_BUCKET=ci \
            "$IMAGE:$IMAGE_TAG"

          for attempt in $(seq 1 10); do
            if docker exec be-test wget -qO- http://localhost:3000/api/health | grep -q ok; then
              echo "Smoke test passed"
              exit 0
            fi
            sleep 2
          done

          docker logs be-test
          exit 1
        '''
      }
    }

    stage('Push image') {
      steps {
        script {
          docker.withRegistry('', 'dockerhub') {
            docker.image("${env.IMAGE}:${env.IMAGE_TAG}").push()

            if (env.BRANCH_NAME == 'main') {
              docker.image("${env.IMAGE}:${env.IMAGE_TAG}").push('latest')
            }
          }
        }
      }
    }
  }

  post {
    always {
      sh 'docker rm -f be-test >/dev/null 2>&1 || true'

      script {
        if (env.IMAGE_TAG) {
          sh "docker image rm ${env.IMAGE}:${env.IMAGE_TAG} >/dev/null 2>&1 || true"
        }
      }
    }
  }
}