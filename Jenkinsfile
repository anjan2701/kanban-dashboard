pipeline {
  agent any

  environment {
    IMAGE     = "anjan27/kanban-dashboard"
    TAG       = "${env.BUILD_NUMBER}"
    CONTAINER = "kanban-app"
    CREDS     = credentials('dockerhub-creds')
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build Image') {
      steps { sh 'docker build -t $IMAGE:$TAG .' }
    }

    stage('Push to Docker Hub') {
      steps {
        sh 'echo $CREDS_PSW | docker login -u $CREDS_USR --password-stdin'
        sh 'docker push $IMAGE:$TAG'
      }
    }

    stage('Deploy') {
      steps {
        sh '''
          docker inspect -f '{{.Config.Image}}' $CONTAINER > prev_image.txt 2>/dev/null || true
          docker rm -f $CONTAINER || true
          docker run -d --name $CONTAINER --restart unless-stopped \
            --memory=256m --cpus=0.5 -p 80:8080 $IMAGE:$TAG
        '''
      }
    }

    stage('Health Check') {
      steps {
        sh '''
          STATUS=starting
          for i in $(seq 1 12); do
            STATUS=$(docker inspect -f '{{.State.Health.Status}}' $CONTAINER)
            echo "Health: $STATUS"
            [ "$STATUS" = "healthy" ] && break
            sleep 5
          done
          [ "$STATUS" = "healthy" ] || exit 1
          curl -f http://localhost:80/
        '''
      }
    }
  }

  post {
    success { echo 'DEPLOYMENT SUCCESSFUL' }
    failure {
      echo 'DEPLOYMENT FAILED - rolling back'
      sh '''
        PREV=$(cat prev_image.txt 2>/dev/null || true)
        docker rm -f $CONTAINER || true
        if [ -n "$PREV" ]; then
          docker run -d --name $CONTAINER --restart unless-stopped \
            --memory=256m --cpus=0.5 -p 80:8080 $PREV
        fi
      '''
    }
    always { sh 'docker logout || true' }
  }
}
