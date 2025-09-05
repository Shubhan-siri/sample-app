pipeline {
    agent any

    environment {
        AWS_DEFAULT_REGION = 'ap-south-1'
        ECR_REPO = '607458533394.dkr.ecr.ap-south-1.amazonaws.com/my-app'
        IMAGE_TAG = "${BUILD_NUMBER}"
        K8S_CLUSTER = 'sample-eks-cluster'
    }

    stages {
        stage('Checkout') {
            steps {
                deleteDir()   // Clean workspace before checkout
                withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
                    sh '''
                        git clone https://$GITHUB_TOKEN@github.com/Shubhan-siri/sample-app.git .
                    '''
                }
            }
        }

        stage('Debug Workspace') {
            steps {
                sh 'echo "📂 Current Workspace:"'
                sh 'pwd'
                sh 'ls -la'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                    docker build -t ${ECR_REPO}:${IMAGE_TAG} .
                """
            }
        }

        stage('Login to ECR') {
            steps {
                sh """
                    aws ecr get-login-password --region $AWS_DEFAULT_REGION | \
                    docker login --username AWS --password-stdin $ECR_REPO
                """
            }
        }

        stage('Push to ECR') {
            steps {
                sh """
                    docker push ${ECR_REPO}:${IMAGE_TAG}
                """
            }
        }

        stage('Update Kubernetes Manifest') {
            steps {
                sh """
                    sed -i '' "s|image:.*|image: ${ECR_REPO}:${IMAGE_TAG}|" k8s/deployment.yaml
                """
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh """
                    aws eks update-kubeconfig --region $AWS_DEFAULT_REGION --name $K8S_CLUSTER
                    kubectl apply -f k8s/deployment.yaml
                    kubectl apply -f k8s/service.yaml
                """
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline executed successfully! Deployed ${IMAGE_TAG} to ${K8S_CLUSTER}"
        }
        failure {
            echo "❌ Pipeline failed. Check the Jenkins logs for details."
        }
    }
}
