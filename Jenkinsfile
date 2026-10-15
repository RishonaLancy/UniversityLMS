pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        timestamps()
    }

    environment {
	PATH+DOCKER = "C:\\Users\\risho\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin;${env.PATH}"

        IMAGE = 'rishonalancy/university-lms'
        DEPLOYMENT = 'lms-portal'
        CONTAINER = 'lms-portal'
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
                bat 'git log -1 --oneline'
            }
        }

        stage('Build Docker Image') {
            steps {
                bat 'docker build -t %IMAGE%:%BUILD_NUMBER% .'
            }
        }

        stage('Test Docker Image') {
            steps {
                bat 'docker run --rm %IMAGE%:%BUILD_NUMBER% nginx -t'
                bat 'docker image inspect %IMAGE%:%BUILD_NUMBER%'
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKERHUB_USER',
                        passwordVariable: 'DOCKERHUB_TOKEN'
                    )
                ]) {
                    bat '''
                    powershell -NoProfile -Command "$env:DOCKERHUB_TOKEN | docker login -u $env:DOCKERHUB_USER --password-stdin"
                    docker push %IMAGE%:%BUILD_NUMBER%
                    docker logout
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubernetes-config',
                        variable: 'KUBECONFIG'
                    )
                ]) {
                    script {
                        bat 'kubectl apply -f deployment.yaml'

                        bat "kubectl set image deployment/%DEPLOYMENT% %CONTAINER%=%IMAGE%:%BUILD_NUMBER%"

                        def rolloutResult = bat(
                            returnStatus: true,
                            script: 'kubectl rollout status deployment/lms-portal --timeout=90s'
                        )

                        if (rolloutResult != 0) {
                            echo 'ROLLOUT FAILED. Restoring previous release.'

                            bat 'kubectl rollout undo deployment/lms-portal'

                            bat 'kubectl rollout status deployment/lms-portal --timeout=90s'

                            bat 'kubectl get pods -l app=lms-portal'

                            error('New release failed. Automatic rollback completed.')
                        }

                        bat 'kubectl get pods -l app=lms-portal'
                        bat 'kubectl get deployment lms-portal'
                    }
                }
            }
        }

        stage('Verify Service') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubernetes-config',
                        variable: 'KUBECONFIG'
                    )
                ]) {
                    bat 'kubectl get service lms-portal-service'
                    bat 'kubectl get pods -l app=lms-portal'
                }
            }
        }
    }

    post {
        success {
            echo 'Release deployed successfully.'
        }

        failure {
            echo 'Pipeline failed. Check the stage logs.'
        }

        always {
            echo 'University LMS pipeline finished.'
        }
    }
}
