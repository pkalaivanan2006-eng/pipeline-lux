pipeline {
    agent any

    environment {
        IMAGE_NAME = "kalai0011/lux"
        IMAGE_TAG = "${BUILD_NUMBER}"
        TEST_CONTAINER = "test-container"
        APP_CONTAINER = "flask-app"
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                docker build -t $IMAGE_NAME:$IMAGE_TAG .
                '''
            }
        }

        stage('Test Container') {
            steps {
                sh '''
                # Remove old test container if it exists
                docker stop $TEST_CONTAINER || true
                docker rm $TEST_CONTAINER || true

                # Run new test container
                docker run -d \
                  --name $TEST_CONTAINER \
                  -p 5002:5000 \
                  $IMAGE_NAME:$IMAGE_TAG

                sleep 10

                # Check container is running
                docker ps

                docker ps | grep $TEST_CONTAINER

                # Cleanup
                docker stop $TEST_CONTAINER
                docker rm $TEST_CONTAINER
                '''
            }
        }

        stage('Docker Login') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-cred',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh '''
                    echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                    '''
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                sh '''
                docker push $IMAGE_NAME:$IMAGE_TAG

                docker tag $IMAGE_NAME:$IMAGE_TAG $IMAGE_NAME:latest

                docker push $IMAGE_NAME:latest
                '''
            }
        }

        stage('Deploy Application') {
            steps {
                sh '''
                # Stop old application
                docker stop $APP_CONTAINER || true
                docker rm $APP_CONTAINER || true

                # Remove old image (optional)
                docker image prune -f || true

                # Run latest application
                docker run -d \
                  --name $APP_CONTAINER \
                  -p 5000:5000 \
                  --restart unless-stopped \
                  $IMAGE_NAME:$IMAGE_TAG

                sleep 5

                docker ps
                '''
            }
        }
    }

    post {

        success {
            echo "CI/CD Pipeline Completed Successfully"
        }

        failure {
            echo "Pipeline Failed"

            sh '''
            docker logs $TEST_CONTAINER || true
            docker logs $APP_CONTAINER || true
            '''
        }

        always {
            sh '''
            docker image prune -f || true
            docker container prune -f || true
            '''
        }
    }
}
