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
                            bat "docker login -u %DOCKER_USER% -p %DOCKER_PASS%"
                        }
                    }
                }
            }
        }
        stage('Build and Push Frontend Image') {
            steps {
                dir('learnerReportCS_frontend') {
                    script {
                        if (fileExists('Dockerfile')) {
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
                        } else {
                            error "Dockerfile not found in learnerReportCS_frontend"
                        }
                    }
                }
            }
        }
        stage('Build and Push Backend Image') {
            steps {
                dir('learnerReportCS_backend') {
                    script {
                        if (fileExists('Dockerfile')) {
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
                        } else {
                            error "Dockerfile not found in learnerReportCS_backend"
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
                                sed -i "s/tag: latest/tag: ${BUILD_NUMBER}/g" learnerreport/values.yaml
                                if helm ls | grep -q "learnerreport"; then
                                    helm upgrade learnerreport ./learnerreport
                                else
                                    helm install learnerreport ./learnerreport
                                fi
                                kubectl get pods
                                kubectl get services
                            '''
                        } else {
                            bat """
                                set KUBECONFIG=%KUBECONFIG%
                                powershell -Command "(Get-Content learnerreport/values.yaml) -replace 'tag: latest', 'tag: %BUILD_NUMBER%' | Set-Content learnerreport/values.yaml"
                                helm ls | findstr "learnerreport" > nul
                                if %ERRORLEVEL% == 0 (
                                    helm upgrade learnerreport ./learnerreport
                                ) else (
                                    helm install learnerreport ./learnerreport
                                )
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
                        sh 'sleep 10' // Wait for services to be ready
                        sh '''
                            set -e
                            BACKEND_URL=$(kubectl get svc learnerreport-backend-service -o jsonpath="{.spec.clusterIP}"):5000
                            curl -f http://$BACKEND_URL || exit 1
                            FRONTEND_PORT=$(kubectl get svc learnerreport-frontend-service -o jsonpath="{.spec.ports[0].nodePort}")
                            NODE_IP=$(kubectl get nodes -o jsonpath="{.items[0].status.addresses[0].address}")
                            curl -f http://$NODE_IP:$FRONTEND_PORT || exit 1
                            echo "Tests passed!"
                        '''
                    } else {
                        bat "timeout /t 10"
                        bat """
                            @echo off
                            for /f "tokens=*" %%a in ('kubectl get svc learnerreport-backend-service -o jsonpath^="{.spec.clusterIP}"') do set BACKEND_IP=%%a
                            curl -f http://%BACKEND_IP%:5000
                            if %ERRORLEVEL% neq 0 exit /b 1
                            for /f "tokens=*" %%a in ('kubectl get svc learnerreport-frontend-service -o jsonpath^="{.spec.ports[0].nodePort}"') do set FRONTEND_PORT=%%a
                            for /f "tokens=*" %%a in ('kubectl get nodes -o jsonpath^="{.items[0].status.addresses[0].address}"') do set NODE_IP=%%a
                            curl -f http://%NODE_IP%:%FRONTEND_PORT%
                            if %ERRORLEVEL% neq 0 exit /b 1
                            echo Tests passed!
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
        }
    }
}