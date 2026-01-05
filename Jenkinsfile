pipeline {
    agent none
    environment {
        DOCKERHUB_AUTH = credentials('blondel')
        ID_DOCKER = "${DOCKERHUB_AUTH_USR}"
        PORT_EXPOSED = "80"
        HOSTNAME_DEPLOY_PROD = "54.226.252.171"
        HOSTNAME_DEPLOY_STAGING = "54.175.193.5"
        IMAGE_NAME = "webappstaticbootcamp28"
        IMAGE_TAG = "v1"
    }
    stages {
        stage ('Build Images with docker') {
            agent any
            steps {
                script {
                    sh 'docker build -t ${ID_DOCKER}/${IMAGE_NAME}:${IMAGE_TAG} .'
                }
            }
        }

        stage('Run container based on builded image') {
            agent any
            steps {
               script {
                 sh '''
                    echo "Clean Environment"
                    docker rm -f $IMAGE_NAME || echo "container does not exist"
                    docker run --name $IMAGE_NAME -d -p ${PORT_EXPOSED}:80 ${ID_DOCKER}/$IMAGE_NAME:$IMAGE_TAG
                    sleep 5
                 '''
               }
            }
        }

        stage('Test image') {
            agent any
            steps {
                script {
                    sh '''
                        curl http://172.17.0.1:${PORT_EXPOSED} | grep -q "Dimension"
                    '''
                }
            }
        }

        stage('Clean Container') {
            agent any
            steps {
                script {
                    sh '''
                        docker stop $IMAGE_NAME
                        docker rm $IMAGE_NAME
                    '''
                }
            }
        }

        stage ('Login and Push Image on docker hub') {
            agent any           
            steps {
                script {
                    sh '''
                        docker login -u $DOCKERHUB_AUTH_USR -p $DOCKERHUB_AUTH_PSW
                        docker push ${ID_DOCKER}/$IMAGE_NAME:$IMAGE_TAG
                    '''
                }
            }
        }

        stage ('Deploy in staging') {
            agent any
            steps {
                sshagent(credentials: ['SSH_AUTH_SERVER']) {
                    sh '''
                        [ -d ~/.ssh ] || mkdir ~/.ssh && chmod 0700 ~/.ssh
                        command1="docker login -u $DOCKERHUB_AUTH_USR -p $DOCKERHUB_AUTH_PSW"
                        command2="docker pull $DOCKERHUB_AUTH_USR/$IMAGE_NAME:$IMAGE_TAG"
                        command3="docker rm -f webappstatic || echo 'app does not exist'"
                        command4="docker run -d -p 80:80 --name webappstatic $DOCKERHUB_AUTH_USR/$IMAGE_NAME:$IMAGE_TAG"
                        ssh -o StrictHostKeyChecking=no ubuntu@${HOSTNAME_DEPLOY_STAGING} \
                            -o SendEnv=IMAGE_NAME \
                            -o SendEnv=IMAGE_TAG \
                            -o SendEnv=DOCKERHUB_AUTH_USR \
                            -o SendEnv=DOCKERHUB_AUTH_PSW \
                            -C "$command1 && $command2 && $command3 && $command4"
                    '''
                }
            }
        }

         stage('Test Staging') {
            agent any
            steps {
              script {
                sh '''
                  curl ${HOSTNAME_DEPLOY_STAGING} | grep -q "Dimension"
                '''
              }
            }
        }

        stage ('Deploy in prod') {
            agent any
            steps {
                sshagent(credentials: ['SSH_AUTH_SERVER']) {
                    sh '''
                        [ -d ~/.ssh ] || mkdir ~/.ssh && chmod 0700 ~/.ssh
                        command1="docker login -u $DOCKERHUB_AUTH_USR -p $DOCKERHUB_AUTH_PSW"
                        command2="docker pull $DOCKERHUB_AUTH_USR/$IMAGE_NAME:$IMAGE_TAG"
                        command3="docker rm -f webappstatic || echo 'app does not exist'"
                        command4="docker run -d -p 80:80 --name webappstatic $DOCKERHUB_AUTH_USR/$IMAGE_NAME:$IMAGE_TAG"
                        ssh -o StrictHostKeyChecking=no ubuntu@${HOSTNAME_DEPLOY_PROD} \
                            -o SendEnv=IMAGE_NAME \
                            -o SendEnv=IMAGE_TAG \
                            -o SendEnv=DOCKERHUB_AUTH_USR \
                            -o SendEnv=DOCKERHUB_AUTH_PSW \
                            -C "$command1 && $command2 && $command3 && $command4"
                    '''
                }
            }
        }

        stage('Test Prod') {
          agent any
          steps {
             script {
               sh '''
                 curl ${HOSTNAME_DEPLOY_PROD} | grep -q "Dimension"
               '''
             }
          }
        }

    }
}
