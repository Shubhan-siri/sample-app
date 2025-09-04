pipeline {
    agent any

    environment {
        AWS_DEFAULT_REGION = 'ap-south-1'
        ECR_REPO = '607458533394.dkr.ecr.ap-south-1.amazonaws.com/my-app'
        IMAGE_TAG = "${BUILD_NUMBER}"   // Har run me unique image tag
        EKS_CLUSTER = 'my-eks-cluster'
        GIT_REPO = 'https://github.com/Shubhan-siri/sample-app.git'
        GIT_BRANCH = 'main'
    }

    stages {
        stage('Clean Workspace') {
            steps {
                echo "🧹 Cleaning workspace..."
                cleanWs()
            }
        }

        stage('Checkout Code') {
            steps {
                echo "🔄 Pulling latest code from GitHub..."
                git branch: "${GIT_BRANCH}", url: "${GIT_REPO}"
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    echo "🔨 Building Docker Image..."
                    dockerImage = docker.build("my-app:latest")
                }
            }
        }

        stage('Push to ECR') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials']]) {
                    sh '''
                        echo "🔐 Logging in to Amazon ECR..."
                        aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $ECR_REPO

                        echo "🏷️ Tagging Docker image..."
                        docker tag my-app:latest $ECR_REPO:$IMAGE_TAG

                        echo "📤 Pushing Docker image to ECR..."
                        docker push $ECR_REPO:$IMAGE_TAG
                    '''
                }
            }
        }

        stage('Ensure EKS Cluster') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials']]) {
                    sh '''
                        echo "🔎 Checking if EKS cluster exists..."
                        if ! aws eks describe-cluster --name $EKS_CLUSTER --region $AWS_DEFAULT_REGION >/dev/null 2>&1; then
                            echo "🚀 Cluster not found. Creating..."

                            cat <<EOF > cluster-config.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: $EKS_CLUSTER
  region: $AWS_DEFAULT_REGION

vpc:
  id: "vpc-0b339a0a078bf77d6"
  subnets:
    public:
      ap-south-1a:
        id: "subnet-062a3fed3a8b1d172"
      ap-south-1b:
        id: "subnet-06b238ff5123bcea5"

nodeGroups:
  - name: standard-workers
    instanceType: t3.small
    desiredCapacity: 2
    ssh:
      allow: true
      publicKeyName: shubhan-key
EOF

                            eksctl create cluster -f cluster-config.yaml
                        else
                            echo "✅ Cluster exists. Skipping creation."
                        fi
                    '''
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials']]) {
                    sh '''
                        echo "⚙️ Updating kubeconfig..."
                        aws eks --region $AWS_DEFAULT_REGION update-kubeconfig --name $EKS_CLUSTER

                        echo "🚢 Deploying Kubernetes manifests..."
                        kubectl apply -f k8s/
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "✅ Docker image pushed and deployed to EKS successfully!"
        }
        failure {
            echo "❌ Pipeline failed. Check logs for details."
        }
    }
}
