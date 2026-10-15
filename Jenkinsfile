pipeline {
    agent any

    environment {
        DOCKER_HUB_USER = 'tanshul12'
        IMAGE_NAME      = 'event-management'
        IMAGE_TAG       = "${BUILD_NUMBER}"
        KUBECONFIG_CRED = 'kuberconfig'
        DOCKER_CRED     = 'dockerhub'
        DOCKER_BIN      = 'C:\\Users\\tanshul\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin'
    }

    stages {
        stage('1. Clone Code') {
            steps {
                checkout scm
            }
        }

        stage('2. Build Docker Image') {
            steps {
                bat "\"${DOCKER_BIN}\\docker.exe\" build -t %DOCKER_HUB_USER%/%IMAGE_NAME%:%IMAGE_TAG% ."
                bat "\"${DOCKER_BIN}\\docker.exe\" tag %DOCKER_HUB_USER%/%IMAGE_NAME%:%IMAGE_TAG% %DOCKER_HUB_USER%/%IMAGE_NAME%:latest"
            }
        }

        stage('3. Test Application') {
            steps {
                bat """
                    "${DOCKER_BIN}\\docker.exe" run -d --name test-event-temp -p 8087:80 %DOCKER_HUB_USER%/%IMAGE_NAME%:%IMAGE_TAG%
                    ping -n 4 127.0.0.1 >nul
                    curl.exe -s -f http://localhost:8087 >nul || ("${DOCKER_BIN}\\docker.exe" rm -f test-event-temp && exit 1)
                    "${DOCKER_BIN}\\docker.exe" rm -f test-event-temp
                """
            }
        }

        stage('4. Push Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: DOCKER_CRED, usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
                    bat "\"${DOCKER_BIN}\\docker.exe\" login -u %DH_USER% -p %DH_PASS%"
                    bat "\"${DOCKER_BIN}\\docker.exe\" push %DOCKER_HUB_USER%/%IMAGE_NAME%:%IMAGE_TAG%"
                    bat "\"${DOCKER_BIN}\\docker.exe\" push %DOCKER_HUB_USER%/%IMAGE_NAME%:latest"
                }
            }
        }

        stage('5. Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: KUBECONFIG_CRED, variable: 'KUBECONFIG_FILE')]) {
                    script {
                        bat "\"${DOCKER_BIN}\\kubectl.exe\" --kubeconfig=\"%KUBECONFIG_FILE%\" apply -f deployment.yaml"
                        bat "\"${DOCKER_BIN}\\kubectl.exe\" --kubeconfig=\"%KUBECONFIG_FILE%\" set image deployment/event-portal-deployment event-portal=%DOCKER_HUB_USER%/%IMAGE_NAME%:%IMAGE_TAG%"

                        def status = bat(
                            script: "\"${DOCKER_BIN}\\kubectl.exe\" --kubeconfig=\"%KUBECONFIG_FILE%\" rollout status deployment/event-portal-deployment --timeout=60s",
                            returnStatus: true
                        )

                        if (status != 0) {
                            echo "Fault detected! Rollback initiated..."
                            bat "\"${DOCKER_BIN}\\kubectl.exe\" --kubeconfig=\"%KUBECONFIG_FILE%\" rollout undo deployment/event-portal-deployment"
                            bat "\"${DOCKER_BIN}\\kubectl.exe\" --kubeconfig=\"%KUBECONFIG_FILE%\" rollout status deployment/event-portal-deployment"
                            error("Failed deployment rolled back to stable release.")
                        } else {
                            echo "Event Portal deployed successfully across 3 replicas!"
                        }
                    }
                }
            }
        }
    }
}
