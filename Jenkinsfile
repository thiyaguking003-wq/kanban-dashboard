pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        IMAGE_REPO = 'kingpull/kanban-dashboard'
        APP_CONTAINER = 'kanban-dashboard'
        SMOKE_CONTAINER = 'kanban-smoke-test'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.IMAGE_TAG = sh(
                        script: 'git rev-parse --short HEAD',
                        returnStdout: true
                    ).trim()
                    echo "Building image tag: ${env.IMAGE_TAG}"
                }
            }
        }

        stage('Lint and Build') {
            steps {
                sh 'npm ci'
                sh 'npm run lint'
                sh 'npm run build'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build --pull \
                      -t "$IMAGE_REPO:$IMAGE_TAG" \
                      -t "$IMAGE_REPO:latest" .
                '''
            }
        }

        stage('Container Smoke Test') {
            steps {
                sh '''
                    set -eu
                    docker rm -f "$SMOKE_CONTAINER" >/dev/null 2>&1 || true
                    docker run -d --name "$SMOKE_CONTAINER" \
                      -p 8081:8080 "$IMAGE_REPO:$IMAGE_TAG"

                    for attempt in $(seq 1 30); do
                        if curl -fsS http://127.0.0.1:8081/health >/dev/null; then
                            echo "Smoke test passed"
                            docker rm -f "$SMOKE_CONTAINER" >/dev/null
                            exit 0
                        fi
                        sleep 2
                    done

                    docker logs "$SMOKE_CONTAINER" || true
                    docker rm -f "$SMOKE_CONTAINER" || true
                    echo "Smoke test failed"
                    exit 1
                '''
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKERHUB_USER',
                        passwordVariable: 'DOCKERHUB_TOKEN'
                    )
                ]) {
                    sh '''
                        set +x
                        printf '%s' "$DOCKERHUB_TOKEN" |
                          docker login -u "$DOCKERHUB_USER" --password-stdin
                        docker push "$IMAGE_REPO:$IMAGE_TAG"
                        docker push "$IMAGE_REPO:latest"
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                sh '''
                    set -eu

                    PREVIOUS_IMAGE="$(docker inspect \
                      --format='{{.Config.Image}}' "$APP_CONTAINER" 2>/dev/null || true)"

                    docker rm -f "$APP_CONTAINER" >/dev/null 2>&1 || true

                    if ! docker run -d --name "$APP_CONTAINER" \
                      --restart unless-stopped \
                      --memory=512m --cpus=0.5 \
                      -p 80:8080 "$IMAGE_REPO:$IMAGE_TAG"; then

                        if [ -n "$PREVIOUS_IMAGE" ]; then
                            docker run -d --name "$APP_CONTAINER" \
                              --restart unless-stopped \
                              -p 80:8080 "$PREVIOUS_IMAGE"
                        fi
                        exit 1
                    fi

                    HEALTHY=false
                    for attempt in $(seq 1 30); do
                        if curl -fsS http://127.0.0.1/health >/dev/null; then
                            HEALTHY=true
                            break
                        fi
                        sleep 2
                    done

                    if [ "$HEALTHY" != true ]; then
                        echo "Deployment health check failed"
                        docker logs "$APP_CONTAINER" || true
                        docker rm -f "$APP_CONTAINER" || true

                        if [ -n "$PREVIOUS_IMAGE" ]; then
                            docker run -d --name "$APP_CONTAINER" \
                              --restart unless-stopped \
                              -p 80:8080 "$PREVIOUS_IMAGE"
                            echo "Previous image restored"
                        fi
                        exit 1
                    fi

                    echo "Deployment successful: $IMAGE_REPO:$IMAGE_TAG"
                '''
            }
        }
    }

    post {
        always {
            sh 'docker rm -f kanban-smoke-test >/dev/null 2>&1 || true'
        }
    }
}
