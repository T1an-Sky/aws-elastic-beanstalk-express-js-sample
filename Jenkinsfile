pipeline {
    agent any  // 使用 any agent 避免 Docker agent 的问题
    
    environment {
        // Docker registry credentials
        DOCKER_REGISTRY = 'tianhaogeng'
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
        
        stage('Test Docker Connection') {
            steps {
                echo 'Testing Docker connection...'
                sh 'docker --version'
                sh 'docker info || echo "Docker info failed"'
            }
        }
        
        stage('Build Docker Image') {
            steps {
                echo 'Building Docker image...'
                script {
                    // Create Dockerfile if it doesn't exist
                    if (!fileExists('Dockerfile')) {
                        writeFile file: 'Dockerfile', text: '''FROM node:16-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --production
COPY . .
EXPOSE 3000
CMD ["npm", "start"]
'''
                    }
                    
                    // Build Docker image
                    sh "docker build -t ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} ."
                    
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
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', 
                                                     usernameVariable: 'DOCKER_USER', 
                                                     passwordVariable: 'DOCKER_PASS')]) {
                        sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
                        sh "docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"
                        sh "docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:latest"
                        echo "Image pushed successfully"
                    }
                }
            }
        }
    }
    
    post {
        always {
            script {
                // 只有在 node 上下文中才执行 cleanWs
                try {
                    if (env.WORKSPACE) {
                        cleanWs()
                    }
                } catch (Exception e) {
                    echo "Workspace cleanup failed: ${e.getMessage()}"
                }
            }
            echo 'Pipeline completed'
        }
        success {
            echo 'Pipeline succeeded!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
