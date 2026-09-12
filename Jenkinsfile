pipeline {
    agent any

    environment {
        IMAGE_NAME = 'voting-app-worker'
        IMAGE_TAG  = "${BUILD_NUMBER}"
        DC_HOME    = '/opt/dependency-check'
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

        stage('OWASP Dependency Check') {
            steps {
                sh '''
                    set -e

                    echo "Running OWASP Dependency-Check..."

                    rm -rf dependency-check-report
                    mkdir -p dependency-check-report

                    "${DC_HOME}/bin/dependency-check.sh" \
                      --project "voting-app-worker" \
                      --scan . \
                      --format HTML \
                      --format XML \
                      --out dependency-check-report \
                      --disableAssembly

                    echo "OWASP Dependency-Check completed"
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

                        docker tag "${IMAGE_NAME}:${IMAGE_TAG}" \
                          "${IMAGE}:${IMAGE_TAG}"

                        docker tag "${IMAGE_NAME}:latest" \
                          "${IMAGE}:latest"

                        docker push "${IMAGE}:${IMAGE_TAG}"
                        docker push "${IMAGE}:latest"

                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
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

                        kubectl -n voting-app set image \
                          deployment/worker \
                          worker="${IMAGE}:${IMAGE_TAG}"

                        kubectl -n voting-app rollout status \
                          deployment/worker \
                          --timeout=180s
                    '''
                }
            }
        }

        stage('Deployment Verification') {
            steps {
                sh '''
                    set -e

                    echo "===== WORKER DEPLOYMENT ====="
                    kubectl get deployment worker -n voting-app

                    echo
                    echo "===== WORKER PODS ====="
                    kubectl get pods \
                      -n voting-app \
                      -l app=worker \
                      -o wide

                    echo
                    echo "===== WORKER IMAGE ====="
                    kubectl get deployment worker \
                      -n voting-app \
                      -o jsonpath='{.spec.template.spec.containers[0].image}'

                    echo
                '''
            }
        }
    }

    post {
        always {
            echo "===== OWASP REPORT ====="

            archiveArtifacts(
                artifacts: 'dependency-check-report/*',
                allowEmptyArchive: true,
                fingerprint: true
            )

            echo "Worker Jenkins pipeline completed."
        }
    }
}
