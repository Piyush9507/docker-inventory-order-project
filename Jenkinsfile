pipeline {
    agent any

    environment {
        AWS_REGION          = 'us-west-2'
        AWS_ACCOUNT_ID      = '212773812862'
        ECR_REGISTRY        = '212773812862.dkr.ecr.us-west-2.amazonaws.com'

        INVENTORY_REPO      = 'inventory-service'
        ORDER_REPO          = 'order-service'

        IMAGE_TAG           = "${BUILD_NUMBER}"

        JUMP_HOST           = '54.189.187.92'
        SSH_USER            = 'ec2-user'
        SSH_CREDENTIALS     = 'target-server-ssh-key'
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/Piyush9507/docker-inventory-order-project.git'
            }
        }

        stage('Build Inventory Image') {
            steps {
                sh '''
                    docker build \
                        -f inventory_service/Dockerfile \
                        -t ${ECR_REGISTRY}/${INVENTORY_REPO}:${IMAGE_TAG} .
                '''
            }
        }

        stage('Build Order Image') {
            steps {
                sh '''
                    docker build \
                        -f order_service/Dockerfile \
                        -t ${ECR_REGISTRY}/${ORDER_REPO}:${IMAGE_TAG} .
                '''
            }
        }

        stage('ECR Login') {
            steps {
                sh '''
                    aws ecr get-login-password \
                        --region ${AWS_REGION} | \
                    docker login \
                        --username AWS \
                        --password-stdin ${ECR_REGISTRY}
                '''
            }
        }

        stage('Push Images to ECR') {
            steps {
                sh '''
                    docker push ${ECR_REGISTRY}/${INVENTORY_REPO}:${IMAGE_TAG}
                    docker push ${ECR_REGISTRY}/${ORDER_REPO}:${IMAGE_TAG}
                '''
            }
        }

        stage('Deploy - Pull Images on Jump Server') {
            steps {
                sshagent(credentials: [SSH_CREDENTIALS]) {
                    sh '''
                        ssh -o StrictHostKeyChecking=accept-new \
                            ${SSH_USER}@${JUMP_HOST} \
                            "aws ecr get-login-password --region ${AWS_REGION} | \
                            docker login --username AWS --password-stdin ${ECR_REGISTRY} && \
                            docker pull ${ECR_REGISTRY}/${INVENTORY_REPO}:${IMAGE_TAG} && \
                            docker pull ${ECR_REGISTRY}/${ORDER_REPO}:${IMAGE_TAG}"
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "SUCCESS: Both microservice images pushed to ECR and pulled on Jump Server."
            echo "Image tag: ${IMAGE_TAG}"
        }

        failure {
            echo "FAILED: Check the Jenkins console output for the failed stage."
        }

        always {
            echo "Pipeline execution completed. Build number: ${BUILD_NUMBER}"
        }
    }
}
