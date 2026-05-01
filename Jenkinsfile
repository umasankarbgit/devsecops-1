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

    stages {

        stage('Checkout Source Code') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/umasankarbgit/devsecops-1.git'
            }
        }

        // ✅ 1. UNIT TESTING
        stage('Run Unit Tests') {
            steps {
                sh '''
                    echo "Running unit tests..."
                    pip install -r requirements.txt
                    pytest || exit 1
                '''
            }
        }

        // ✅ 2. CODE QUALITY + SECURITY SCAN
        stage('SonarQube Scan') {
            steps {
                withSonarQubeEnv('sonarqube-server') {
                    sh '''
                        sonar-scanner \
                        -Dsonar.projectKey=myapp \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=$SONAR_HOST_URL \
                        -Dsonar.login=$SONAR_AUTH_TOKEN
                    '''
                }
            }
        }

        // ✅ 3. QUALITY GATE (IMPORTANT)
        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        // ✅ 4. DEPENDENCY SCAN (Trivy FS)
        stage('Dependency Scan') {
            steps {
                sh '''
                    echo "Scanning dependencies..."
                    trivy fs --exit-code 1 --severity HIGH,CRITICAL .
                '''
            }
        }

        // ✅ 5. BUILD IMAGE
        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t $FULL_IMAGE .
                '''
            }
        }

        // ✅ 6. IMAGE SCAN (CRITICAL)
        stage('Trivy Image Scan') {
            steps {
                sh '''
                    trivy image --exit-code 1 --severity HIGH,CRITICAL $FULL_IMAGE
                '''
            }
        }

        // ✅ 7. LOGIN
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
                    docker push $FULL_IMAGE
                '''
            }
        }

        // ✅ 9. DEPLOY ONLY IF SAFE
        stage('Deploy to EKS') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds'
                ]]) {
                    sh '''
                        aws eks update-kubeconfig \
                            --region ap-south-1 \
                            --name test-eks

                        sed -i "s|image:.*|image: $FULL_IMAGE|g" K8s/deployment.yaml

                        kubectl apply -f K8s/deployment.yaml
                        kubectl apply -f K8s/service.yaml

                        kubectl rollout status deployment/dockerhub-sample-app
                    '''
                }
            }
        }
    }
}
