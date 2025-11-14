pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'sampleapp'
        CONTAINER_NAME = 'samplerunning'
        APP_PORT = '5050'
    }

    stages {
        stage('Preparation') {
            steps {
                script {
                    echo "Cleaning up any existing containers and images..."
                    catchError(buildResult: 'SUCCESS', stageResult: 'SUCCESS') {
                        sh """
                            docker stop ${CONTAINER_NAME} || true
                            docker rm ${CONTAINER_NAME} || true
                            docker image rm ${DOCKER_IMAGE}:latest || true
                        """
                    }
                }
            }
        }

        stage('Checkout') {
            steps {
                // Clean workspace
                cleanWs()

                // Checkout the repository
                checkout scm

                echo "Repository checked out successfully"
                sh 'ls -la'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    echo "Building Docker image: ${DOCKER_IMAGE}"

                    sh """
                        # Create temporary build directory
                        rm -rf tempdir
                        mkdir -p tempdir/templates tempdir/static

                        # Copy application files
                        cp sample_app.py tempdir/.
                        cp -r templates/* tempdir/templates/.
                        cp -r static/* tempdir/static/.

                        # Create Dockerfile
                        cat > tempdir/Dockerfile << 'EOF'
FROM python:3.9-slim

# Set working directory
WORKDIR /home/myapp

# Install dependencies
RUN pip install --no-cache-dir flask

# Copy application files
COPY ./static /home/myapp/static/
COPY ./templates /home/myapp/templates/
COPY sample_app.py /home/myapp/

# Expose port
EXPOSE 5050

# Run the application
CMD ["python3", "sample_app.py"]
EOF

                        # Build the Docker image
                        cd tempdir
                        docker build -t ${DOCKER_IMAGE}:latest .
                    """
                }
            }
        }

        stage('Run Application') {
            steps {
                script {
                    echo "Starting application container: ${CONTAINER_NAME}"

                    sh """
                        docker run -d \\
                            --name ${CONTAINER_NAME} \\
                            -p ${APP_PORT}:${APP_PORT} \\
                            ${DOCKER_IMAGE}:latest
                    """

                    echo "Container started successfully"
                    sh 'docker ps -a | grep ${CONTAINER_NAME}'
                }
            }
        }

        stage('Wait for Application') {
            steps {
                script {
                    echo "Waiting for application to be ready..."

                    // Wait up to 30 seconds for the app to start
                    timeout(time: 30, unit: 'SECONDS') {
                        waitUntil {
                            try {
                                sh script: """
                                    curl --connect-timeout 2 --max-time 5 \\
                                        --silent --fail \\
                                        http://localhost:${APP_PORT}/ >/dev/null 2>&1
                                """, returnStatus: true
                                return true
                            } catch (Exception e) {
                                echo "Application not ready yet, waiting..."
                                sleep 2
                                return false
                            }
                        }
                    }

                    echo "Application is ready!"
                }
            }
        }

        stage('Test Application') {
            steps {
                script {
                    echo "Testing application functionality..."

                    sh """
                        # Test 1: Basic connectivity
                        echo "Test 1: Checking basic connectivity..."
                        curl --fail http://localhost:${APP_PORT}/

                        # Test 2: Check for expected content
                        echo "Test 2: Verifying expected response content..."
                        if curl http://localhost:${APP_PORT}/ | grep -q "You are calling me from"; then
                            echo "✅ Response content test PASSED"
                        else
                            echo "❌ Response content test FAILED"
                            exit 1
                        fi

                        # Test 3: Check container is running
                        echo "Test 3: Verifying container status..."
                        if docker ps --filter "name=${CONTAINER_NAME}" --filter "status=running" --format '{{.Names}}' | grep -q '^${CONTAINER_NAME}$'; then
                            echo "✅ Container status test PASSED"
                        else
                            echo "❌ Container status test FAILED"
                            exit 1
                        fi

                        echo "🎉 All tests PASSED!"
                    """
                }
            }
        }

        stage('Application Info') {
            steps {
                script {
                    echo "=== Application Information ==="
                    sh """
                        echo "Container status:"
                        docker ps | grep ${CONTAINER_NAME} || echo "Container not found in running list"

                        echo ""
                        echo "Application logs (last 10 lines):"
                        docker logs --tail 10 ${CONTAINER_NAME}

                        echo ""
                        echo "Port binding:"
                        docker port ${CONTAINER_NAME} || echo "No port mappings found"
                    """
                }
            }
        }
    }

    post {
        success {
            echo """
            🎉 PIPELINE SUCCESS! 🎉

            Application is running successfully:
            - URL: http://localhost:${APP_PORT}
            - Container: ${CONTAINER_NAME}
            - Image: ${DOCKER_IMAGE}:latest

            You can access the application at: http://localhost:${APP_PORT}
            """

            // Optional: Send success notification
            // emailext (
            //     subject: "✅ Sample App Pipeline Success: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
            //     body: "The sample application pipeline completed successfully.\\n\\nBuild URL: ${env.BUILD_URL}"
            // )
        }

        failure {
            echo """
            ❌ PIPELINE FAILED! ❌

            Troubleshooting steps:
            1. Check the stage that failed above
            2. Review container logs: docker logs ${CONTAINER_NAME}
            3. Check port availability: ss -lntp | grep ${APP_PORT}
            4. Verify Docker images: docker images | grep ${DOCKER_IMAGE}
            """

            // Optional: Send failure notification
            // emailext (
            //     subject: "❌ Sample App Pipeline FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
            //     body: "The sample application pipeline FAILED.\\n\\nBuild URL: ${env.BUILD_URL}\\n\\nPlease check the logs for details."
            // )
        }

        cleanup {
            script {
                echo "Cleaning up workspace..."
                cleanWs()

                // Keep the application running after successful build
                // Comment out the following lines if you want to auto-stop the container
                if (currentBuild.currentResult == 'SUCCESS') {
                    echo "✅ Application container ${CONTAINER_NAME} left running for use"
                    echo "📝 To stop manually: docker stop ${CONTAINER_NAME}"
                } else {
                    echo "❌ Stopping container due to build failure"
                    catchError(buildResult: 'SUCCESS', stageResult: 'SUCCESS') {
                        sh """
                            docker stop ${CONTAINER_NAME} || true
                            docker rm ${CONTAINER_NAME} || true
                        """
                    }
                }
            }
        }
    }
}