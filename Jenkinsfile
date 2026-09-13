pipeline {
    agent any

    environment {
        IMAGE_NAME = 'voting-app-worker'
        IMAGE_TAG  = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Validate') {
            steps {
                sh '''
                    set -e
                    test -f Worker.csproj
                    test -f Program.cs
                    test -f Dockerfile
                    echo "Worker project structure validated"
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    set -e
                    docker buildx build \
                      --load \
                      -t "${IMAGE_NAME}:${IMAGE_TAG}" \
                      -t "${IMAGE_NAME}:latest" \
                      .
                '''
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-worker',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {
                    sh '''
                        set -e

                        IMAGE="${DOCKER_USERNAME}/${IMAGE_NAME}"

                        echo "$DOCKER_PASSWORD" | docker login \
                          --username "$DOCKER_USERNAME" \
                          --password-stdin

                        docker tag "${IMAGE_NAME}:${IMAGE_TAG}" "${IMAGE}:${IMAGE_TAG}"
                        docker tag "${IMAGE_NAME}:latest" "${IMAGE}:latest"

                        docker push "${IMAGE}:${IMAGE_TAG}"
                        docker push "${IMAGE}:latest"

                        docker logout
                    '''
                }
            }
        }

        stage('Update GitOps') {
            steps {
                sh '''
                    set -e

                    IMAGE="${DOCKER_USERNAME}/${IMAGE_NAME}"

                    sed -i "s|image: .*|image: ${IMAGE}:${IMAGE_TAG}|" \
                      k8s/deployment.yaml

                    git config user.name "jenkins"
                    git config user.email "jenkins@localhost"

                    git add k8s/deployment.yaml
                    git commit -m "Update worker image to ${IMAGE_TAG}" || exit 0
                    git push origin HEAD:main
                '''
            }
        }
    }

    post {
        always {
            echo "Worker Jenkins CI/CD pipeline completed."
        }
    }
}
