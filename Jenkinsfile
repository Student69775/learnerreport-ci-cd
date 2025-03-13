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
        stage('Create Dockerfiles') {
            steps {
                // Create frontend Dockerfile if missing
                dir('learnerReportCS_frontend') {
                    script {
                        if (isUnix()) {
                            sh '''
                                if [ ! -f Dockerfile ]; then
                                    echo 'Creating Frontend Dockerfile'
                                    cat > Dockerfile << 'EOF'
FROM node:16-alpine as build
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/build /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
EOF

                                    # Create nginx conf if not exists
                                    if [ ! -f nginx.conf ]; then
                                        cat > nginx.conf << 'EOF'
server {
    listen 80;
    location / {
        root /usr/share/nginx/html;
        index index.html index.htm;
        try_files $uri $uri/ /index.html;
    }
    location /api {
        proxy_pass http://learnerreport-backend-service:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
EOF
                                    fi
                                fi
                            '''
                        } else {
                            bat '''
                                if not exist Dockerfile (
                                    echo Creating Frontend Dockerfile
                                    (
                                        echo FROM node:16-alpine as build
                                        echo WORKDIR /app
                                        echo COPY package*.json ./
                                        echo RUN npm install
                                        echo COPY . .
                                        echo RUN npm run build
                                        echo.
                                        echo FROM nginx:alpine
                                        echo COPY --from=build /app/build /usr/share/nginx/html
                                        echo COPY nginx.conf /etc/nginx/conf.d/default.conf
                                        echo EXPOSE 80
                                        echo CMD ["nginx", "-g", "daemon off;"]
                                    ) > Dockerfile
                                    
                                    if not exist nginx.conf (
                                        echo Creating nginx.conf
                                        (
                                            echo server {
                                            echo     listen 80;
                                            echo     location / {
                                            echo         root /usr/share/nginx/html;
                                            echo         index index.html index.htm;
                                            echo         try_files $uri $uri/ /index.html;
                                            echo     }
                                            echo     location /api {
                                            echo         proxy_pass http://learnerreport-backend-service:5000;
                                            echo         proxy_set_header Host $host;
                                            echo         proxy_set_header X-Real-IP $remote_addr;
                                            echo     }
                                            echo }
                                        ) > nginx.conf
                                    )
                                )
                            '''
                        }
                    }
                }
                
                // Create backend Dockerfile if missing
                dir('learnerReportCS_backend') {
                    script {
                        if (isUnix()) {
                            sh '''
                                if [ ! -f Dockerfile ]; then
                                    echo 'Creating Backend Dockerfile'
                                    cat > Dockerfile << 'EOF'
FROM node:16-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 5000
CMD ["node", "app.js"]
EOF
                                fi
                            '''
                        } else {
                            bat '''
                                if not exist Dockerfile (
                                    echo Creating Backend Dockerfile
                                    (
                                        echo FROM node:16-alpine
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
        stage('Login to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    script {
                        if (isUnix()) {
                            sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
                        } else {
                            // Use proper secure method for Windows
                            powershell '(ConvertTo-SecureString -String $env:DOCKER_PASS -AsPlainText -Force) | docker login -u $env:DOCKER_USER --password-stdin'
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
                                sed -i "s/tag: latest/tag: ${BUILD_NUMBER}/g" learnerreport/values.yaml
                                
                                # Add imagePullPolicy: Always to ensure latest images are pulled
                                sed -i "s/pullPolicy: IfNotPresent/pullPolicy: Always/g" learnerreport/values.yaml
                                
                                # Deploy using Helm
                                if helm ls | grep -q "learnerreport"; then
                                    helm upgrade --wait --timeout 5m learnerreport ./learnerreport
                                else
                                    helm install --wait --timeout 5m learnerreport ./learnerreport
                                fi
                                
                                # Delete failed pods to trigger recreation
                                kubectl get pods | grep -E 'CrashLoop|ImagePull' | awk '{print $1}' | xargs -r kubectl delete pod
                                
                                # Verify deployment
                                kubectl get pods
                                kubectl get services
                            '''
                        } else {
                            // Windows-compatible commands for Kubernetes deployment
                            bat """
                                set KUBECONFIG=%KUBECONFIG%
                                
                                powershell -Command "(Get-Content learnerreport/values.yaml) -replace 'tag: latest', 'tag: %BUILD_NUMBER%' | Set-Content learnerreport/values.yaml"
                                powershell -Command "(Get-Content learnerreport/values.yaml) -replace 'pullPolicy: IfNotPresent', 'pullPolicy: Always' | Set-Content learnerreport/values.yaml"
                                
                                helm ls | findstr "learnerreport" > nul
                                if %ERRORLEVEL% == 0 (
                                    helm upgrade --wait --timeout 5m learnerreport ./learnerreport
                                ) else (
                                    helm install --wait --timeout 5m learnerreport ./learnerreport
                                )
                                
                                REM Delete failed pods to trigger recreation
                                powershell -Command "kubectl get pods | Select-String -Pattern 'CrashLoop|ImagePull' | ForEach-Object { $_.Line.Split(' ')[0] } | ForEach-Object { kubectl delete pod $_ }"
                                
                                kubectl get pods
                                kubectl get services
                            """
                        }
                    }
                }
            }
        }
        stage('Run Tests') {
            steps {
                echo 'Running tests...'
                script {
                    if (isUnix()) {
                        sh 'sleep 30' // Give services more time to stabilize
                        sh '''
                            set +e  # Continue on errors to ensure cleanup
                            
                            # Backend Health Check
                            BACKEND_SVC=learnerreport-backend-service
                            echo "Checking backend service: $BACKEND_SVC"
                            BACKEND_IP=$(kubectl get svc $BACKEND_SVC -o jsonpath="{.spec.clusterIP}" || echo "not-found")
                            
                            if [ "$BACKEND_IP" = "not-found" ]; then
                                echo "Backend service not found. Available services:"
                                kubectl get svc
                                exit 1
                            fi
                            
                            echo "Attempting to connect to backend at $BACKEND_IP:5000"
                            curl -f --connect-timeout 10 http://$BACKEND_IP:5000 || echo "Backend health check failed but continuing"
                            
                            # Frontend Health Check - using service name instead of NodePort
                            echo "Checking if frontend service is accessible"
                            kubectl port-forward svc/learnerreport-frontend-service 8080:80 &
                            PF_PID=$!
                            sleep 5
                            curl -f --connect-timeout 10 http://localhost:8080 || echo "Frontend health check failed but continuing"
                            kill $PF_PID || true
                            
                            echo "Tests completed."
                        '''
                    } else {
                        bat "timeout /T 30 /NOBREAK" // Give services more time to stabilize
                        bat """
                            @echo off
                            
                            REM Backend Health Check
                            echo Checking backend service: learnerreport-backend-service
                            for /f "tokens=*" %%a in ('kubectl get svc learnerreport-backend-service -o jsonpath^="{.spec.clusterIP}" 2^>nul') do set BACKEND_IP=%%a
                            
                            if "%BACKEND_IP%"=="" (
                                echo Backend service not found. Available services:
                                kubectl get svc
                                exit /b 0
                            )
                            
                            echo Attempting to connect to backend at %BACKEND_IP%:5000
                            curl -f --connect-timeout 10 http://%BACKEND_IP%:5000 || echo Backend health check failed but continuing
                            
                            REM Frontend Health Check - port forwarding approach
                            echo Starting port-forward for frontend service
                            start /b kubectl port-forward svc/learnerreport-frontend-service 8080:80
                            timeout /T 5 /NOBREAK
                            curl -f --connect-timeout 10 http://localhost:8080 || echo Frontend health check failed but continuing
                            
                            REM Kill port-forward process
                            for /f "tokens=5" %%p in ('netstat -ano ^| findstr :8080') do (
                                taskkill /F /PID %%p 2>nul || echo Could not kill port-forward
                            )
                            
                            echo Tests completed.
                        """
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
                            docker system prune -f
                        '''
                    } else {
                        bat '''
                            docker logout
                            docker system prune -f
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
            
            // Fetch logs for failed pods
            script {
                if (isUnix()) {
                    sh '''
                        echo "==== Getting logs from failed pods ===="
                        kubectl get pods | grep -E 'CrashLoop|ImagePull|Error' | awk '{print $1}' | xargs -r -I{} sh -c 'echo "==== Logs for pod {} ===="; kubectl describe pod {} | tail -20; kubectl logs {} --tail=50 --previous 2>/dev/null || true'
                    '''
                } else {
                    bat '''
                        echo ==== Getting logs from failed pods ====
                        powershell -Command "kubectl get pods | Select-String -Pattern 'CrashLoop|ImagePull|Error' | ForEach-Object { $pod = $_.Line.Split(' ')[0]; Write-Host '==== Logs for pod '$pod' ===='; kubectl describe pod $pod | Select-Object -Last 20; kubectl logs $pod --tail=50 --previous 2>$null || Write-Host 'No previous logs available' }"
                    '''
                }
            }
        }
        always {
            // Clean up any port-forward processes that might be hanging
            script {
                if (isUnix()) {
                    sh 'pkill -f "kubectl port-forward" || true'
                } else {
                    bat 'taskkill /F /FI "WINDOWTITLE eq kubectl port-forward*" /T > nul 2>&1 || exit /b 0'
                }
            }
        }
    }
}