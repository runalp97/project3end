pipeline {
    agent any

    environment {
        IMAGE_NAME = "my_appy"
        ECR_REGISTRY = "122610480795.dkr.ecr.ap-south-1.amazonaws.com"
        REGION = "ap-south-1"
        SSH_HOST = "ubuntu@ec2-65-0-85-76.ap-south-1.compute.amazonaws.com"
    }

    stages {
        stage('Checkout Source') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t myappimg:latest .'
                sh 'docker tag myappimg:latest $ECR_REGISTRY/$IMAGE_NAME:latest'
            }
        }

        stage('Push to ECR') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY',
                    credentialsId: 'aws-ecr-creds'  // 🔁 replace with your Jenkins AWS credentials ID
                ]]) {
                    sh 'aws ecr get-login-password --region $REGION | docker login --username AWS --password-stdin $ECR_REGISTRY'
                    sh 'docker push $ECR_REGISTRY/$IMAGE_NAME:latest'
                }
            }
        }

        stage('Deploy on EC2') {
            steps {
                sshagent(['ec2-ssh-key']) {  // 🔁 replace with your SSH key ID stored in Jenkins
                    sh """
                    ssh -o StrictHostKeyChecking=no $SSH_HOST << EOF
                        docker pull $ECR_REGISTRY/$IMAGE_NAME:latest
                        docker stop myapp || true
                        docker rm myapp || true
                        docker run -d --name myapp -p 80:80 $ECR_REGISTRY/$IMAGE_NAME:latest
                    EOF
                    """
                }
            }
        }
    }

    post {
        failure {
            echo '❌ Deployment failed!'
        }
        success {
            echo '✅ Deployment completed successfully!'
        }
    }
}

