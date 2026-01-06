pipeline {
    agent none

    environment {
        DOCKERHUB_CREDS = credentials('DOCKERHUB-CREDS')  // DockerHub
        SLACK_TOKEN = credentials('slack-token')          // Slack webhook
        IMAGE_NAME = "paymybuddy"
        IMAGE_TAG = "v1"
        PORT_EXPOSED = "80"
        HOSTNAME_DEPLOY_STAGING = "54.175.193.5"
        HOSTNAME_DEPLOY_PROD = "54.226.252.171"
    }

    stages {

        stage('Checkout source') {
            agent any
            steps {
                // Repo Jenkinsfile
                checkout scm

                // Repo application
                dir('paymybuddy') {
                    git branch: 'main', url: 'https://github.com/yannick0405/PayMyBuddy.git'
                }
            }
        }

        stage('Build JAR with Maven Docker') {
            agent {
                docker { image 'maven:3.9.1-eclipse-temurin-17' }  // Maven + JDK 17
            }
            steps {
                dir('paymybuddy') {
                    sh '''
                        chmod +x mvnw || true
                        ./mvnw clean package -DskipTests || mvn clean package -DskipTests
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            agent any
            steps {
                sh """
                    docker build -t ${DOCKERHUB_CREDS_USR}/${IMAGE_NAME}:${IMAGE_TAG} paymybuddy
                """
            }
        }

        stage('Run Container Locally for Test') {
            agent any
            steps {
                sh """
                    docker rm -f ${IMAGE_NAME} || true
                    docker run -d --name ${IMAGE_NAME} -p ${PORT_EXPOSED}:80 ${DOCKERHUB_CREDS_USR}/${IMAGE_NAME}:${IMAGE_TAG}
                    sleep 5
                    curl -f http://localhost:${PORT_EXPOSED} || (echo "App test failed" && exit 1)
                    docker stop ${IMAGE_NAME}
                    docker rm ${IMAGE_NAME}
                """
            }
        }

        stage('Push Docker Image') {
            agent any
            steps {
                sh """
                    echo ${DOCKERHUB_CREDS_PSW} | docker login -u ${DOCKERHUB_CREDS_USR} --password-stdin
                    docker push ${DOCKERHUB_CREDS_USR}/${IMAGE_NAME}:${IMAGE_TAG}
                """
            }
        }

        stage('Deploy to Staging') {
            agent any
            steps {
                sshagent(credentials: ['SSH_AUTH_SERVER']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ubuntu@${HOSTNAME_DEPLOY_STAGING} '
                            docker login -u ${DOCKERHUB_CREDS_USR} -p ${DOCKERHUB_CREDS_PSW} &&
                            docker pull ${DOCKERHUB_CREDS_USR}/${IMAGE_NAME}:${IMAGE_TAG} &&
                            docker rm -f ${IMAGE_NAME} || true &&
                            docker run -d -p 80:80 --name ${IMAGE_NAME} ${DOCKERHUB_CREDS_USR}/${IMAGE_NAME}:${IMAGE_TAG}
                        '
                    """
                }
            }
        }

        stage('Test Staging') {
            agent any
            steps {
                sh """
                    curl -f http://${HOSTNAME_DEPLOY_STAGING} || (echo "Staging test failed" && exit 1)
                """
            }
        }

        stage('Deploy to Production') {
            agent any
            steps {
                sshagent(credentials: ['SSH_AUTH_SERVER']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ubuntu@${HOSTNAME_DEPLOY_PROD} '
                            docker login -u ${DOCKERHUB_CREDS_USR} -p ${DOCKERHUB_CREDS_PSW} &&
                            docker pull ${DOCKERHUB_CREDS_USR}/${IMAGE_NAME}:${IMAGE_TAG} &&
                            docker rm -f ${IMAGE_NAME} || true &&
                            docker run -d -p 80:80 --name ${IMAGE_NAME} ${DOCKERHUB_CREDS_USR}/${IMAGE_NAME}:${IMAGE_TAG}
                        '
                    """
                }
            }
        }

        stage('Test Production') {
            agent any
            steps {
                sh """
                    curl -f http://${HOSTNAME_DEPLOY_PROD} || (echo "Production test failed" && exit 1)
                """
            }
        }
    }

    post {
        failure {
            slackSend(
                channel: '#jenkins-notification2025',
                color: 'danger',
                tokenCredentialId: 'slack-token',
                message: "Pipeline FAILED : ${env.JOB_NAME} Build #${env.BUILD_NUMBER}"
            )
        }
        success {
            slackSend(
                channel: '#jenkins-notification2025',
                color: 'good',
                tokenCredentialId: 'slack-token',
                message: "Pipeline SUCCESS : ${env.JOB_NAME} Build #${env.BUILD_NUMBER}"
            )
        }
    }
}
