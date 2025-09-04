pipeline {
    agent any

    environment {
        AWS_DEFAULT_REGION = 'ap-south-1'
        ECR_REPO = '607458533394.dkr.ecr.ap-south-1.amazonaws.com/my-app'
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        EKS_CLUSTER = 'sample-eks-cluster'
        VPC_ID = 'vpc-0b339a0a078bf77d6'
        SUBNETS = 'subnet-062a3fed3a8b1d172,subnet-06b238ff5123bcea5'
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/Shubhan-siri/sample-app.git'
            }
        }

        stage('Create EKS Cluster (if not exists)') {
            steps {
                script {
                    def clusterExists = sh(script: "aws eks describe-cluster --name ${EKS_CLUSTER} --region ${AWS_DEFAULT_REGION} >/dev/null 2>&1 && echo true || echo false", returnStdout: true).trim()
                    if (clusterExists == 'false') {
                        echo "Cluster does not exist. Creating..."
                        sh """
                        eksctl create cluster \\
                            --name ${EKS_CLUSTER} \\
                            --region ${AWS_DEFAULT_REGION} \\
                            --version 1.30 \\
                            --vpc-private-subnets=${SUBNETS} \\
                            --vpc-public-subnets=${SUBNETS} \\
                            --nodegroup-name standard-workers \\
                            --node-type t3.small \\
                            --nodes 2 \\
                            --nodes-min 1 \\
                            --nodes-max 3 \\
                            --ssh-access \\
                            --ssh-public-key shubhan-key
                        """
                    } else {
                        echo "Cluster already exists. Skipping creation."
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh 'docker build -t my-app .'
                    sh "docker tag my-app:latest ${ECR_REPO}:${IMAGE_TAG}"
                }
            }
        }

        stage('Push to ECR') {
            steps {
                script {
                    sh "aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${ECR_REPO}"
                    sh "docker push ${ECR_REPO}:${IMAGE_TAG}"
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                script {
                    sh "aws eks --region ${AWS_DEFAULT_REGION} update-kubeconfig --name ${EKS_CLUSTER}"
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
