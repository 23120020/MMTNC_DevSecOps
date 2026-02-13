pipeline {
    agent any

    environment {
        IMAGE_NAME = "mmtnc-app"
        CONTAINER_NAME = "mmtnc-app-container"
        HOST_IP = "172.17.0.1" 
        DOCKERHUB_USER = "sybew"
        DOCKERHUB_REPO = "mmtnc-app" 
    }

    stages {
        stage('1. Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('2. SAST (SonarQube)') {
            steps {
                echo 'Quét mã nguồn HTML/JS...'
                script {
                    def scannerHome = tool 'sonar-scanner'
                    withSonarQubeEnv('SonarQubeServer') { 
                        sh "${scannerHome}/bin/sonar-scanner"
                    }
                }
            }
        }

        stage('3. Build & Deploy') {
            steps {
                echo 'Build Docker Image...'
                sh "docker build -t ${IMAGE_NAME} ."
                
                echo 'Dọn dẹp container cũ...'
                sh "docker rm -f ${CONTAINER_NAME} || true"

                sh "docker run -d --name ${CONTAINER_NAME} -p 3000:3000 ${IMAGE_NAME}"
            }
        }

        stage('4. DAST (ZAP Stable)') {
            steps {
                echo 'Quét lỗ hổng động với ZAP Stable...'
                script {
                    sh "docker rm -f zap-scan-container || true"

                    sh """
                    docker run -d --name zap-scan-container -u 0 -t zaproxy/zap-stable \
                    /bin/bash -c "mkdir -p /zap/wrk && zap-baseline.py -t http://${HOST_IP}:3000 -r zap_report.html"
                    """

                    sh "docker wait zap-scan-container || true"
                    
                    sh "docker logs zap-scan-container"
                    
                    sh "docker exec zap-scan-container ls -R /zap/wrk || true"

                    sh "docker cp zap-scan-container:/zap/wrk/zap_report.html ./zap_report.html"
                    
                    sh "docker rm -f zap-scan-container || true"
                }
            }
        }

        stage('5. Push to DockerHub') {
            steps {
                echo "Đang đẩy image phiên bản v${env.BUILD_NUMBER} lên DockerHub..."
                script { 
                    def exactVersion = env.BUILD_NUMBER.toInteger() - 1
                
                    if (exactVersion < 1) {
                        exactVersion = 1
                    }
                    withCredentials([usernamePassword(credentialsId: 'jenkins-dockerhub', passwordVariable: 'DOCKER_PASS', usernameVariable: 'DOCKER_USER')]) {
                        sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
                        
                        sh "docker tag ${IMAGE_NAME} ${DOCKERHUB_USER}/${DOCKERHUB_REPO}:v${exactVersion}"
                        
                        sh "docker tag ${IMAGE_NAME} ${DOCKERHUB_USER}/${DOCKERHUB_REPO}:latest"
                        
                        sh "docker push ${DOCKERHUB_USER}/${DOCKERHUB_REPO}:v${exactVersion}"
                        sh "docker push ${DOCKERHUB_USER}/${DOCKERHUB_REPO}:latest"

                        sh "docker rmi ${DOCKERHUB_USER}/${DOCKERHUB_REPO}:v${exactVersion} || true"
                        sh "docker rmi ${DOCKERHUB_USER}/${DOCKERHUB_REPO}:latest || true"
                    }
                }
            }
        }
    }

    post {
        always {
            echo 'Lưu báo cáo và dọn dẹp...'
            archiveArtifacts artifacts: 'zap_report.html', allowEmptyArchive: true
            sh "docker rm -f ${CONTAINER_NAME} || true"
            sh "docker rmi ${IMAGE_NAME} || true"
        }
    }
}
