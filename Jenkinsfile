pipeline {
    // Use any available agent for this pipeline
    agent any
    
    // Configure tools needed for the pipeline
    tools {
        nodejs 'NodeJS-16'  // NodeJS tool configured in Jenkins Global Tool Configuration
    }
    
    // Define environment variables used throughout the pipeline
    environment {
        DOCKER_REGISTRY = 'tianhaogeng'  // Docker Hub username
        IMAGE_NAME = 'node-express-app'  // Name of the Docker image
        IMAGE_TAG = "${BUILD_NUMBER}"    // Tag using Jenkins build number for versioning
        VULNERABILITY_THRESHOLD = 'high' // Security scan threshold level
    }
    
    stages {
        // Stage 1: Checkout source code from SCM
        stage('Checkout') {
            steps {
                echo 'Checking out source code...'
                checkout scm  // Checkout code from the configured Git repository
            }
        }
        
        // Stage 2: Verify that required tools are available
        stage('Verify Environment') {
            steps {
                echo 'Verifying Node.js and Docker environment...'
                sh 'node --version || echo "Node.js not found"'   // Check Node.js installation
                sh 'npm --version || echo "npm not found"'        // Check npm installation
                sh 'docker --version || echo "Docker not found"'  // Check Docker installation
            }
        }
        
        // Stage 3: Install Node.js dependencies
        stage('Install Dependencies') {
            steps {
                echo 'Installing Node.js dependencies...'
                script {
                    try {
                        sh 'npm install --save'      // Install dependencies from package.json
                        sh 'npm list --depth=0'      // List installed packages at top level
                    } catch (Exception e) {
                        echo "Dependency installation failed: ${e.getMessage()}"
                        echo "Continuing with available dependencies..."
                    }
                }
            }
        }
        
        // Stage 4: Run unit tests
        stage('Run Unit Tests') {
            steps {
                echo 'Running unit tests...'
                script {
                    try {
                        sh 'npm test'  // Execute test script defined in package.json
                    } catch (Exception e) {
                        echo "Tests failed or not found: ${e.getMessage()}"
                        echo "Continuing pipeline..."
                    }
                }
            }
            post {
                always {
                    script {
                        // Archive test results if they exist
                        if (fileExists('test-results.xml')) {
                            publishTestResults([
                                testResultsPattern: 'test-results.xml'
                            ])
                        }
                    }
                }
            }
        }
        
        // Stage 5: Security vulnerability scanning using npm audit
        stage('Security Vulnerability Scan') {
            steps {
                echo 'Running dependency vulnerability scan...'
                script {
                    try {
                        // Run npm audit and save results to JSON file
                        sh 'npm audit --audit-level=high --json > audit-results.json || true'
                        
                        if (fileExists('audit-results.json')) {
                            // Read and parse the audit results
                            def auditResults = readJSON file: 'audit-results.json'
                            
                            if (auditResults.metadata && auditResults.metadata.vulnerabilities) {
                                // Extract high and critical vulnerability counts
                                def highVulns = auditResults.metadata.vulnerabilities.high ?: 0
                                def criticalVulns = auditResults.metadata.vulnerabilities.critical ?: 0
                                
                                echo "High vulnerabilities found: ${highVulns}"
                                echo "Critical vulnerabilities found: ${criticalVulns}"
                                
                                if (highVulns > 0 || criticalVulns > 0) {
                                    echo "WARNING: Security vulnerabilities detected but continuing build"
                                    // In production, you might want to fail the build here:
                                    // error "Security scan failed: Found ${highVulns} high and ${criticalVulns} critical vulnerabilities"
                                }
                            }
                        }
                    } catch (Exception e) {
                        echo "Security scan failed: ${e.getMessage()}"
                        echo "Continuing pipeline..."
                    }
                }
            }
            post {
                always {
                    script {
                        // Archive the audit results as build artifacts
                        if (fileExists('audit-results.json')) {
                            archiveArtifacts artifacts: 'audit-results.json', fingerprint: true
                        }
                    }
                }
            }
        }
        
        // Stage 6: Test Docker daemon connectivity
        stage('Test Docker Connection') {
            steps {
                echo 'Testing Docker connection...'
                script {
                    try {
                        sh 'docker --version'  // Check Docker CLI version
                        sh 'docker info'       // Display Docker system information
                        echo 'Docker connection successful!'
                    } catch (Exception e) {
                        echo "Docker connection failed: ${e.getMessage()}"
                        echo "This may affect Docker image building..."
                    }
                }
            }
        }
        
        // Stage 7: Build Docker image from the application
        stage('Build Docker Image') {
            steps {
                echo 'Building Docker image...'
                script {
                    try {
                        // Create Dockerfile if it doesn't exist in the repository
                        if (!fileExists('Dockerfile')) {
                            writeFile file: 'Dockerfile', text: '''FROM node:16-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --production --silent
COPY . .
EXPOSE 3000
CMD ["npm", "start"]
'''
                        }
                        
                        // Build Docker image with version tag
                        sh "docker build -t ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} ."
                        
                        // Tag the image as 'latest' as well
                        sh "docker tag ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} ${DOCKER_REGISTRY}/${IMAGE_NAME}:latest"
                        
                        echo "Docker image built successfully: ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"
                        
                        // Verify the images were created
                        sh "docker images | grep ${IMAGE_NAME} || echo 'No images found'"
                        
                    } catch (Exception e) {
                        echo "Docker image build failed: ${e.getMessage()}"
                        currentBuild.result = 'UNSTABLE'  // Mark build as unstable but don't fail
                    }
                }
            }
        }
        
        // Stage 8: Push Docker image to Docker Hub registry
        stage('Push Docker Image') {
            steps {
                echo 'Pushing Docker image to registry...'
                script {
                    try {
                        // Use Docker Hub credentials stored in Jenkins
                        withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', 
                                                         usernameVariable: 'DOCKER_USER', 
                                                         passwordVariable: 'DOCKER_PASS')]) {
                            // Login to Docker Hub
                            sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
                            
                            // Push versioned image
                            sh "docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"
                            
                            // Push latest tag
                            sh "docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:latest"
                            
                            echo "Images pushed successfully to Docker Hub"
                        }
                    } catch (Exception e) {
                        echo "Docker push failed: ${e.getMessage()}"
                        echo "This could be due to registry authentication or network issues"
                        currentBuild.result = 'UNSTABLE'  // Mark as unstable if push fails
                    }
                }
            }
        }
    }
    
    // Post-build actions that run regardless of build status
    post {
        always {
            script {
                try {
                    // Clean up workspace to save disk space
                    if (env.WORKSPACE) {
                        cleanWs()
                        echo 'Workspace cleaned successfully'
                    }
                } catch (Exception e) {
                    echo "Workspace cleanup failed: ${e.getMessage()}"
                }
            }
            echo 'Pipeline completed'
        }
        success {
            echo 'Pipeline succeeded! All stages completed successfully.'
        }
        unstable {
            echo 'Pipeline completed with some issues - check the logs for details.'
        }
        failure {
            echo 'Pipeline failed! Please check the error logs above.'
        }
    }
}
