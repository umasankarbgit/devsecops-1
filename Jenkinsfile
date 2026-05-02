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

        // ✅ 1. UNIT TESTING (FIXED)
        stage('Run Unit Tests') {
            steps {
                sh '''
                    set -e
                    echo "Running unit tests..."

                    # Install deps in user space (jenkins-safe)
                    pip3 install --user -r requirements.txt

                    # Run pytest without PATH issue
                    python3 -m pytest
                '''
            }
        }

        // ✅ 2. SONARQUBE SCAN (FIXED)
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

        // ✅ 3. QUALITY GATE
        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        // ✅ 4. FILE SYSTEM SCAN (TRIVY)
        stage('Dependency Scan') {
            steps {
                sh '''
                    set -e
                    echo "Scanning dependencies..."
                    trivy fs --exit-code 1 --severity HIGH,CRITICAL .
                '''
            }
        }

        // ✅ 5. BUILD IMAGE
        stage('Build Docker Image') {
            steps {
                sh '''
                    set -e
                    docker build -t $FULL_IMAGE .
                '''
            }
        }

        // ✅ 6. IMAGE SCAN
        stage('Trivy Image Scan') {
            steps {
                sh '''
                    set -e
                    trivy image --exit-code 1 --severity HIGH,CRITICAL $FULL_IMAGE
                '''
            }
        }

        // ✅ 7. LOGIN DOCKER HUB
        stage('Login to Docker Hub') {
            steps {
                sh '''
                    echo "$DOCKERHUB_CREDS_PSW" | \
                    docker login -u "$DOCKERHUB_CREDS_USR" --password-stdin
                '''
            }
        }

        // ✅ 8. PUSH IMAGE
        stage('Push Image to Docker Hub') {
            steps {
                sh '''
                    set -e
                    docker push $FULL_IMAGE
                '''
            }
        }

        // ✅ 9. DEPLOY TO EKS
        stage('Deploy to EKS') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds'
                ]]) {
                    sh '''
                        set -e
                        echo "Updating kubeconfig..."
                        aws eks update-kubeconfig \
                            --region $AWS_REGION \
                            --name $EKS_CLUSTER_NAME

                        echo "Updating image..."
                        sed -i "s|image:.*|image: $FULL_IMAGE|g" K8s/deployment.yaml

                        echo "Deploying..."
                        kubectl apply -f K8s/deployment.yaml
                        kubectl apply -f K8s/service.yaml

                        echo "Checking rollout..."
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
