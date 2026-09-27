pipeline {
    agent {
        label 'blogapp-agent'
    }

    environment {
        AWS_REGION      = 'ap-south-1'
        AWS_ACCOUNT_ID  = '354596761006'

        ECR_REGISTRY    = '354596761006.dkr.ecr.ap-south-1.amazonaws.com'
        ECR_FRONTEND    = 'blogapp-frontend'
        ECR_BACKEND     = 'blogapp-backend'

        EKS_CLUSTER     = 'blogapp-dev-eks'
        K8S_NAMESPACE   = 'blogapp'

        SONAR_HOST_URL     = 'http://localhost:9000'
        SONAR_PROJECT_KEY  = 'blogapp'
        SONAR_PROJECT_NAME = 'BlogReact'
    }

    options {
        timestamps()
        disableConcurrentBuilds()
        skipDefaultCheckout(true)
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                dir('frontend') {
                    sh 'npm ci'
                }

                dir('backend') {
                    sh 'npm ci'
                }
            }
        }

        stage('Build & Test') {
            parallel {
                stage('Frontend Build') {
                    steps {
                        dir('frontend') {
                            sh 'npm run build'
                        }
                    }
                }

                stage('Backend Test') {
                    steps {
                        dir('backend') {
                            sh 'npm test'
                        }
                    }
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withCredentials([
                    string(
                        credentialsId: 'sonar-token',
                        variable: 'SONAR_TOKEN'
                    )
                ]) {
                    sh '''
                        docker run --rm \
                          --network host \
                          -e SONAR_HOST_URL="$SONAR_HOST_URL" \
                          -e SONAR_TOKEN="$SONAR_TOKEN" \
                          -v "$WORKSPACE:/usr/src" \
                          sonarsource/sonar-scanner-cli:latest \
                          -Dsonar.projectKey="$SONAR_PROJECT_KEY" \
                          -Dsonar.projectName="$SONAR_PROJECT_NAME" \
                          -Dsonar.sources=backend/src,frontend/src \
                          -Dsonar.exclusions="**/node_modules/**,**/dist/**,**/coverage/**" \
                          -Dsonar.qualitygate.wait=true \
                          -Dsonar.qualitygate.timeout=300
                    '''
                }
            }
        }

        stage('Prepare Image Tags') {
            steps {
                script {
                    env.GIT_SHA = sh(
                        script: 'git rev-parse --short=7 HEAD',
                        returnStdout: true
                    ).trim()

                    env.FRONTEND_IMAGE =
                        "${ECR_REGISTRY}/${ECR_FRONTEND}:sha-${GIT_SHA}"

                    env.BACKEND_IMAGE =
                        "${ECR_REGISTRY}/${ECR_BACKEND}:sha-${GIT_SHA}"

                    echo "Git SHA: ${GIT_SHA}"
                    echo "Frontend image: ${FRONTEND_IMAGE}"
                    echo "Backend image: ${BACKEND_IMAGE}"
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    docker build \
                      -t "$FRONTEND_IMAGE" \
                      frontend

                    docker build \
                      -t "$BACKEND_IMAGE" \
                      backend
                '''
            }
        }

        stage('Trivy Security Scan') {
            steps {
                sh '''
                    echo "Scanning frontend image..."

                    trivy image \
                      --severity HIGH,CRITICAL \
                      --ignore-unfixed \
                      --exit-code 0 \
                      "$FRONTEND_IMAGE"

                    echo "Scanning backend image..."

                    trivy image \
                      --severity HIGH,CRITICAL \
                      --ignore-unfixed \
                      --exit-code 0 \
                      "$BACKEND_IMAGE"
                '''
            }
        }

        stage('Push Images to ECR') {
            steps {
                sh '''
                    aws ecr get-login-password \
                      --region "$AWS_REGION" |
                    docker login \
                      --username AWS \
                      --password-stdin "$ECR_REGISTRY"

                    docker push "$FRONTEND_IMAGE"
                    docker push "$BACKEND_IMAGE"
                '''
            }
        }

        stage('Deploy to EKS') {
            steps {
                sh '''
                    aws eks update-kubeconfig \
                      --region "$AWS_REGION" \
                      --name "$EKS_CLUSTER"

                    kubectl -n "$K8S_NAMESPACE" \
                      set image deployment/blogapp-frontend \
                      frontend="$FRONTEND_IMAGE"

                    kubectl -n "$K8S_NAMESPACE" \
                      set image deployment/blogapp-backend \
                      backend="$BACKEND_IMAGE"

                    kubectl -n "$K8S_NAMESPACE" \
                      rollout status deployment/blogapp-frontend \
                      --timeout=5m

                    kubectl -n "$K8S_NAMESPACE" \
                      rollout status deployment/blogapp-backend \
                      --timeout=5m
                '''
            }
        }

        stage('Deployment Summary') {
            steps {
                sh '''
                    echo "=========================================="
                    echo "BLOGAPP DEPLOYMENT COMPLETED"
                    echo "=========================================="

                    kubectl -n "$K8S_NAMESPACE" get pods
                    kubectl -n "$K8S_NAMESPACE" get deployments
                    kubectl -n "$K8S_NAMESPACE" get ingress
                '''
            }
        }
    }

    post {
        always {
            sh 'docker image prune -f || true'
        }

        success {
            echo 'BlogApp CI/CD pipeline completed successfully.'
        }

        failure {
            echo 'BlogApp CI/CD pipeline failed.'
        }
    }
}
