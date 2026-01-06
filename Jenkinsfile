pipeline {
    agent none

    environment {
        // Credentials Jenkins
        DOCKERHUB_CREDS = credentials('DOCKERHUB-CREDS')  // ID de ton credential DockerHub
        SLACK_TOKEN = credentials('slack-webhook')         // ID de ton credential Slack

        // Variables du projet
        IMAGE_NAME = "paymybuddy"
        IMAGE_TAG = "v1"
        PORT_EXPOSED = "80"

        // Hôtes de déploiement
        HOSTNAME_DEPLOY_STAGING = "54.175.193.5"
        HOSTNAME_DEPLOY_PROD = "54.226.252.171"
    }

    stages {

        stage('Checkout source') {
    steps {
        // Repo Jenkins (celui avec Jenkinsfile)
        checkout scm

        // Repo application
        dir('paymybuddy') {
            git branch: 'main',
                url: 'https://github.com/yannick0405/PayMyBuddy.git'
        }
    }
}

       stage('Build JAR with Maven Wrapper') {
    agent any
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
            agent any
            steps {
                script {
                    sh """
                        docker build -t ${DOCKERHUB_CREDS_USR}/${IMAGE_NAME}:${IMAGE_TAG} .
                    """
                }
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
                script {
                    sh """
                        echo ${DOCKERHUB_CREDS_PSW} | docker login -u ${DOCKERHUB_CREDS_USR} --password-stdin
                        docker push ${DOCKERHUB_CREDS_USR}/${IMAGE_NAME}:${IMAGE_TAG}
                    """
                }
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
        echo "Build failed"
    }
}
