pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-south-1'

        ECR_REGISTRY = '239711841813.dkr.ecr.ap-south-1.amazonaws.com'
        ECR_REPOSITORY = 'devops-cicd-demo'

        IMAGE_TAG = "${BUILD_NUMBER}"
        IMAGE_NAME = "${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG}"

        TEST_CONTAINER = 'devops-cicd-demo-test'

        APP_SERVER = '172.31.4.226'
        DEPLOY_CONTAINER = 'devops-cicd-demo'

        SSH_KEY = '/var/lib/jenkins/.ssh/devops-cicd-key.pem'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Test Application') {
            steps {
                sh '''
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install -r requirements.txt
                    pytest -v
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    docker build -t ${IMAGE_NAME} .
                '''
            }
        }

        stage('Docker Test') {
            steps {
                sh '''
                    docker rm -f ${TEST_CONTAINER} 2>/dev/null || true

                    docker run -d \
                        --name ${TEST_CONTAINER} \
                        -p 18000:8000 \
                        ${IMAGE_NAME}

                    sleep 5

                    curl --fail http://127.0.0.1:18000/health

                    docker logs ${TEST_CONTAINER}
                '''
            }
        }

        stage('ECR Login') {
            steps {
                sh '''
                    /usr/local/bin/aws ecr get-login-password \
                        --region ${AWS_REGION} | \
                    docker login \
                        --username AWS \
                        --password-stdin ${ECR_REGISTRY}
                '''
            }
        }

        stage('Push to ECR') {
            steps {
                sh '''
                    docker push ${IMAGE_NAME}
                '''
            }
        }

        stage('Deploy to Application EC2') {
            steps {
                sh '''
                    ssh \
                    -i ${SSH_KEY} \
                    -o StrictHostKeyChecking=no \
                    ubuntu@${APP_SERVER} \
                    "AWS_REGION=${AWS_REGION} \
                     ECR_REGISTRY=${ECR_REGISTRY} \
                     ECR_REPOSITORY=${ECR_REPOSITORY} \
                     IMAGE_TAG=${IMAGE_TAG} \
                     DEPLOY_CONTAINER=${DEPLOY_CONTAINER} \
                     bash -s" << 'REMOTE_SCRIPT'

                    set -e

                    echo "Logging into ECR..."

                    /usr/local/bin/aws ecr get-login-password \
                        --region "$AWS_REGION" | \
                    docker login \
                        --username AWS \
                        --password-stdin "$ECR_REGISTRY"

                    echo "Pulling image..."

                    docker pull \
                        "$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG"

                    echo "Stopping old container..."

                    docker stop "$DEPLOY_CONTAINER" 2>/dev/null || true

                    echo "Removing old container..."

                    docker rm "$DEPLOY_CONTAINER" 2>/dev/null || true

                    echo "Starting new container..."

                    docker run -d \
                        --name "$DEPLOY_CONTAINER" \
                        -p 8000:8000 \
                        "$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG"

                    echo "Waiting for application..."

                    sleep 5

                    echo "Health check..."

                    curl --fail \
                        http://127.0.0.1:8000/health

                    echo "Deployment successful!"

                    REMOTE_SCRIPT
                '''
            }
        }
    }

    post {
        always {
            sh '''
                docker rm -f ${TEST_CONTAINER} 2>/dev/null || true
            '''
        }

        success {
            echo 'CI/CD pipeline completed successfully!'
            echo "Deployed image: ${IMAGE_NAME}"
        }

        failure {
            echo 'CI/CD pipeline failed.'
        }
    }
}