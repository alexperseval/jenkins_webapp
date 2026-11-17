
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
        HOST_PORT = '8067'
        CONTAINER_PORT = '89'

        // Gmail SMTP
        EMAIL_CREDENTIALS = 'Cred-Gmail-Mezghich'
        EMAIL_TO = 'alexandre.perseval@magellangroup.eu'
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
            echo 'DEPLOIEMENT SPRING BOOT REUSSI !'

            emailext(
                subject: "SUCCESS: ${JOB_NAME} - Build #${BUILD_NUMBER}",
                mimeType: 'text/html',
                body: """
                    <h2 style="color:green;">Pipeline CI/CD réussie !</h2>

                    <p>Bonjour,</p>

                    <p>Le déploiement de votre application Spring Boot
                    a été effectué avec succès.</p>

                    <table border="1" cellpadding="8" cellspacing="0">
                        <tr><td><b>Projet</b></td><td>${JOB_NAME}</td></tr>
                        <tr><td><b>Build</b></td><td>#${BUILD_NUMBER}</td></tr>
                        <tr><td><b>Statut</b></td><td>SUCCESS</td></tr>
                        <tr><td><b>Image Docker</b></td><td>${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${BUILD_NUMBER}</td></tr>
                        <tr><td><b>Container</b></td><td>${CONTAINER_NAME}</td></tr>
                        <tr><td><b>Application</b></td><td>http://jenkins.sip-academy.com:${HOST_PORT}</td></tr>
                    </table>

                    <p>
                        <a href="${BUILD_URL}">Consulter le build Jenkins</a>
                    </p>

                    <p>Cordialement,<br>Jenkins CI/CD</p>
                """,
                to: "${EMAIL_TO}"
            )
        }

        failure {
            echo 'ECHEC DE LA PIPELINE CI/CD !'

            emailext(
                subject: "FAILURE: ${JOB_NAME} - Build #${BUILD_NUMBER}",
                mimeType: 'text/html',
                body: """
                    <h2 style="color:red;">Échec de la pipeline CI/CD !</h2>

                    <p>Bonjour,</p>

                    <p>Une erreur est survenue pendant l'exécution
                    de votre pipeline Spring Boot.</p>

                    <table border="1" cellpadding="8" cellspacing="0">
                        <tr><td><b>Projet</b></td><td>${JOB_NAME}</td></tr>
                        <tr><td><b>Build</b></td><td>#${BUILD_NUMBER}</td></tr>
                        <tr><td><b>Statut</b></td><td>FAILURE</td></tr>
                        <tr><td><b>Image Docker</b></td><td>${DOCKERHUB_USERNAME}/${IMAGE_NAME}</td></tr>
                    </table>

                    <p>
                        <a href="${BUILD_URL}console">
                            Consulter les logs Jenkins
                        </a>
                    </p>

                    <p>Cordialement,<br>Jenkins CI/CD</p>
                """,
                to: "${EMAIL_TO}"
            )
        }

        always {
            echo 'Fin de la pipeline Jenkins.'
        }
    }
}