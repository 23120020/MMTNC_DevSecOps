pipeline {
    agent any

    tools {
        sonarScanner 'SonarScanner' 
    }

    environment {
        IMAGE_NAME = "mmtnc-app"
        CONTAINER_NAME = "mmtnc-app-container"
        
        HOST_IP = "172.17.0.1" 
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

                withSonarQubeEnv('SonarQubeServer') { 
                    sh "sonar-scanner"
                }
            }
        }

        stage('3. Build & Deploy') {
            steps {
                echo 'Build Docker Image...'
                sh "docker build -t ${IMAGE_NAME} ."
                
                echo 'Dọn dẹp container cũ...'
                sh "docker rm -f ${CONTAINER_NAME} || true"

                sh "docker run -d --name ${CONTAINER_NAME} -p 3000:80 ${IMAGE_NAME}"
                
                sleep time: 5, unit: 'SECONDS'
            }
        }

        stage('4. DAST (ZAP Stable)') {
            steps {
                echo 'Quét lỗ hổng động với ZAP Stable...'
                sh """
                docker run --rm -v \$(pwd):/zap/wrk/:rw -t zaproxy/zap-stable zap-baseline.py \
                -t http://${HOST_IP}:3000 -r zap_report.html || true
                """
            }
        }
    }

    post {
        always {
            echo 'Lưu báo cáo và dọn dẹp...'
            archiveArtifacts artifacts: 'zap_report.html', allowEmptyArchive: true
            sh "docker rm -f ${CONTAINER_NAME} || true"
        }
    }
}
