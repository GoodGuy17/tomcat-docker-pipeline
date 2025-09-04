pipeline {
    agent any
    
    environment {
        DOCKER_IMAGE = "my-tomcat-app"
        DOCKER_TAG = "${BUILD_NUMBER}"
        TEST_PORT = "9090"
        APP_CONTEXT = "Amazon"
    }
    
    stages {
        stage('Cleanup Previous') {
            steps {
                script {
                    // Clean up any existing test containers
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
                    def image = docker.build("${DOCKER_IMAGE}:${DOCKER_TAG}")
                    echo "Docker image built successfully"
                }
            }
        }
        
        stage('Container Test') {
            steps {
                script {
                    echo "Starting container for testing..."
                    sh """
                        # Start test container
                        docker run -d --name test-container -p ${TEST_PORT}:8080 ${DOCKER_IMAGE}:${DOCKER_TAG}
                        
                        # Wait for Tomcat to fully start
                        echo "Waiting for Tomcat to start..."
                        sleep 45
                        
                        # Check container logs
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
                        # Test Tomcat is running
                        curl -f http://localhost:${TEST_PORT}/ && echo "✓ Tomcat is accessible"
                        
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
                        # Check container health
                        container_status=\$(docker inspect test-container --format='{{.State.Status}}')
                        if [ "\$container_status" = "running" ]; then
                            echo "✓ Container is running healthy"
                        else
                            echo "✗ Container status: \$container_status"
                            exit 1
                        fi
                        
                        # Check resource usage
                        echo "=== Container Resource Usage ==="
                        docker stats test-container --no-stream
                        
                        # Check if WAR is properly deployed
                        docker exec test-container ls -la /usr/local/tomcat/webapps/
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
                    # Stop and remove test container
                    docker stop test-container || true
                    docker rm test-container || true
                    
                    # Optional: Clean up old test images (keep last 3)
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
