pipeline {
    agent any

    environment {
        AWS_REGION = "ap-south-1"
        EKS_CLUSTER_NAME = "test-eks"

        // DockerHub credentials (already correct)
        DOCKERHUB_CREDS = credentials('dockerhub-creds')

        // Image details
        IMAGE_NAME = "${DOCKERHUB_CREDS_USR}/myapp"
        IMAGE_TAG  = "${BUILD_NUMBER}"
        FULL_IMAGE = "${IMAGE_NAME}:${IMAGE_TAG}"
    }

    stages {

        stage('Checkout Source Code') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/umasankarbgit/project-6.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    echo "Building Docker image..."
                    docker build -t $FULL_IMAGE .
                '''
            }
        }

        stage('Login to Docker Hub') {
            steps {
                sh '''
                    echo "$DOCKERHUB_CREDS_PSW" | \
                    docker login -u "$DOCKERHUB_CREDS_USR" --password-stdin
                '''
            }
        }

        stage('Push Image to Docker Hub') {
            steps {
                sh '''
                    echo "Pushing image to Docker Hub..."
                    docker push $FULL_IMAGE
                '''
            }
        }

        stage('Update Kubeconfig') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds'
                ]]) {
                    sh '''
                        echo "Updating kubeconfig..."
                        aws eks update-kubeconfig \
                            --region ap-south-1 \
                            --name test-eks
                    '''
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds'
                ]]) {
                    sh '''
                        echo "Updating image in deployment YAML..."
                        sed -i "s|image:.*|image: $FULL_IMAGE|g" K8s/deployment.yaml

                        echo "Deploying to EKS..."
                        kubectl apply -f K8s/deployment.yaml
                        kubectl apply -f K8s/service.yaml

                        echo "Checking rollout status..."
                        kubectl rollout status deployment/dockerhub-sample-app
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "✅ CI/CD pipeline completed successfully. App deployed to EKS!"
        }
        failure {
            echo "❌ Pipeline failed. Please check logs."
        }
    }
}
