pipeline {
    agent any 

    environment {
        DOCKERHUB_CREDS = credentials('yann')
        SLACK_TOKEN_ID = 'slack-token'
        SONAR_TOKEN = credentials('sonar-token') 
        
        IMAGE_NAME = "yannick0405/paymybuddy"
        IMAGE_TAG = "v${env.BUILD_NUMBER}"
        
        STAGING_IP = "44.211.85.128"
        PROD_IP = "98.92.95.208"
    }

    stages {
        stage('Checkout Source') {
            steps {
                deleteDir()
                // On clone le code de l'application
                git branch: 'main', url: 'https://github.com/yannick0405/PayMyBuddy.git'
            }
        }

        stage('Tests & SonarCloud') {
            // Utilisation correcte du plugin Docker Pipeline dans un bloc script
            steps {
                script {
                    docker.image('maven:3.9.1-eclipse-temurin-17').inside('-v $HOME/.m2:/root/.m2') {
                        echo "Exécution des tests et Analyse Sonar..."
                        // On utilise mvn directement (plus stable que ./mvnw en conteneur)
                        sh "mvn clean verify sonar:sonar \
                            -Dsonar.projectKey=paymybuddy-project \
                            -Dsonar.organization=yannick-org \
                            -Dsonar.host.url=https://sonarcloud.io \
                            -Dsonar.login=${SONAR_TOKEN}"
                    }
                }
            }
        }

        stage('Build & Push Docker') {
            steps {
                echo "Génération du JAR et de l'image Docker..."
                // Build du JAR via conteneur éphémère
                script {
                    docker.image('maven:3.9.1-eclipse-temurin-17').inside('-v $HOME/.m2:/root/.m2') {
                        sh "mvn package -DskipTests"
                    }
                }
                
                // Build et Push de l'image sur l'hôte
                sh """
                    docker build -t $IMAGE_NAME:$IMAGE_TAG .
                    echo $DOCKERHUB_CREDS_PSW | docker login -u $DOCKERHUB_CREDS_USR --password-stdin
                    docker push $IMAGE_NAME:$IMAGE_TAG
                """
            }
        }

        stage('Deploy Staging') {
            when { branch 'main' }
            steps {
                sshagent(['SSH_AUTH_SERVER']) {
                    sh "ssh -o StrictHostKeyChecking=no ubuntu@${STAGING_IP} 'docker pull ${IMAGE_NAME}:${IMAGE_TAG} && docker rm -f paymybuddy-staging || true && docker run -d --name paymybuddy-staging -p 8081:8080 ${IMAGE_NAME}:${IMAGE_TAG}'"
                }
            }
        }

        stage('Deploy Production') {
            when { branch 'main' }
            steps {
                sshagent(['SSH_AUTH_SERVER']) {
                    sh "ssh -o StrictHostKeyChecking=no ubuntu@${PROD_IP} 'docker pull ${IMAGE_NAME}:${IMAGE_TAG} && docker rm -f paymybuddy-prod || true && docker run -d --name paymybuddy-prod -p 80:8080 ${IMAGE_NAME}:${IMAGE_TAG}'"
                }
            }
        }

        stage('Validation') {
            when { branch 'main' }
            steps {
                sleep 15
                sh "curl -f http://${PROD_IP}:80 || exit 1"
            }
        }
    }

    post {
        always {
            script {
                try {
                    def color = (currentBuild.currentResult == 'SUCCESS') ? 'good' : 'danger'
                    slackSend(channel: '#jenkins-notification2025', color: color, tokenCredentialId: "${SLACK_TOKEN_ID}", message: "PayMyBuddy #${env.BUILD_NUMBER} - Result: ${currentBuild.currentResult}")
                } catch (e) {
                    echo "Slack fail: ${e.message}"
                }
            }
        }
    }
}
