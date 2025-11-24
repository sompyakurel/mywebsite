pipeline {
    agent any

    environment {
        AWS_REGION     = "us-east-1"
        AWS_ACCOUNT_ID = "588738575196"

        ECR_REPO_NAME  = "mywebsite"
        ECR_URI        = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}"

        EC2_HOST       = "18.204.28.177"
        EC2_USER       = "ubuntu"

        DOCKER_IMAGE_TAG = "build-${env.BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/sompyakurel/mywebsite.git',
                    credentialsId: 'github-token'
            }
        }

        stage('Build & Push Image') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                                  credentialsId: 'aws-creds']]) {
                    sh """
                      docker build --platform linux/amd64 -t ${ECR_REPO_NAME}:${DOCKER_IMAGE_TAG} .

                      aws ecr get-login-password --region ${AWS_REGION} \\
                        | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

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
                        aws ecr get-login-password --region ${AWS_REGION} \\
                          | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com &&

                        docker pull ${ECR_URI}:${DOCKER_IMAGE_TAG} &&

                        docker rm -f mywebsite || true &&

                        docker run -d --name mywebsite -p 80:80 ${ECR_URI}:${DOCKER_IMAGE_TAG}
                      '
                    """
                }
            }
        }
    }
}
