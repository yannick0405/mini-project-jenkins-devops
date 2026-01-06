pipeline {
    agent any

    environment {
        DOCKERHUB_CREDS = credentials('dockerhub-creds')  // tes credentials DockerHub
        SLACK_TOKEN = credentials('slack-token')          // ton token Slack
        IMAGE_NAME = "yannick0405/paymybuddy"
        IMAGE_TAG = "v1"
        HOST_PORT = "8081"  // port de l'hôte pour éviter conflit avec Jenkins sur 8080
        CONTAINER_PORT = "8080"
    }

    stages {

        stage('Checkout Source') {
            steps {
                git branch: 'main', url: 'https://github.com/yannick0405/mini-project-jenkins-devops.git'
            }
        }

        stage('Build JAR with Maven') {
            agent {
                docker {
                    image 'maven:3.9.1-eclipse-temurin-17'
                    args '-v $HOME/.m2:/root/.m2'
                }
            }
            steps {
                dir('paymybuddy') {
                    sh './mvnw clean package -DskipTests'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                dir('paymybuddy') {
                    sh """
                        docker build -t $IMAGE_NAME:$IMAGE_TAG .
                    """
                }
            }
        }

        stage('Run Container Locally for Test') {
            steps {
                script {
                    try {
                        sh "docker rm -f paymybuddy || true"
                        sh "docker run -d --name paymybuddy -p $HOST_PORT:$CONTAINER_PORT $IMAGE_NAME:$IMAGE_TAG"
                        sleep 5
                        sh "curl -f http://localhost:$HOST_PORT || (echo 'App test failed' && exit 1)"
                    } finally {
                        sh "docker rm -f paymybuddy || true"
                    }
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    sh "echo $DOCKERHUB_CREDS_PSW | docker login -u $DOCKERHUB_CREDS --password-stdin"
                    sh "docker push $IMAGE_NAME:$IMAGE_TAG"
                }
            }
        }

        stage('Deploy to Staging') {
            steps {
                echo "Deployment to staging would go here"
            }
        }

        stage('Test Staging') {
            steps {
                echo "Staging tests would go here"
            }
        }

        stage('Deploy to Production') {
            steps {
                echo "Deployment to production would go here"
            }
        }

        stage('Test Production') {
            steps {
                echo "Production tests would go here"
            }
        }
    }

    post {
        success {
            slackSend(channel: '#jenkins-notification2025', color: 'good', tokenCredentialId: 'slack-token', message: "Pipeline SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}")
        }
        failure {
            slackSend(channel: '#jenkins-notification2025', color: 'danger', tokenCredentialId: 'slack-token', message: "Pipeline FAILURE: ${env.JOB_NAME} #${env.BUILD_NUMBER}")
        }
    }
}
