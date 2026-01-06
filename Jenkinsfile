pipeline {
    agent any

    environment {
        // Credentials
        DOCKERHUB_CREDS = credentials('DOCKERHUB-CREDS')
        SLACK_TOKEN     = credentials('slack-webhook')

        // Image Docker
        IMAGE_NAME = "paymybuddy"
        IMAGE_TAG  = "v1"
        PORT_EXPOSED = "80"

        // Serveurs
        HOSTNAME_DEPLOY_STAGING = "54.175.193.5"
        HOSTNAME_DEPLOY_PROD    = "54.226.252.171"
    }

    stages {

        stage('Checkout sources') {
            steps {
                // Repo Jenkinsfile
                checkout scm

                // Repo application
                dir('paymybuddy') {
                    git branch: 'main',
                        url: 'https://github.com/yannick0405/PayMyBuddy.git'
                }
            }
        }

        stage('Build JAR with Maven Wrapper') {
            steps {
                dir('paymybuddy') {
                    sh '''
                        chmod +x mvnw
                        ./mvnw clean package -DskipTests
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                dir('paymybuddy') {
                    sh '''
                        docker build -t ${DOCKERHUB_CREDS_USR}/${IMAGE_NAME}:${IMAGE_TAG} .
                    '''
                }
            }
        }

        stage('Run Container Locally for Test') {
            steps {
                sh '''
                    docker rm -f ${IMAGE_NAME} || true
                    docker run -d --name ${IMAGE_NAME} -p ${PORT_EXPOSED}:80 \
                        ${DOCKERHUB_CREDS_USR}/${IMAGE_NAME}:${IMAGE_TAG}

                    sleep 5
                    curl -f http://localhost:${PORT_EXPOSED}
                    docker rm -f ${IMAGE_NAME}
                '''
            }
        }

        stage('Push Docker Image') {
            steps {
                sh '''
                    echo ${DOCKERHUB_CREDS_PSW} | docker login \
                        -u ${DOCKERHUB_CREDS_USR} --password-stdin

                    docker push ${DOCKERHUB_CREDS_USR}/${IMAGE_NAME}:${IMAGE_TAG}
                '''
            }
        }

        stage('Deploy to Staging') {
            steps {
                sshagent(credentials: ['SSH_AUTH_SERVER']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no ubuntu@${HOSTNAME_DEPLOY_STAGING} "
                            docker login -u ${DOCKERHUB_CREDS_USR} -p ${DOCKERHUB_CREDS_PSW} &&
                            docker pull ${DOCKERHUB_CREDS_USR}/${IMAGE_NAME}:${IMAGE_TAG} &&
                            docker rm -f ${IMAGE_NAME} || true &&
                            docker run -d -p 80:80 --name ${IMAGE_NAME} \
                                ${DOCKERHUB_CREDS_USR}/${IMAGE_NAME}:${IMAGE_TAG}
                        "
                    '''
                }
            }
        }

        stage('Test Staging') {
            steps {
                sh '''
                    curl -f http://${HOSTNAME_DEPLOY_STAGING}
                '''
            }
        }

        stage('Deploy to Production') {
            steps {
                sshagent(credentials: ['SSH_AUTH_SERVER']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no ubuntu@${HOSTNAME_DEPLOY_PROD} "
                            docker login -u ${DOCKERHUB_CREDS_USR} -p ${DOCKERHUB_CREDS_PSW} &&
                            docker pull ${DOCKERHUB_CREDS_USR}/${IMAGE_NAME}:${IMAGE_TAG} &&
                            docker rm -f ${IMAGE_NAME} || true &&
                            docker run -d -p 80:80 --name ${IMAGE_NAME} \
                                ${DOCKERHUB_CREDS_USR}/${IMAGE_NAME}:${IMAGE_TAG}
                        "
                    '''
                }
            }
        }

        stage('Test Production') {
            steps {
                sh '''
                    curl -f http://${HOSTNAME_DEPLOY_PROD}
                '''
            }
        }
    }

    post {
        success {
            echo "Pipeline succeeded"
        }

        failure {
            echo "Pipeline failed"
        }
    }
}
