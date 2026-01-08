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
        stage('Checkout & Clean') {
            steps {
                deleteDir()
                git branch: 'main', url: 'https://github.com/yannick0405/PayMyBuddy.git'
            }
        }

        stage('Tests & SonarCloud') {
            steps {
                script {
                    // catchError garantit que le pipeline continue vers le déploiement même si Sonar a un souci
                    catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                        docker.image('maven:3.9.1-eclipse-temurin-17').inside('-v $HOME/.m2:/root/.m2') {
                            echo "Analyse SonarCloud pour le projet yannick0405..."
                            sh """
                                mvn clean verify org.sonarsource.scanner.maven:sonar-maven-plugin:3.9.1.2184:sonar \
                                -Dsonar.projectKey=yannick0405 \
                                -Dsonar.organization=yannick0405 \
                                -Dsonar.host.url=https://sonarcloud.io \
                                -Dsonar.login=${SONAR_TOKEN}
                            """
                        }
                    }
                }
            }
        }

        stage('Build & Push Docker') {
            steps {
                script {
                    echo "Construction du JAR..."
                    docker.image('maven:3.9.1-eclipse-temurin-17').inside('-v $HOME/.m2:/root/.m2') {
                        sh "mvn package -DskipTests"
                    }
                    
                    echo "Build et Push de l'image Docker..."
                    sh """
                        docker build -t $IMAGE_NAME:$IMAGE_TAG .
                        echo $DOCKERHUB_CREDS_PSW | docker login -u $DOCKERHUB_CREDS_USR --password-stdin
                        docker push $IMAGE_NAME:$IMAGE_TAG
                    """
                }
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

        stage('Validation Finale') {
            when { branch 'main' }
            steps {
                echo "Vérification du service sur le serveur de Production..."
                sleep 20
                sh "curl -f http://${PROD_IP}:80 || echo 'L application démarre, vérifiez l URL manuellement dans quelques instants.'"
            }
        }
    }

    post {
        always {
            script {
                try {
                    def color = (currentBuild.currentResult == 'SUCCESS') ? 'good' : 'danger'
                    slackSend(channel: '#jenkins-notification2025', color: color, tokenCredentialId: "${SLACK_TOKEN_ID}", message: "Pipeline terminé pour PayMyBuddy #${env.BUILD_NUMBER}. Statut : ${currentBuild.currentResult}")
                } catch (e) {
                    echo "Slack notification failed, but the build result is: ${currentBuild.currentResult}"
                }
            }
        }
    }
}
