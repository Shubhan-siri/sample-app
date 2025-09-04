pipeline {
    agent any

    environment {
        AWS_DEFAULT_REGION = 'ap-south-1'
        ECR_REPO = '607458533394.dkr.ecr.ap-south-1.amazonaws.com/my-app'
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        EKS_CLUSTER = 'sample-eks-cluster'
    }

    stages {
        stage('Checkout Code') {
            steps {
                // Pull code from your GitHub repo
                git branch: 'main', url: 'https://github.com/Shubhan-siri/sample-app.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    // Build Docker image from Dockerfile
                    sh 'docker build -t my-app .'
                    // Tag the image with BUILD_NUMBER for versioning
                    sh "docker tag my-app:latest ${ECR_REPO}:${IMAGE_TAG}"
                }
            }
        }

        stage('Push to ECR') {
            steps {
                script {
                    // Login to ECR and push the image
                    sh "aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${ECR_REPO}"
                    sh "docker push ${ECR_REPO}:${IMAGE_TAG}"
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                script {
                    // Configure kubectl to connect to your EKS cluster
                    sh "aws eks --region ${AWS_DEFAULT_REGION} update-kubeconfig --name ${EKS_CLUSTER}"
                    // Update deployment with the new image
                    sh """
                    kubectl set image deployment/my-app my-app=${ECR_REPO}:${IMAGE_TAG} --record
                    kubectl rollout status deployment/my-app
                    """
                }
            }
        }
    }

    post {
        success {
            echo "✅ Deployment to EKS successful!"
        }
        failure {
            echo "❌ Deployment failed. Check Jenkins logs."
        }
    }
}
