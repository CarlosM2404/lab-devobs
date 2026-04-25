pipeline {
    agent any

    options {
        timeout(time: 10, unit: 'MINUTES') // Tiempo máximo para la ejecución del pipeline
    }

    environment {
        NEXUS_URL = "http://host.docker.internal:8083"
        CREDENTIALS_ID = "nexus-credentials"
        IMAGE_NAME = "sumador" // Nombre de la imagen Docker
        IMAGE_TAG = "${env.BUILD_NUMBER}" // Etiqueta de la imagen basada en el número de build
        NEXUS_HOST = "host.docker.internal:8083" // Host y puerto de Nexus
        ARTIFACT_ID = "elbuo8/webapp:${env.BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                echo "Código obtenido desde GitHub"
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "Building Docker image..."
                sh """
                docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                """
            }
        }

        stage('Run tests') {
          steps {
            echo "Ejecutando tests..."
            sh "docker run --rm ${IMAGE_NAME}:${IMAGE_TAG} npm test"
          }
        }

        stage('Security - Dependency Audit') {
            steps {
                echo "Auditando dependencias npm..."
                sh "docker run --rm ${IMAGE_NAME}:${IMAGE_TAG} npm audit --audit-level=critical"
            }
        }

        stage('Security - Trivy Image Scan') {
            steps {
                echo "Escaneando imagen con Trivy (vulnerabilidades criticas)..."
                sh """
                    docker run --rm \
                        -v /var/run/docker.sock:/var/run/docker.sock \
                        aquasec/trivy image \
                        --severity CRITICAL \
                        --exit-code 1 \
                        ${IMAGE_NAME}:${IMAGE_TAG}
                """
            }
        }
        
        stage('Tag Docker Image') {
            steps {
                echo "Tagging Docker image for Nexus repository..."
                sh """
                docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${NEXUS_HOST}/${IMAGE_NAME}:${IMAGE_TAG}
                """
            }
        }

        stage('Deploy Image to Nexus') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "${CREDENTIALS_ID}",
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    sh """
                        echo \${NEXUS_PASS} | docker login ${NEXUS_HOST} -u \${NEXUS_USER} --password-stdin
                        docker push ${NEXUS_HOST}/${IMAGE_NAME}:${IMAGE_TAG}
                        docker logout ${NEXUS_HOST}
                    """
                }
            }
        }
    }

    post {
        always {
            echo "Cleaning up local Docker images..."
            sh """
            docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true
            docker rmi ${NEXUS_HOST}/${IMAGE_NAME}:${IMAGE_TAG} || true
            """
        }
        success {
            echo "Pipeline completed successfully!"
        }
        failure {
            echo "Pipeline failed. Check the logs for details."
        }
    }
}