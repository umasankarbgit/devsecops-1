pipeline {
    agent any

    environment {
        AWS_REGION = "ap-south-1"
        EKS_CLUSTER_NAME = "test-eks"

        DOCKERHUB_CREDS = credentials('dockerhub-creds')

        IMAGE_NAME = "${DOCKERHUB_CREDS_USR}/myapp"
        IMAGE_TAG  = "${BUILD_NUMBER}-${GIT_COMMIT[0..6]}"
        FULL_IMAGE = "${IMAGE_NAME}:${IMAGE_TAG}"
    }

    options {
        timestamps()
    }

    stages {

        stage('Checkout Source Code') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/umasankarbgit/devsecops-1.git'
            }
        }

               // =========================
        // SONARQUBE SCAN
        // =========================
        stage('SonarQube Scan') {
            steps {
                withSonarQubeEnv('sonarqube-server') {
                    sh '''
                        set -e
                        sonar-scanner \
                        -Dsonar.projectKey=myapp \
                        -Dsonar.sources=.
                    '''
                }
            }
        }

        // =========================
        // QUALITY GATE
        // =========================
        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        // =========================
        // TRIVY FILE SCAN
        // =========================
        stage('Dependency Scan') {
            steps {
                sh '''
                    set -e
                    trivy fs --exit-code 1 --severity HIGH,CRITICAL .
                '''
            }
        }

        // =========================
        // BUILD IMAGE
        // =========================
        stage('Build Docker Image') {
            steps {
                sh '''
                    set -e
                    docker build -t $FULL_IMAGE .
                '''
            }
        }

        // =========================
        // IMAGE SCAN
        // =========================
        stage('Trivy Image Scan') {
            steps {
                sh '''
                    set -e
                    trivy image --exit-code 1 --severity HIGH,CRITICAL $FULL_IMAGE
                '''
            }
        }

        // =========================
        // DOCKER LOGIN
        // =========================
        stage('Login to Docker Hub') {
            steps {
                sh '''
                    echo "$DOCKERHUB_CREDS_PSW" | \
                    docker login -u "$DOCKERHUB_CREDS_USR" --password-stdin
                '''
            }
        }

        // =========================
        // PUSH IMAGE
        // =========================
        stage('Push Image to Docker Hub') {
            steps {
                sh '''
                    set -e
                    docker push $FULL_IMAGE
                '''
            }
        }

        // =========================
        // DEPLOY TO EKS
        // =========================
        stage('Deploy to EKS') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds'
                ]]) {
                    sh '''
                        set -e

                        aws eks update-kubeconfig \
                            --region $AWS_REGION \
                            --name $EKS_CLUSTER_NAME

                        sed -i "s|image:.*|image: $FULL_IMAGE|g" K8s/deployment.yaml

                        kubectl apply -f K8s/deployment.yaml
                        kubectl apply -f K8s/service.yaml

                        kubectl rollout status deployment/dockerhub-sample-app
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline completed successfully!"
        }
        failure {
            echo "❌ Pipeline failed. Check logs."
        }
    }
}
