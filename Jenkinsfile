pipeline {
    agent none

    environment {
        // DockerHub
        DOCKERHUB_CREDS = credentials('DOCKERHUB-CREDS')
        

        IMAGE_NAME = "paymybuddy"
        IMAGE_TAG  = "${env.BUILD_NUMBER}"

        // SonarCloud
        SONAR_TOKEN = credentials('slack-webhook')
        SONAR_ORG   = "yannick0405"
        SONAR_PROJECT_KEY = "yannick0405_PayMyBuddy"

        // Deployment
        STAGING_HOST = "44.211.85.128"
        PROD_HOST    = "98.92.95.208"

        APP_PORT = "8080"
    }

    stages {

        stage('Checkout') {
            agent any
            steps {
                git branch: 'main',
                    url: 'https://github.com/yannick0405/PayMyBuddy.git'
            }
        }

        stage('Tests') {
            agent {
                docker {
                    image 'maven:3.9.6-eclipse-temurin-17'
                }
            }
            steps {
                sh 'mvn clean test'
            }
        }

        stage('SonarCloud Analysis') {
            agent {
                docker {
                    image 'maven:3.9.6-eclipse-temurin-17'
                }
            }
            steps {
                sh """
                  mvn sonar:sonar \
                  -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                  -Dsonar.organization=${SONAR_ORG} \
                  -Dsonar.host.url=https://sonarcloud.io \
                  -Dsonar.login=${SONAR_TOKEN}
                """
            }
        }

        stage('Build Docker Image') {
            agent any
            steps {
                sh """
                  docker build -t ${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG} .
                """
            }
        }

        stage('Push Docker Image') {
            agent any
            steps {
                sh """
                  echo ${DOCKER_PASS} | docker login -u ${DOCKER_USER} --password-stdin
                  docker push ${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG}
                """
            }
        }

        stage('Deploy Staging') {
            when {
                branch 'main'
            }
            agent any
            steps {
                sshagent(credentials: ['SSH_AUTH_SERVER']) {
                    sh """
                      ssh -o StrictHostKeyChecking=no ubuntu@${STAGING_HOST} '
                      docker login -u ${DOCKER_USER} -p ${DOCKER_PASS} &&
                      docker pull ${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG} &&
                      docker rm -f paymybuddy || true &&
                      docker run -d -p ${APP_PORT}:8080 --name paymybuddy ${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG}
                      '
                    """
                }
            }
        }

        stage('Test Staging') {
            when {
                branch 'main'
            }
            agent any
            steps {
                sh "curl http://${STAGING_HOST}:${APP_PORT}/actuator/health"
            }
        }

        stage('Deploy Production') {
            when {
                branch 'main'
            }
            agent any
            steps {
                sshagent(credentials: ['SSH_AUTH_SERVER']) {
                    sh """
                      ssh -o StrictHostKeyChecking=no ubuntu@${PROD_HOST} '
                      docker login -u ${DOCKER_USER} -p ${DOCKER_PASS} &&
                      docker pull ${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG} &&
                      docker rm -f paymybuddy || true &&
                      docker run -d -p ${APP_PORT}:8080 --name paymybuddy ${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG}
                      '
                    """
                }
            }
        }
    }

    post {
        success {
            slackSend(channel: '#jenkins',
                      message: "✅ PayMyBuddy pipeline SUCCESS – Build ${BUILD_NUMBER}")
        }
        failure {
            slackSend(channel: '#jenkins',
                      message: "❌ PayMyBuddy pipeline FAILED – Build ${BUILD_NUMBER}")
        }
    }
}
