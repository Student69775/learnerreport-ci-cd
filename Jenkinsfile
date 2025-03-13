pipeline {
    agent any
    environment {
        DOCKER_HUB_CREDS = credentials('docker-hub-credentials')
        DOCKER_USERNAME = 'student9393'
        FRONTEND_IMAGE_NAME = "${DOCKER_USERNAME}/frontend-image"
        BACKEND_IMAGE_NAME = "${DOCKER_USERNAME}/backend-image"
        KUBECONFIG = credentials('kubeconfig-credentials')
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Login to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    script {
                        if (isUnix()) {
                            sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
                        } else {
                            // Using password-stdin for security
                            bat 'echo %DOCKER_PASS% | docker login -u %DOCKER_USER% --password-stdin'
                        }
                    }
                }
            }
        }
        stage('Create Dockerfiles if Missing') {
            steps {
                script {
                    // Check and create frontend Dockerfile if missing
                    dir('learnerReportCS_frontend') {
                        if (isUnix()) {
                            sh '''
                                if [ ! -f Dockerfile ]; then
                                    echo "Creating frontend Dockerfile"
                                    echo 'FROM node:14-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build
EXPOSE 3000
CMD ["npm", "start"]' > Dockerfile
                                fi
                            '''
                        } else {
                            bat '''
                                if not exist Dockerfile (
                                    echo Creating frontend Dockerfile
                                    (
                                        echo FROM node:14-alpine
                                        echo WORKDIR /app
                                        echo COPY package*.json ./
                                        echo RUN npm install
                                        echo COPY . .
                                        echo RUN npm run build
                                        echo EXPOSE 3000
                                        echo CMD ["npm", "start"]
                                    ) > Dockerfile
                                )
                            '''
                        }
                    }
                    
                    // Check and create backend Dockerfile if missing
                    dir('learnerReportCS_backend') {
                        if (isUnix()) {
                            sh '''
                                if [ ! -f Dockerfile ]; then
                                    echo "Creating backend Dockerfile"
                                    echo 'FROM node:14-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 5000
CMD ["node", "app.js"]' > Dockerfile
                                fi
                            '''
                        } else {
                            bat '''
                                if not exist Dockerfile (
                                    echo Creating backend Dockerfile
                                    (
                                        echo FROM node:14-alpine
                                        echo WORKDIR /app
                                        echo COPY package*.json ./
                                        echo RUN npm install
                                        echo COPY . .
                                        echo EXPOSE 5000
                                        echo CMD ["node", "app.js"]
                                    ) > Dockerfile
                                )
                            '''
                        }
                    }
                }
            }
        }
        stage('Build and Push Frontend Image') {
            steps {
                dir('learnerReportCS_frontend') {
                    script {
                        if (isUnix()) {
                            sh '''
                                docker build -t ${FRONTEND_IMAGE_NAME}:${BUILD_NUMBER} -t ${FRONTEND_IMAGE_NAME}:latest .
                                docker push ${FRONTEND_IMAGE_NAME}:${BUILD_NUMBER}
                                docker push ${FRONTEND_IMAGE_NAME}:latest
                            '''
                        } else {
                            bat """
                                docker build -t %FRONTEND_IMAGE_NAME%:%BUILD_NUMBER% -t %FRONTEND_IMAGE_NAME%:latest .
                                docker push %FRONTEND_IMAGE_NAME%:%BUILD_NUMBER%
                                docker push %FRONTEND_IMAGE_NAME%:latest
                            """
                        }
                    }
                }
            }
        }
        stage('Build and Push Backend Image') {
            steps {
                dir('learnerReportCS_backend') {
                    script {
                        if (isUnix()) {
                            sh '''
                                docker build -t ${BACKEND_IMAGE_NAME}:${BUILD_NUMBER} -t ${BACKEND_IMAGE_NAME}:latest .
                                docker push ${BACKEND_IMAGE_NAME}:${BUILD_NUMBER}
                                docker push ${BACKEND_IMAGE_NAME}:latest
                            '''
                        } else {
                            bat """
                                docker build -t %BACKEND_IMAGE_NAME%:%BUILD_NUMBER% -t %BACKEND_IMAGE_NAME%:latest .
                                docker push %BACKEND_IMAGE_NAME%:%BUILD_NUMBER%
                                docker push %BACKEND_IMAGE_NAME%:latest
                            """
                        }
                    }
                }
            }
        }
        stage('Deploy to Kubernetes using Helm') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig-credentials', variable: 'KUBECONFIG')]) {
                    script {
                        if (isUnix()) {
                            sh '''
                                export KUBECONFIG=$KUBECONFIG
                                
                                # Update Helm chart values with new image tag
                                sed -i "s/tag: .*/tag: ${BUILD_NUMBER}/g" learnerreport/values.yaml
                                
                                # Deploy using Helm
                                if helm ls | grep -q "learnerreport"; then
                                    helm upgrade learnerreport ./learnerreport --wait --timeout 5m0s
                                else
                                    helm install learnerreport ./learnerreport --wait --timeout 5m0s
                                fi
                                # Verify deployment
                                kubectl get pods
                                kubectl get services
                            '''
                        } else {
                            bat """
                                set KUBECONFIG=%KUBECONFIG%
                                
                                powershell -Command "(Get-Content learnerreport/values.yaml) -replace 'tag: .*', 'tag: %BUILD_NUMBER%' | Set-Content learnerreport/values.yaml"
                                
                                for /f "tokens=*" %%i in ('helm ls ^| findstr "learnerreport"') do set HELM_INSTALLED=%%i
                                if defined HELM_INSTALLED (
                                    helm upgrade learnerreport ./learnerreport --wait --timeout 5m0s
                                ) else (
                                    helm install learnerreport ./learnerreport --wait --timeout 5m0s
                                )
                                kubectl get pods
                                kubectl get services
                            """
                        }
                    }
                }
            }
        }
        stage('Wait for Deployment') {
            steps {
                script {
                    if (isUnix()) {
                        sh 'sleep 30'
                    } else {
                        bat 'ping 127.0.0.1 -n 30 > nul'
                    }
                }
            }
        }
        stage('Run Tests') {
            steps {
                echo 'Running tests...'
                script {
                    if (isUnix()) {
                        sh '''
                            # Simple health check using kubectl port-forward instead of direct access
                            kubectl port-forward svc/learnerreport-backend-service 5000:5000 &
                            PF_PID=$!
                            sleep 5
                            
                            # Backend Health Check
                            if curl -s -f http://localhost:5000 > /dev/null; then
                              echo "Backend is healthy"
                            else
                              echo "Backend health check failed"
                              kill $PF_PID
                              exit 1
                            fi
                            
                            kill $PF_PID
                            
                            # Basic frontend service check
                            if kubectl get svc learnerreport-frontend-service; then
                              echo "Frontend service exists"
                            else
                              echo "Frontend service not found"
                              exit 1
                            fi
                            
                            echo "Tests passed!"
                        '''
                    } else {
                        bat '''
                            @echo off
                            
                            REM Simple service existence check
                            kubectl get svc learnerreport-backend-service || exit /b 1
                            echo Backend service exists
                            
                            kubectl get svc learnerreport-frontend-service || exit /b 1
                            echo Frontend service exists
                            
                            echo Tests passed!
                        '''
                    }
                }
            }
        }
        stage('Cleanup') {
            steps {
                script {
                    if (isUnix()) {
                        sh '''
                            docker logout
                            docker system prune -af --volumes
                        '''
                    } else {
                        bat '''
                            docker logout
                            docker system prune -af --volumes
                        '''
                    }
                }
                cleanWs()
            }
        }
    }
    post {
        success {
            echo '✅ Pipeline completed successfully!'
        }
        failure {
            echo '❌ Pipeline failed! Check logs for errors.'
            script {
                if (isUnix()) {
                    sh 'kubectl get pods -o wide'
                    sh 'kubectl describe pods -l app=learnerreport'
                } else {
                    bat 'kubectl get pods -o wide'
                    bat 'kubectl describe pods -l app=learnerreport'
                }
            }
        }
    }
}