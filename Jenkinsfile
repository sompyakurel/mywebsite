pipeline {
    agent any

    environment {
        AWS_REGION     = "us-east-1"
        AWS_ACCOUNT_ID = "588738575196"

        ECR_REPO_NAME  = "mywebsite"
        ECR_REGISTRY   = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        ECR_URI        = "${ECR_REGISTRY}/${ECR_REPO_NAME}"

        // Terraform-managed app EC2 (NOT Jenkins IP)
        EC2_HOST       = "18.204.28.177"
        EC2_USER       = "ubuntu"

        DOCKER_IMAGE_TAG = "build-${env.BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build & Push Image') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                                  credentialsId: 'aws-creds']]) {
                    sh """
                      echo 'Building Docker image...'
                      docker build --platform linux/amd64 -t ${ECR_REPO_NAME}:${DOCKER_IMAGE_TAG} .

                      echo 'Logging into ECR from Jenkins...'
                      aws ecr get-login-password --region ${AWS_REGION} \\
                        | docker login --username AWS --password-stdin ${ECR_REGISTRY}

                      echo 'Tagging and pushing image...'
                      docker tag ${ECR_REPO_NAME}:${DOCKER_IMAGE_TAG} ${ECR_URI}:${DOCKER_IMAGE_TAG}
                      docker push ${ECR_URI}:${DOCKER_IMAGE_TAG}
                    """
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                sshagent (credentials: ['ec2-ssh-key']) {
                    sh """
                      ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} '
                        echo "Logging into ECR on EC2..." &&
                        aws ecr get-login-password --region ${AWS_REGION} \\
                          | docker login --username AWS --password-stdin ${ECR_REGISTRY} &&

                        echo "Pulling new image..." &&
                        docker pull ${ECR_URI}:${DOCKER_IMAGE_TAG} &&

                        echo "Restarting container..." &&
                        docker rm -f mywebsite || true &&
                        docker run -d --name mywebsite -p 80:80 ${ECR_URI}:${DOCKER_IMAGE_TAG}
                      '
                    """
                }
            }
        }
    }
}
