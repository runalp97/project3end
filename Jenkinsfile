pipeline {
    agent any

    environment {
        SSH_HOST = 'ubuntu@ec2-65-0-85-76.ap-south-1.compute.amazonaws.com'
        ECR_REGISTRY = '122610480795.dkr.ecr.ap-south-1.amazonaws.com'
        IMAGE_NAME = 'my_appy'
    }

    parameters {
        booleanParam(name: 'DESTROY', defaultValue: false, description: 'Destroy deployment on EC2 after completion')
    }

    stages {
        stage('Checkout Source') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t myappimg:latest ."
                sh "docker tag myappimg:latest ${ECR_REGISTRY}/${IMAGE_NAME}:latest"
            }
        }

        stage('Push to ECR') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-ecr-creds']]) {
                    sh """
                        aws ecr get-login-password --region ap-south-1 | \
                        docker login --username AWS --password-stdin ${ECR_REGISTRY}
                        docker push ${ECR_REGISTRY}/${IMAGE_NAME}:latest
                    """
                }
            }
        }

        stage('Deploy on EC2') {
            steps {
                sshagent(['ec2-ssh-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${SSH_HOST} '
                            docker pull ${ECR_REGISTRY}/${IMAGE_NAME}:latest &&
                            docker stop myapp || true &&
                            docker rm myapp || true &&
                            docker run -d -p 80:80 --name myapp ${ECR_REGISTRY}/${IMAGE_NAME}:latest
                        '
                    """
                }
            }
        }

        stage('Destroy') {
            when {
                expression { return params.DESTROY == true }
            }
            steps {
                sshagent(['ec2-ssh-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${SSH_HOST} '
                            docker stop myapp || true &&
                            docker rm myapp || true &&
                            docker rmi ${ECR_REGISTRY}/${IMAGE_NAME}:latest || true
                        '
                    """
                }
            }
        }
    }

    post {
        success {
            echo "✅ Deployment succeeded!"
        }
        failure {
            echo "❌ Deployment failed!"
        }
    }
}

