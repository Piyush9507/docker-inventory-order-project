pipeline {
    agent any

    environment {
        AWS_REGION       = 'us-west-2'
        AWS_ACCOUNT_ID   = '212773812862'
        ECR_REGISTRY     = '212773812862.dkr.ecr.us-west-2.amazonaws.com'

        INVENTORY_REPO   = 'inventory-service'
        ORDER_REPO       = 'order-service'

        IMAGE_TAG        = "${BUILD_NUMBER}"

        JUMP_HOST        = '54.189.187.92'
        SSH_USER         = 'ec2-user'
        SSH_CREDENTIALS  = 'target-server-ssh-key'

        DOCKER_NETWORK   = 'inventory-order-net'
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
                    aws ecr get-login-password --region ${AWS_REGION} | \
                    docker login --username AWS --password-stdin ${ECR_REGISTRY}
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

        stage('Deploy to Jump Server') {
            steps {
                sshagent(credentials: [SSH_CREDENTIALS]) {
                    sh '''
                        set -e

                        ssh -o StrictHostKeyChecking=accept-new \
                            ${SSH_USER}@${JUMP_HOST} "

                            set -e

                            echo 'Logging in to ECR...'

                            aws ecr get-login-password --region ${AWS_REGION} | \
                            docker login --username AWS --password-stdin ${ECR_REGISTRY}

                            echo 'Pulling latest Inventory image...'

                            docker pull ${ECR_REGISTRY}/${INVENTORY_REPO}:${IMAGE_TAG}

                            echo 'Pulling latest Order image...'

                            docker pull ${ECR_REGISTRY}/${ORDER_REPO}:${IMAGE_TAG}

                            echo 'Creating Docker network if it does not exist...'

                            docker network inspect ${DOCKER_NETWORK} >/dev/null 2>&1 || \
                            docker network create ${DOCKER_NETWORK}

                            echo 'Removing old containers...'

                            docker rm -f inventory-service order-service >/dev/null 2>&1 || true

                            echo 'Starting Inventory service...'

                            docker run -d \
                                --name inventory-service \
                                --restart unless-stopped \
                                --network ${DOCKER_NETWORK} \
                                -p 8082:8082 \
                                ${ECR_REGISTRY}/${INVENTORY_REPO}:${IMAGE_TAG}

                            echo 'Starting Order service...'

                            docker run -d \
                                --name order-service \
                                --restart unless-stopped \
                                --network ${DOCKER_NETWORK} \
                                -p 8083:8083 \
                                -e INVENTORY_SERVICE_URL=http://inventory-service:8082 \
                                ${ECR_REGISTRY}/${ORDER_REPO}:${IMAGE_TAG}

                            echo 'Waiting for containers to start...'

                            sleep 5

                            echo 'Verifying Inventory container...'

                            docker inspect -f '{{.State.Running}}' inventory-service | grep -q true

                            echo 'Verifying Order container...'

                            docker inspect -f '{{.State.Running}}' order-service | grep -q true

                            echo 'Deployment successful!'

                            docker ps --filter name=inventory-service
                            docker ps --filter name=order-service
                        "
                    '''
                }
            }
        }
    }

    post {
        success {
            echo """
            Deployment successful!

            Inventory Service:
            http://${JUMP_HOST}:8082

            Order Service:
            http://${JUMP_HOST}:8083
            """
        }

        failure {
            echo 'Pipeline failed. Check the Jenkins console output.'
        }
    }
}
