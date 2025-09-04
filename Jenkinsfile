pipeline {
    agent any

    environment {
        AWS_DEFAULT_REGION = 'ap-south-1'
        ECR_REPO = '607458533394.dkr.ecr.ap-south-1.amazonaws.com/my-app'
        IMAGE_TAG = "${BUILD_NUMBER}"
        K8S_CLUSTER = 'sample-eks-cluster'
        VPC_ID = 'vpc-0b339a0a078bf77d6'
        SUBNET_IDS = 'subnet-062a3fed3a8b1d172,subnet-06b238ff5123bcea5'
    }

    stages {
        stage('Checkout') {
            steps {
                // Clean workspace to avoid "not in a git directory" error
                sh 'rm -rf *' 
                // Clone GitHub repo
                git branch: 'main', url: 'https://github.com/Shubhan-siri/sample-app.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${ECR_REPO}:${IMAGE_TAG} ."
            }
        }

        stage('Login to ECR') {
            steps {
                sh '''
                    aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $ECR_REPO
                '''
            }
        }

        stage('Push to ECR') {
            steps {
                sh "docker push ${ECR_REPO}:${IMAGE_TAG}"
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    sh """
                        aws eks update-kubeconfig --region $AWS_DEFAULT_REGION --name $K8S_CLUSTER
                        kubectl apply -f k8s/service.yaml
                        kubectl apply -f k8s/deployment.yaml
                        kubectl set image deployment/$(kubectl get deployment -o jsonpath='{.items[0].metadata.name}') \
                        $(kubectl get deployment -o jsonpath='{.items[0].spec.template.spec.containers[0].name}')=${ECR_REPO}:${IMAGE_TAG} --record
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline executed successfully!"
        }
        failure {
            echo "Pipeline failed. Check the logs."
        }
    }
}
