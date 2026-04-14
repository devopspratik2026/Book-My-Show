# Create a new Jenkinsfile or edit existing one
cat > Jenkinsfile << 'EOF'
pipeline {
    agent any
    
    tools {
        jdk 'jdk17'
        nodejs 'node23'
    }
    
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        DOCKER_REGISTRY = 'devopsengineerpratik'
        IMAGE_NAME = 'bms'
        IMAGE_TAG = "${BUILD_NUMBER}"
    }
    
    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }
        
        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/devopspratik2026/Book-My-Show.git'
                sh 'ls -la'
            }
        }
        
        stage('Install Dependencies') {
            steps {
                sh '''
                    echo "Installing Node.js dependencies..."
                    
                    if [ -d "bookmyshow-app" ]; then
                        cd bookmyshow-app
                        ls -la
                    else
                        pwd
                        ls -la
                    fi
                    
                    if [ -f package.json ]; then
                        echo "Found package.json, installing dependencies..."
                        rm -rf node_modules package-lock.json
                        npm install
                    else
                        echo "Error: package.json not found!"
                        exit 1
                    fi
                '''
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    withCredentials([string(credentialsId: 'Sonar-token', variable: 'SONAR_TOKEN')]) {
                        sh '''
                            echo "Running SonarQube analysis..."
                            $SCANNER_HOME/bin/sonar-scanner \
                            -Dsonar.projectName=BMS \
                            -Dsonar.projectKey=BMS \
                            -Dsonar.sources=. \
                            -Dsonar.exclusions=**/node_modules/**,**/dist/**,**/build/** \
                            -Dsonar.token=$SONAR_TOKEN \
                            -Dsonar.host.url=http://localhost:9000
                        '''
                    }
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }
        
        stage('OWASP Dependency Check') {
            steps {
                script {
                    dependencyCheck additionalArguments: '--scan ./ --format HTML --format JSON', 
                                   odcInstallation: 'DP-Check'
                    
                    publishHTML([
                        reportDir: '.',
                        reportFiles: 'dependency-check-report.html',
                        reportName: 'OWASP Dependency-Check Report'
                    ])
                }
            }
        }
        
        stage('Trivy FS Scan') {
            steps {
                sh '''
                    echo "Running Trivy filesystem scan..."
                    trivy fs . --format table --output trivyfs.txt || true
                    cat trivyfs.txt
                '''
            }
        }
        
        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh '''
                            echo "Building Docker image..."
                            
                            if [ -d "bookmyshow-app" ]; then
                                docker build --no-cache \
                                    -t ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} \
                                    -t ${DOCKER_REGISTRY}/${IMAGE_NAME}:latest \
                                    -f bookmyshow-app/Dockerfile \
                                    bookmyshow-app
                            else
                                docker build --no-cache \
                                    -t ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} \
                                    -t ${DOCKER_REGISTRY}/${IMAGE_NAME}:latest \
                                    .
                            fi
                            
                            echo "Pushing Docker image to registry..."
                            docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
                            docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:latest
                        '''
                    }
                }
            }
        }
        
        stage('Trivy Image Scan') {
            steps {
                sh '''
                    echo "Running Trivy image scan..."
                    trivy image --format table --output trivyimage.txt ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} || true
                    cat trivyimage.txt
                '''
            }
        }
        
        stage('Deploy to Container') {
            steps {
                sh '''
                    echo "Stopping and removing old container..."
                    docker stop bms || true
                    docker rm bms || true
                    
                    echo "Running new container on port 3000..."
                    docker run -d \
                        --restart=always \
                        --name bms \
                        -p 3000:3000 \
                        ${DOCKER_REGISTRY}/${IMAGE_NAME}:latest
                    
                    echo "Checking running containers..."
                    docker ps -a
                    
                    sleep 5
                    docker logs bms || echo "No logs available yet"
                '''
            }
        }
    }
    
    post {
        always {
            echo "Pipeline completed - Status: ${currentBuild.result}"
        }
        
        success {
            echo "✅ Pipeline executed successfully!"
            emailext(
                attachLog: true,
                subject: "✅ Build Successful - ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
                    <h2>Build Successful</h2>
                    <p><b>Project:</b> ${env.JOB_NAME}</p>
                    <p><b>Build Number:</b> ${env.BUILD_NUMBER}</p>
                    <p><b>Build URL:</b> ${env.BUILD_URL}</p>
                    <p><b>Docker Image:</b> ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}</p>
                """,
                to: 'devops.engineer.pratik@gmail.com',
                attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
            )
        }
        
        failure {
            echo "❌ Pipeline failed!"
            emailext(
                attachLog: true,
                subject: "❌ Build Failed - ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
                    <h2>Build Failed</h2>
                    <p><b>Project:</b> ${env.JOB_NAME}</p>
                    <p><b>Build Number:</b> ${env.BUILD_NUMBER}</p>
                    <p><b>Build URL:</b> ${env.BUILD_URL}</p>
                """,
                to: 'devops.engineer.pratik@gmail.com'
            )
        }
    }
}
EOF
