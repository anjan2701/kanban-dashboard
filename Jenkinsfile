// Blue-green deployment pipeline (minimal / zero downtime)
//
// How it works:
//   * An nginx proxy container (kanban-proxy) owns port 80 and forwards to the
//     ACTIVE colour (kanban-blue or kanban-green) over a private Docker network.
//   * Each deploy starts the NEW colour next to the old one, health-checks it,
//     and only then switches the proxy (graceful nginx reload).
//   * The old container is removed only after the end-to-end check passes.
//   * If anything fails, traffic stays on (or returns to) the previous version.
//
// State file: $JENKINS_HOME/proxy/active  (contains "blue" or "green")

pipeline {
  agent any

  options {
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '20'))
  }

  environment {
    IMAGE     = "anjan27/kanban-dashboard"
    TAG       = "${env.BUILD_NUMBER}"
    NET       = "kanban-net"
    PROXY     = "kanban-proxy"
    PROXY_DIR = "${env.JENKINS_HOME}/proxy"
    CREDS     = credentials('dockerhub-creds')
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
        // remove markers left over from a previous build
        sh 'rm -f new_color.txt old_color.txt'
      }
    }

    stage('Build Image') {
      steps {
        sh 'docker build -t $IMAGE:$TAG .'
      }
    }

    stage('Push to Docker Hub') {
      steps {
        sh 'echo $CREDS_PSW | docker login -u $CREDS_USR --password-stdin'
        sh 'docker push $IMAGE:$TAG'
      }
    }

    stage('Deploy New Version (idle colour)') {
      steps {
        sh '''
          mkdir -p "$PROXY_DIR"
          docker network inspect "$NET" >/dev/null 2>&1 || docker network create "$NET"

          ACTIVE=$(cat "$PROXY_DIR/active" 2>/dev/null || echo none)
          if [ "$ACTIVE" = "blue" ]; then NEW=green; else NEW=blue; fi
          echo "$NEW" > new_color.txt
          echo "$ACTIVE" > old_color.txt
          echo "Active colour: $ACTIVE  ->  deploying: $NEW"

          docker rm -f "kanban-$NEW" >/dev/null 2>&1 || true
          docker run -d --name "kanban-$NEW" --network "$NET" --restart unless-stopped --memory=256m --cpus=0.5 "$IMAGE:$TAG"
        '''
      }
    }

    stage('Health Check New Version') {
      steps {
        sh '''
          NEW=$(cat new_color.txt)
          STATUS=starting
          for i in $(seq 1 18); do
            STATUS=$(docker inspect -f '{{.State.Health.Status}}' "kanban-$NEW")
            echo "Health of kanban-$NEW: $STATUS"
            [ "$STATUS" = "healthy" ] && break
            [ "$STATUS" = "unhealthy" ] && break
            sleep 5
          done
          if [ "$STATUS" != "healthy" ]; then
            echo "New container is NOT healthy - traffic was NOT switched"
            exit 1
          fi

          IP=$(docker inspect -f '{{(index .NetworkSettings.Networks "kanban-net").IPAddress}}' "kanban-$NEW")
          curl -fsS "http://$IP:8080/" > /dev/null
          echo "Direct HTTP check on new container passed"
        '''
      }
    }

    stage('Switch Traffic') {
      steps {
        sh '''
          NEW=$(cat new_color.txt)
          printf 'server { listen 8080; location / { proxy_pass http://kanban-%s:8080; proxy_set_header Host $host; } }' "$NEW" > "$PROXY_DIR/default.conf"

          if docker ps -a --format '{{.Names}}' | grep -qx "$PROXY"; then
            docker start "$PROXY" >/dev/null 2>&1 || true
            docker exec "$PROXY" nginx -s reload
          else
            echo "First blue-green deploy: freeing port 80 from the old single-container setup"
            docker rm -f kanban-app >/dev/null 2>&1 || true
            docker run -d --name "$PROXY" --network "$NET" --restart unless-stopped --memory=64m --cpus=0.25 -p 80:8080 -v "$PROXY_DIR/default.conf:/etc/nginx/conf.d/default.conf:ro" nginxinc/nginx-unprivileged:alpine
          fi

          echo "$NEW" > "$PROXY_DIR/active"
          echo "Traffic now routed to kanban-$NEW"
        '''
      }
    }

    stage('Verify via Port 80') {
      steps {
        sh '''
          for i in 1 2 3 4 5 6 7 8; do
            if curl -fsS http://localhost:80/ > /dev/null; then
              echo "End-to-end check passed through the proxy"
              exit 0
            fi
            sleep 2
          done
          echo "End-to-end check FAILED"
          exit 1
        '''
      }
    }

    stage('Remove Old Version') {
      steps {
        sh '''
          OLD=$(cat old_color.txt)
          if [ "$OLD" != "none" ]; then
            docker rm -f "kanban-$OLD" || true
            echo "Removed old container kanban-$OLD"
          fi
          docker image prune -f
        '''
      }
    }
  }

  post {
    success {
      echo "DEPLOYMENT SUCCESSFUL: ${env.IMAGE}:${env.TAG}"
    }

    failure {
      echo 'DEPLOYMENT FAILED - keeping or restoring the previous version'
      sh '''
        [ -f new_color.txt ] || exit 0
        NEW=$(cat new_color.txt)
        OLD=$(cat old_color.txt)
        ACTIVE=$(cat "$PROXY_DIR/active" 2>/dev/null || echo none)

        # Which colour should be serving traffic?
        if [ "$ACTIVE" = "$NEW" ]; then TARGET="$OLD"; else TARGET="$ACTIVE"; fi

        if [ "$TARGET" != "none" ] && docker ps --format '{{.Names}}' | grep -qx "kanban-$TARGET"; then
          printf 'server { listen 8080; location / { proxy_pass http://kanban-%s:8080; proxy_set_header Host $host; } }' "$TARGET" > "$PROXY_DIR/default.conf"
          docker exec "$PROXY" nginx -s reload || true
          echo "$TARGET" > "$PROXY_DIR/active"
          docker rm -f "kanban-$NEW" || true
          echo "Rollback complete: traffic is served by kanban-$TARGET"
        else
          echo "No previous version to roll back to; kanban-$NEW left in place for inspection"
        fi
      '''
    }

    always {
      sh 'docker logout || true'
    }
  }
}
