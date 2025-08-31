pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-south-1'
        ECR_REPO  = '607458533394.dkr.ecr.ap-south-1.amazonaws.com/my-app'
        IMAGE_TAG = "${env.BUILD_NUMBER}"
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/<YOUR-GITHUB-REPO>.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t my-app:latest .'
                sh "docker tag my-app:latest ${ECR_REPO}:${IMAGE_TAG}"
            }
        }

        stage('Push to ECR') {
            steps {
                sh """
                aws ecr get-login-password --region ${AWS_REGION} | \
                docker login --username AWS --password-stdin ${ECR_REPO}
                docker push ${ECR_REPO}:${IMAGE_TAG}
                docker push ${ECR_REPO}:latest
                """
            }
        }

        stage('Deploy to EKS') {
            steps {
                sh """
                aws eks --region ${AWS_REGION} update-kubeconfig --name my-eks-cluster
                kubectl set image deployment/my-app my-app=${ECR_REPO}:${IMAGE_TAG}
                kubectl rollout status deployment/my-app
                """
            }
        }
    }

    post {
        success {
            echo "App deployed successfully! 🎉"
        }
        failure {
            echo "Deployment failed 😢"
        }
    }
}

