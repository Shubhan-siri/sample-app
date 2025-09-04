pipeline {
    agent any

    environment {
        AWS_DEFAULT_REGION = 'ap-south-1'
        ECR_REPO = '607458533394.dkr.ecr.ap-south-1.amazonaws.com/my-app'
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        EKS_CLUSTER = 'sample-eks-cluster'
        EKS_ROLE_ARN = 'arn:aws:iam::607458533394:role/<Your-EKS-Role>'
        SUBNET_IDS = '<subnet-1>,<subnet-2>'
        SECURITY_GROUP_IDS = '<sg-id>'
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/Shubhan-siri/sample-app.git'
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

        stage('Create EKS Cluster (if not exists)') {
            steps {
                script {
                    def clusters = sh(script: "aws eks list-clusters --region ${AWS_DEFAULT_REGION} --query 'clusters' --output text", returnStdout: true).trim()
                    if (!clusters.contains("${EKS_CLUSTER}")) {
                        echo "Creating EKS cluster ${EKS_CLUSTER}..."
                        sh """
                        aws eks create-cluster \
                            --name ${EKS_CLUSTER} \
                            --region ${AWS_DEFAULT_REGION} \
                            --kubernetes-version 1.30 \
                            --role-arn ${EKS_ROLE_ARN} \
                            --resources-vpc-config subnetIds=${SUBNET_IDS},securityGroupIds=${SECURITY_GROUP_IDS}
                        """
                        echo "Waiting for EKS cluster to become ACTIVE..."
                        sh "aws eks wait cluster-active --name ${EKS_CLUSTER} --region ${AWS_DEFAULT_REGION}"
                        echo "EKS cluster is ready!"
                    } else {
                        echo "EKS cluster ${EKS_CLUSTER} already exists."
                    }
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                script {
                    sh "aws eks --region ${AWS_DEFAULT_REGION} update-kubeconfig --name ${EKS_CLUSTER}"
                    sh """
                    kubectl set image deployment/my-app my-app=${ECR_REPO}:${IMAGE_TAG} --record || \
                    kubectl create deployment my-app --image=${ECR_REPO}:${IMAGE_TAG}
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
