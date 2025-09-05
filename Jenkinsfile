pipeline {
    agent any

    environment {
        AWS_DEFAULT_REGION = 'ap-south-1'
        ECR_REPO = '607458533394.dkr.ecr.ap-south-1.amazonaws.com/my-app'
        IMAGE_TAG = "${BUILD_NUMBER}"
        EKS_CLUSTER = 'sample-eks-cluster'
        NODE_TYPE = 't3.small'               // Node type t3.small
        NODEGROUP_NAME = 'standard-workers'
        GITHUB_TOKEN = credentials('github-token')
        VPC_PRIVATE_SUBNETS = 'subnet-062a3fed3a8b1d172,subnet-06b238ff5123bcea5'
        VPC_PUBLIC_SUBNETS = 'subnet-062a3fed3a8b1d172,subnet-06b238ff5123bcea5'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/Shubhan-siri/sample-app.git', credentialsId: 'github-token'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                docker build -t $ECR_REPO:$IMAGE_TAG .
                '''
            }
        }

        stage('Login to ECR') {
            steps {
                sh '''
                aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $ECR_REPO
                '''
            }
        }

        stage('Push Docker Image') {
            steps {
                sh '''
                docker push $ECR_REPO:$IMAGE_TAG
                '''
            }
        }

        stage('Deploy to EKS') {
            steps {
                script {
                    sh '''
                    # Install eksctl if missing
                    if ! command -v eksctl >/dev/null 2>&1; then
                      echo "Installing eksctl..."
                      curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
                      mv /tmp/eksctl /usr/local/bin
                    fi

                    # Create cluster if not exists
                    if ! eksctl get cluster --name $EKS_CLUSTER --region $AWS_DEFAULT_REGION >/dev/null 2>&1; then
                        echo "Creating EKS Cluster with t3.small nodes..."
                        eksctl create cluster \
                        --name $EKS_CLUSTER \
                        --region $AWS_DEFAULT_REGION \
                        --version 1.30 \
                        --vpc-private-subnets=$VPC_PRIVATE_SUBNETS \
                        --vpc-public-subnets=$VPC_PUBLIC_SUBNETS \
                        --nodegroup-name $NODEGROUP_NAME \
                        --node-type $NODE_TYPE \
                        --nodes 2 \
                        --nodes-min 1 \
                        --nodes-max 3 \
                        --ssh-access \
                        --ssh-public-key shubhan-key
                    else
                        echo "Cluster $EKS_CLUSTER already exists. Skipping creation."
                    fi

                    # Update kubeconfig
                    aws eks update-kubeconfig --region $AWS_DEFAULT_REGION --name $EKS_CLUSTER

                    # Deploy or update application
                    if kubectl get deployment my-deployment >/dev/null 2>&1; then
                        echo "Updating existing deployment..."
                        kubectl set image deployment/my-deployment my-container=$ECR_REPO:$IMAGE_TAG
                        kubectl rollout status deployment/my-deployment
                    else
                        echo "Creating new deployment..."
                        kubectl create deployment my-deployment --image=$ECR_REPO:$IMAGE_TAG
                        kubectl expose deployment my-deployment --type=LoadBalancer --port=80 --target-port=80
                    fi
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "Deployment Successful!"
        }
        failure {
            echo "Deployment Failed!"
        }
    }
}
