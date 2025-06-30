pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-south-1'
        ECR_REPO = 'my_appy'
        ECR_REGISTRY = '122610480795.dkr.ecr.ap-south-1.amazonaws.com'
        IMAGE_NAME = 'myappimg'
        IMAGE_TAG = 'latest'
        REMOTE_USER = 'ubuntu'
        REMOTE_HOST = 'ec2-65-0-85-76.ap-south-1.compute.amazonaws.com'
        SSH_KEY = credentials('ec2-ssh-key')
    }

    stages {
        stage('Checkout Source') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                docker build -t $IMAGE_NAME:$IMAGE_TAG .
                docker tag $IMAGE_NAME:$IMAGE_TAG $ECR_REGISTRY/$ECR_REPO:$IMAGE_TAG
                """
            }
        }

        stage('Push to ECR') {
            steps {
                sh """
                aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY
                docker push $ECR_REGISTRY/$ECR_REPO:$IMAGE_TAG
                """
            }
        }

        stage('Deploy on EC2') {
            steps {
                sshagent(['ec2-ssh-key']) {
                    sh """
                    ssh -o StrictHostKeyChecking=no $REMOTE_USER@$REMOTE_HOST << 'EOF'
                      sudo apt update -y
                      sudo apt install -y awscli docker.io
                      sudo systemctl start docker
                      sudo usermod -aG docker \$USER

                      aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY
                      docker pull $ECR_REGISTRY/$ECR_REPO:$IMAGE_TAG
                      docker stop myapp || true
                      docker rm myapp || true
                      docker run -d --name myapp -p 80:80 $ECR_REGISTRY/$ECR_REPO:$IMAGE_TAG
                    EOF
                    """
                }
            }
        }
    }

    post {
        success {
            echo '✅ Deployment successful!'
        }
        failure {
            echo '❌ Deployment failed!'
        }
    }
}

