
pipeline {
    agent any

    tools {
        maven 'Maven310'
        jdk 'Java17'
    }

    environment {
        // GitHub public repository
        GITHUB_REPO = 'https://github.com/alexperseval/jenkins_webapp.git'
        GITHUB_BRANCH = 'master'

        // Docker Hub
        DOCKERHUB_USERNAME = 'alexperseval'
        DOCKERHUB_CREDENTIALS = 'dockerhub-credentials-alexandre'
        IMAGE_NAME = 'ibcgos-app-springboot-alexandre'

        // Docker container
        CONTAINER_NAME = 'ibcgos-app-springboot-alexandre-container'
        HOST_PORT = '8065'
        CONTAINER_PORT = '89'
    }

    stages {

        stage('1. Checkout') {
            steps {
                echo 'Recuperation du code depuis GitHub...'

                git branch: "${GITHUB_BRANCH}",
                    url: "${GITHUB_REPO}"
            }
        }

        stage('2. Build JAR') {
            steps {
                echo 'Compilation du projet Spring Boot...'

                sh '''
                    mvn clean package -DskipTests
                '''

                echo 'JAR genere avec succes.'
            }
        }

        stage('3. Build Docker Image') {
            steps {
                echo 'Construction de l image Docker...'

                sh '''
                    docker build \
                      -t ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${BUILD_NUMBER} \
                      -t ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:latest \
                      .
                '''
            }
        }

        stage('4. DockerHub Login & Push') {
            steps {
                echo 'Connexion et publication sur Docker Hub...'

                withCredentials([
                    usernamePassword(
                        credentialsId: "${DOCKERHUB_CREDENTIALS}",
                        usernameVariable: 'DH_USER',
                        passwordVariable: 'DH_TOKEN'
                    )
                ]) {
                    sh '''
                        set +x
                        trap 'docker logout >/dev/null 2>&1 || true' EXIT

                        printf '%s' "$DH_TOKEN" | docker login \
                          -u "$DH_USER" \
                          --password-stdin

                        docker push \
                          ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${BUILD_NUMBER}

                        docker push \
                          ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:latest
                    '''
                }
            }
        }

        stage('5. Stop Old Container') {
            steps {
                echo 'Arret de l ancienne instance...'

                sh '''
                    docker stop ${CONTAINER_NAME} || true
                    docker rm ${CONTAINER_NAME} || true
                '''
            }
        }

        stage('6. Deploy Container') {
            steps {
                echo 'Demarrage du nouveau conteneur Spring Boot...'

                sh '''
                    docker run -d \
                      --name ${CONTAINER_NAME} \
                      --restart unless-stopped \
                      -p ${HOST_PORT}:${CONTAINER_PORT} \
                      ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${BUILD_NUMBER}
                '''
            }
        }

        stage('7. Verify Deployment') {
            steps {
                echo 'Verification du deploiement...'

                sh '''
                    sleep 10

                    docker ps --filter "name=${CONTAINER_NAME}"

                    HTTP_CODE=$(curl -sS \
                      -o /dev/null \
                      -w "%{http_code}" \
                      http://localhost:${HOST_PORT}/)

                    echo "HTTP Status: ${HTTP_CODE}"

                    if [ "$HTTP_CODE" = "000" ] || [ "$HTTP_CODE" -ge 500 ]; then
                        echo "Application non accessible !"
                        exit 1
                    fi
                '''
            }
        }
    }

    post {
        success {
            echo '======================================'
            echo 'DEPLOIEMENT SPRING BOOT REUSSI !'
            echo "Docker Hub: ${DOCKERHUB_USERNAME}/${IMAGE_NAME}"
            echo "Tag: ${BUILD_NUMBER}"
            echo '======================================'
        }

        failure {
            echo 'ECHEC DE LA PIPELINE CI/CD !'
        }

        always {
            echo 'Fin de la pipeline Jenkins.'
        }
    }
}
