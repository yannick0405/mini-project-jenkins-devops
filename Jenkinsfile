pipeline {
    agent any 

    environment {
        // Tes credentials Jenkins
        DOCKERHUB_CREDS = credentials('yann')
        SLACK_TOKEN_ID = 'slack-token'
        SONAR_TOKEN = credentials('sonar-token') 
        
        IMAGE_NAME = "yannick0405/paymybuddy"
        IMAGE_TAG = "v${env.BUILD_NUMBER}"
        
        // Serveurs cibles AWS
        STAGING_IP = "44.211.85.128"
        PROD_IP = "98.92.95.208"
    }

    stages {
        stage('Checkout & Clean') {
            steps {
                deleteDir()
                // Récupération du code de l'application
                git branch: 'main', url: 'https://github.com/yannick0405/PayMyBuddy.git'
            }
        }

       stage('Tests & SonarCloud') {
            steps {
                script {
                    docker.image('maven:3.9.1-eclipse-temurin-17').inside('-v $HOME/.m2:/root/.m2') {
                        echo "Exécution des tests et Analyse SonarCloud..."
                        
                        // Utilisation du GAV (GroupId:ArtifactId:Version) complet du plugin
                        sh """
                            mvn clean verify org.sonarsource.scanner.maven:sonar-maven-plugin:3.9.1.2184:sonar \
                            -Dsonar.projectKey=paymybuddy-project \
                            -Dsonar.organization=yannick-org \
                            -Dsonar.host.url=https://sonarcloud.io \
                            -Dsonar.login=${SONAR_TOKEN}
                        """
                    }
                }
            }
        }
        stage('Build & Push Docker') {
            steps {
                script {
                    // 1. Génération du JAR (sera target/paymybuddy.jar)
                    docker.image('maven:3.9.1-eclipse-temurin-17').inside('-v $HOME/.m2:/root/.m2') {
                        sh "mvn package -DskipTests"
                    }
                    
                    // 2. Build de l'image Docker et envoi vers DockerHub
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
                    sh """
                        ssh -o StrictHostKeyChecking=no ubuntu@${STAGING_IP} "
                            docker pull ${IMAGE_NAME}:${IMAGE_TAG} && \
                            docker rm -f paymybuddy-staging || true && \
                            docker run -d --name paymybuddy-staging -p 8081:8080 ${IMAGE_NAME}:${IMAGE_TAG}
                        "
                    """
                }
            }
        }

        stage('Deploy Production') {
            when { branch 'main' }
            steps {
                // Demande de validation avant la prod (optionnel, mais pro)
                input message: "Déployer en Production ?", ok: "Oui"
                
                sshagent(['SSH_AUTH_SERVER']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ubuntu@${PROD_IP} "
                            docker pull ${IMAGE_NAME}:${IMAGE_TAG} && \
                            docker rm -f paymybuddy-prod || true && \
                            docker run -d --name paymybuddy-prod -p 80:8080 ${IMAGE_NAME}:${IMAGE_TAG}
                        "
                    """
                }
            }
        }

        stage('Validation Finale') {
            when { branch 'main' }
            steps {
                echo "Vérification que l'application répond..."
                sleep 20 // On laisse le temps à Spring de démarrer
                sh "curl -f http://${PROD_IP}:80 || echo 'L application met du temps à répondre, vérifiez les logs Docker sur le serveur'"
            }
        }
    }

    post {
        always {
            script {
                try {
                    def color = (currentBuild.currentResult == 'SUCCESS') ? 'good' : 'danger'
                    slackSend(channel: '#jenkins-notification2025', color: color, tokenCredentialId: "${SLACK_TOKEN_ID}", message: "Pipeline PayMyBuddy #${env.BUILD_NUMBER} : ${currentBuild.currentResult}")
                } catch (e) {
                    echo "Notification Slack échouée, mais le build est terminé."
                }
            }
        }
    }
}
