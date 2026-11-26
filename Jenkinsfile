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
                    echo "Image tag: ${env.IMAGE_TAG}"
                }
            }
        }

        stage('Lint, Build and Validate') {
            steps {
                sh '''
                    set -eu
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
                      "$IMAGE_REPO:$IMAGE_TAG"

                    HEALTHY=false
                    for attempt in $(seq 1 30); do
                        if docker exec "$SMOKE_CONTAINER" \
                          wget -q -O /dev/null \
                          http://127.0.0.1:8080/health; then
                            HEALTHY=true
                            break
                        fi
                        sleep 2
                    done

                    docker rm -f "$SMOKE_CONTAINER" >/dev/null 2>&1 || true

                    if [ "$HEALTHY" != true ]; then
                        echo "Smoke test failed"
                        exit 1
                    fi
                    echo "Smoke test passed"
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
                        set -eu
                        set +x
                        trap 'docker logout >/dev/null 2>&1 || true' EXIT

                        printf '%s' "$DOCKERHUB_TOKEN" |
                          docker login -u "$DOCKERHUB_USER" --password-stdin

                        docker push "$IMAGE_REPO:$IMAGE_TAG"
                        docker push "$IMAGE_REPO:latest"
                    '''
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                sh '''
                    set -eu

                    PREVIOUS_IMAGE="$(docker inspect \
                      --format='{{.Config.Image}}' \
                      "$APP_CONTAINER" 2>/dev/null || true)"

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
                        if docker exec "$APP_CONTAINER" \
                          wget -q -O /dev/null \
                          http://127.0.0.1:8080/health; then
                            HEALTHY=true
                            break
                        fi
                        sleep 2
                    done

                    if [ "$HEALTHY" != true ]; then
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

                    echo "Deployment successful"
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
