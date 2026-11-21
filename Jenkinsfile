pipeline {
    agent any

    environment {
        AWS_REGION       = 'us-west-2'
        AWS_ACCOUNT_ID   = '212773812862'
        ECR_REGISTRY     = '212773812862.dkr.ecr.us-west-2.amazonaws.com'

        INVENTORY_REPO   = 'inventory-service'
        ORDER_REPO       = 'order-service'

        IMAGE_TAG        = "${BUILD_NUMBER}"

        JUMP_HOST        = '35.86.117.150'
        SSH_USER         = 'ec2-user'
        SSH_CREDENTIALS  = 'target-server-ssh-key'

        DEPLOY_DIR       = '/home/ec2-user/inventory-order'
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

        stage('Copy Docker Compose File') {
            steps {
                sshagent(credentials: [SSH_CREDENTIALS]) {
                    sh '''
                        set -e

                        ssh -o StrictHostKeyChecking=accept-new \
                            ${SSH_USER}@${JUMP_HOST} \
                            "mkdir -p ${DEPLOY_DIR}"

                        scp -o StrictHostKeyChecking=accept-new \
                            docker-compose.yml \
                            ${SSH_USER}@${JUMP_HOST}:${DEPLOY_DIR}/docker-compose.yml
                    '''
                }
            }
        }

        stage('Deploy Using Docker Compose') {
            steps {
                sshagent(credentials: [SSH_CREDENTIALS]) {
                    sh '''
                        set -e

                        ssh -o StrictHostKeyChecking=accept-new \
                            ${SSH_USER}@${JUMP_HOST} "

                            set -e

                            echo 'Checking Docker Compose...'
                            docker compose version

                            echo 'Logging in to ECR...'

                            aws ecr get-login-password --region ${AWS_REGION} | \
                            docker login --username AWS --password-stdin ${ECR_REGISTRY}

                            cd ${DEPLOY_DIR}

                            export ECR_REGISTRY=${ECR_REGISTRY}
                            export IMAGE_TAG=${IMAGE_TAG}

                            echo 'Validating Compose configuration...'
                            docker compose config -q

                            echo 'Pulling latest images...'
                            docker compose pull

                            echo 'Removing old manually deployed containers if present...'
                            docker rm -f inventory-service order-service >/dev/null 2>&1 || true

                            echo 'Starting services with Docker Compose...'
                            docker compose up -d --force-recreate --remove-orphans

                            echo 'Waiting for services...'
                            sleep 5

                            echo 'Checking Inventory service...'
                            docker compose ps --status running -q inventory-service | grep -q .

                            echo 'Checking Order service...'
                            docker compose ps --status running -q order-service | grep -q .

                            echo 'Deployment successful!'

                            docker compose ps
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
