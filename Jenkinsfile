
pipeline {
    agent any

    environment {
        IMAGE_NAME = 'manoj1812/product-service'
        KUBECONFIG = '/var/lib/jenkins/.kube/config'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build and Test') {
            steps {
                dir('Product-service') {
                    sh 'chmod +x mvnw'
                    sh './mvnw -B clean test package'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build \
                      -t ${IMAGE_NAME}:${BUILD_NUMBER} \
                      -t ${IMAGE_NAME}:latest \
                      Product-service
                '''
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_TOKEN'
                )]) {
                    sh '''
                        set +x
                        echo "$DOCKER_TOKEN" | docker login \
                          -u "$DOCKER_USER" --password-stdin

                        docker push ${IMAGE_NAME}:${BUILD_NUMBER}
                        docker push ${IMAGE_NAME}:latest

                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                sh '''
                    if ! kubectl set image deployment/product-service \
                      product-service=${IMAGE_NAME}:${BUILD_NUMBER}; then
                        echo "Image update failed; deployment unchanged."
                        exit 1
                    fi

                    if kubectl rollout status deployment/product-service \
                      --timeout=180s; then
                        echo "Deployment successful."
                    else
                        echo "Deployment failed. Starting rollback..."

                        if kubectl rollout undo deployment/product-service; then
                            if kubectl rollout status deployment/product-service \
                              --timeout=180s; then
                                echo "Rollback successful; previous version restored."
                            else
                                echo "ERROR: Rollback did not become healthy."
                            fi
                        else
                            echo "ERROR: Rollback command failed."
                        fi

                        exit 1
                    fi
                '''
            }
        }
    }

    post {
        success {
            echo 'CI/CD pipeline completed successfully.'
        }
        failure {
            echo 'Pipeline failed. Check deployment and rollback logs.'
        }
    }
}
