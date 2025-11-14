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
                            docker stop ${env.CONTAINER_NAME} || true
                            docker rm ${env.CONTAINER_NAME} || true
                            docker image rm ${env.DOCKER_IMAGE}:latest || true
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
                    echo "Building Docker image: ${env.DOCKER_IMAGE}"

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
                        docker build -t ${env.DOCKER_IMAGE}:latest .
                    """
                }
            }
        }

        stage('Run Application') {
            steps {
                script {
                    echo "Starting application container: ${env.CONTAINER_NAME}"

                    sh """
                        docker run -d \\
                            --name ${env.CONTAINER_NAME} \\
                            -p ${env.APP_PORT}:${env.APP_PORT} \\
                            ${env.DOCKER_IMAGE}:latest
                    """

                    echo "Container started successfully"
                    sh "docker ps -a | grep ${env.CONTAINER_NAME}"
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
                                        http://localhost:${env.APP_PORT}/ >/dev/null 2>&1
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
                        curl --fail http://localhost:${env.APP_PORT}/

                        # Test 2: Check for expected content
                        echo "Test 2: Verifying expected response content..."
                        if curl http://localhost:${env.APP_PORT}/ | grep -q "You are calling me from"; then
                            echo "✅ Response content test PASSED"
                        else
                            echo "❌ Response content test FAILED"
                            exit 1
                        fi

                        # Test 3: Check container is running
                        echo "Test 3: Verifying container status..."
                        if docker ps --filter "name=${env.CONTAINER_NAME}" --filter "status=running" | grep -q "${env.CONTAINER_NAME}"; then
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
                        docker ps | grep ${env.CONTAINER_NAME} || echo "Container not found in running list"

                        echo ""
                        echo "Application logs (last 10 lines):"
                        docker logs --tail 10 ${env.CONTAINER_NAME}

                        echo ""
                        echo "Port binding:"
                        docker port ${env.CONTAINER_NAME} || echo "No port mappings found"
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
            - URL: http://localhost:${env.APP_PORT}
            - Container: ${env.CONTAINER_NAME}
            - Image: ${env.DOCKER_IMAGE}:latest

            You can access the application at: http://localhost:${env.APP_PORT}
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
            2. Review container logs: docker logs ${env.CONTAINER_NAME}
            3. Check port availability: ss -lntp | grep ${env.APP_PORT}
            4. Verify Docker images: docker images | grep ${env.DOCKER_IMAGE}
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
                    echo "✅ Application container ${env.CONTAINER_NAME} left running for use"
                    echo "📝 To stop manually: docker stop ${env.CONTAINER_NAME}"
                } else {
                    echo "❌ Stopping container due to build failure"
                    catchError(buildResult: 'SUCCESS', stageResult: 'SUCCESS') {
                        sh """
                            docker stop ${env.CONTAINER_NAME} || true
                            docker rm ${env.CONTAINER_NAME} || true
                        """
                    }
                }
            }
        }
    }
}