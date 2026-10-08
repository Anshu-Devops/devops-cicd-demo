pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-south-1'
        ECR_REGISTRY = '239711841813.dkr.ecr.ap-south-1.amazonaws.com'
        ECR_REPOSITORY = 'devops-cicd-demo'
        IMAGE_TAG = "${BUILD_NUMBER}"
        IMAGE_NAME = "${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG}"
        TEST_CONTAINER = 'devops-cicd-demo-test'
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

                    docker ps

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
    }

    post {
        always {
            sh '''
                docker rm -f ${TEST_CONTAINER} 2>/dev/null || true
            '''
        }

        success {
            echo 'CI pipeline completed successfully!'
            echo "Docker image pushed: ${IMAGE_NAME}"
        }

        failure {
            echo 'CI pipeline failed. Check the stage logs.'
        }
    }
}