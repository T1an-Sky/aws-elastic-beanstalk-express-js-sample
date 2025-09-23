pipeline {
    agent {
        docker {
            image 'node:16'  // Use Node.js 16 as build agent
            args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
    }
    
    environment {
        // Docker registry credentials (replace with your Docker Hub username)
        DOCKER_REGISTRY = 'T1an-Sky'  // 改为你的 Docker Hub 用户名
        IMAGE_NAME = 'node-express-app'
        IMAGE_TAG = "${BUILD_NUMBER}"
        
        // Security scan thresholds
        VULNERABILITY_THRESHOLD = 'high'
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out source code...'
                checkout scm
            }
        }
        
        stage('Install Dependencies') {
            steps {
                echo 'Installing Node.js dependencies...'
                sh 'npm install --save'
                sh 'npm list --depth=0'  // Show installed packages
            }
        }
        
        stage('Run Unit Tests') {
            steps {
                echo 'Running unit tests...'
                sh 'npm test || echo "No tests found, continuing..."'
            }
            post {
                always {
                    // Archive test results if they exist
                    script {
                        if (fileExists('test-results.xml')) {
                            publishTestResults([
                                testResultsPattern: 'test-results.xml'
                            ])
                        }
                    }
                }
            }
        }
        
        stage('Security Vulnerability Scan') {
            steps {
                echo 'Running dependency vulnerability scan...'
                script {
                    // Install and run npm audit
                    sh 'npm audit --audit-level=high --json > audit-results.json || true'
                    
                    // Check audit results
                    def auditResults = readJSON file: 'audit-results.json'
                    
                    if (auditResults.metadata && auditResults.metadata.vulnerabilities) {
                        def highVulns = auditResults.metadata.vulnerabilities.high || 0
                        def criticalVulns = auditResults.metadata.vulnerabilities.critical || 0
                        
                        echo "High vulnerabilities found: ${highVulns}"
                        echo "Critical vulnerabilities found: ${criticalVulns}"
                        
                        if (highVulns > 0 || criticalVulns > 0) {
                            error "Security scan failed: Found ${highVulns} high and ${criticalVulns} critical vulnerabilities"
                        }
                    }
                }
            }
            post {
                always {
                    // Archive security scan results
                    archiveArtifacts artifacts: 'audit-results.json', fingerprint: true
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                echo 'Building Docker image...'
                script {
                    // Create Dockerfile if it doesn't exist
                    if (!fileExists('Dockerfile')) {
                        writeFile file: 'Dockerfile', text: '''
FROM node:16-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --production
COPY . .
EXPOSE 3000
CMD ["npm", "start"]
'''
                    }
                    
                    // Build Docker image
                    def image = docker.build("${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}")
                    
                    // Tag as latest
                    sh "docker tag ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} ${DOCKER_REGISTRY}/${IMAGE_NAME}:latest"
                    
                    echo "Docker image built successfully: ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"
                }
            }
        }
        
        stage('Push Docker Image') {
            when {
                branch 'main'  // Only push on main branch
            }
            steps {
                echo 'Pushing Docker image to registry...'
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'docker-hub-credentials') {
                        def image = docker.image("${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}")
                        image.push()
                        image.push('latest')
                    }
                    echo "Image pushed successfully"
                }
            }
        }
    }
    
    post {
        always {
            echo 'Pipeline completed'
            // Clean up workspace
            cleanWs()
        }
        success {
            echo 'Pipeline succeeded!'
            // Send success notification (optional)
        }
        failure {
            echo 'Pipeline failed!'
            // Send failure notification (optional)
        }
    }
}
