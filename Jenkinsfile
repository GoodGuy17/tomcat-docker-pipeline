pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "my-tomcat-app"
        DOCKER_TAG = "${BUILD_NUMBER}"
        TEST_PORT = "9091"
        APP_CONTEXT = "Amazon"
    }

    stages {
        stage('Cleanup Previous') {
            steps {
                script {
                    sh '''
                        docker stop test-container || true
                        docker rm test-container || true
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    echo "Building Docker image..."
                    sh "docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} ."
                    echo "Docker image built successfully"
                }
            }
        }

        stage('Container Test') {
            steps {
                script {
                    echo "Starting container for testing..."
                    sh """
                        docker run -d --name test-container -p ${TEST_PORT}:8080 ${DOCKER_IMAGE}:${DOCKER_TAG}
                        echo "Waiting for Tomcat to start..."
                        sleep 45
                        echo "=== Container Logs ==="
                        docker logs test-container
                    """
                }
            }
        }

        stage('Application Tests') {
            steps {
                script {
                    echo "Testing application endpoints..."
                    sh """
                        # Test Amazon application
                        curl -f http://localhost:${TEST_PORT}/${APP_CONTEXT}/ && echo "✓ Amazon app is accessible"

                        # Test response status
                        response_code=\$(curl -s -o /dev/null -w "%{http_code}" http://localhost:${TEST_PORT}/${APP_CONTEXT}/)
                        if [ \$response_code -eq 200 ]; then
                            echo "✓ Application returns HTTP 200"
                        else
                            echo "✗ Application returned HTTP \$response_code"
                            exit 1
                        fi

                        # Basic content check
                        curl -s http://localhost:${TEST_PORT}/${APP_CONTEXT}/ | grep -q "html\\|HTML" && echo "✓ HTML content detected"

                        echo "All tests passed successfully!"
                    """
                }
            }
        }

        stage('Health Check') {
            steps {
                script {
                    echo "Running health checks..."
                    sh """
                        container_status=\$(docker inspect test-container --format='{{.State.Status}}')
                        if [ "\$container_status" = "running" ]; then
                            echo "✓ Container is running healthy"
                        else
                            echo "✗ Container status: \$container_status"
                            exit 1
                        fi

                        echo "=== Container Resource Usage ==="
                        docker stats test-container --no-stream

                        docker exec test-container ls -la /usr/local/tomcat/webapps/
                    """
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    echo "Deploying container..."
                    sh """
                        docker stop prod-container || true
                        docker rm prod-container || true
                        docker run -d --name prod-container -p 9092:8080 ${DOCKER_IMAGE}:${DOCKER_TAG}
                    """
                }
            }
        }
    }

    post {
        always {
            script {
                echo "Cleaning up test resources..."
                sh '''
                    docker stop test-container || true
                    docker rm test-container || true
                    docker images ${DOCKER_IMAGE} --format "table {{.Tag}}" | grep -E "^[0-9]+$" | sort -nr | tail -n +4 | xargs -r -I {} docker rmi ${DOCKER_IMAGE}:{} || true
                '''
            }
        }

        success {
            echo "🎉 All tests passed! Docker image is ready for deployment."
        }

        failure {
            echo "❌ Tests failed. Check the logs above for details."
            script {
                sh '''
                    echo "=== Failed Container Logs ==="
                    docker logs test-container || true
                '''
            }
        }
    }
}
