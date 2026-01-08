pipeline {
    agent any 

    environment {
        // Tes credentials
        DOCKERHUB_CREDS = credentials('yann')
        SLACK_TOKEN_ID = 'slack-token'
        SONAR_TOKEN = credentials('sonar-token') // À créer dans Jenkins
        
        IMAGE_NAME = "yannick0405/paymybuddy"
        IMAGE_TAG = "v${env.BUILD_NUMBER}"
        
        // Serveurs cibles
        STAGING_IP = "44.211.85.128"
        PROD_IP = "98.92.95.208"
    }

    stages {
        stage('Checkout Source') {
            steps {
                // On récupère le code de l'application
                git branch: 'main', url: 'https://github.com/yannick0405/PayMyBuddy.git'
            }
        }

        stage('Tests & SonarCloud') {
            agent {
                docker {
                    image 'maven:3.9.1-eclipse-temurin-17'
                    args '-v $HOME/.m2:/root/.m2'
                }
            }
            steps {
                dir('paymybuddy') {
                    echo "Exécution des tests unitaires..."
                    sh './mvnw clean test'
                    
                    echo "Analyse SonarCloud..."
                    // Remplace projectKey et organization par tes valeurs SonarCloud
                    sh "export SONAR_TOKEN=${SONAR_TOKEN} && ./mvnw sonar:sonar -Dsonar.projectKey=paymybuddy-project -Dsonar.organization=yannick-org -Dsonar.host.url=https://sonarcloud.io"
                }
            }
        }

        stage('Build & Push Docker') {
            steps {
                dir('paymybuddy') {
                    echo "Compilation du JAR..."
                    // On utilise un conteneur temporaire pour le build Maven
                    sh "docker run --rm -v $HOME/.m2:/root/.m2 -v \$(pwd):/app -w /app maven:3.9.1-eclipse-temurin-17 ./mvnw package -DskipTests"
                    
                    echo "Build et Push de l'image..."
                    sh "docker build -t $IMAGE_NAME:$IMAGE_TAG ."
                    sh "echo $DOCKERHUB_CREDS_PSW | docker login -u $DOCKERHUB_CREDS_USR --password-stdin"
                    sh "docker push $IMAGE_NAME:$IMAGE_TAG"
                }
            }
        }

        // --- DÉPLOIEMENT SSH (BRANCH MAIN UNIQUEMENT) ---

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
                // Validation manuelle optionnelle
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

        stage('Validation') {
            when { branch 'main' }
            steps {
                echo "Vérification du déploiement..."
                sleep 10
                sh "curl -f http://${PROD_IP}:80 || exit 1"
            }
        }
    }

    post {
        success {
            slackSend(channel: '#jenkins-notification2025', color: 'good', tokenCredentialId: "${SLACK_TOKEN_ID}", message: "✅ SUCCESS: PayMyBuddy #${env.BUILD_NUMBER} déployé avec succès !")
        }
        failure {
            slackSend(channel: '#jenkins-notification2025', color: 'danger', tokenCredentialId: "${SLACK_TOKEN_ID}", message: "❌ FAILURE: PayMyBuddy #${env.BUILD_NUMBER} a échoué.")
        }
    }
}
