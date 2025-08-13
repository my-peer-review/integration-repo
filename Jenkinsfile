pipeline {
    agent any

    parameters {
        string(name: 'SERVICE_NAME', defaultValue: '', description: 'Nome del microservizio (es: assignment)')
        string(name: 'IMAGE_TAG', defaultValue: '', description: 'Tag dell\'immagine da deployare')
    }

    environment {
        DOCKER_USER = 'ale175'
        NAMESPACE = "${params.SERVICE_NAME}"
    }

    stages {

        stage('Checkout integration repo') {
            steps {
                checkout scm
            }
        }

        stage('Aggiorna deployment') {
            steps {
                script {
                    def deployFile = "k8s/services/${SERVICE_NAME}/deployment.yaml"

                    echo "Aggiorno immagine in ${deployFile} a ${DOCKER_USER}/${SERVICE_NAME}:${IMAGE_TAG}"

                    sh """
                        sed -i 's|image: .*${SERVICE_NAME}:.*|image: ${DOCKER_USER}/${SERVICE_NAME}:${IMAGE_TAG}|' ${deployFile}
                    """
                }
            }
        }

        stage('Applica manifest su cluster') {
            steps {
                script {
                    sh """
                        microk8s kubectl apply -f k8s/services/${SERVICE_NAME}/deployment.yaml -n ${NAMESPACE}
                        microk8s kubectl apply -f k8s/services/${SERVICE_NAME}/service.yaml -n ${NAMESPACE}
                    """
                }
            }
        }

        stage('Rollout restart e verifica') {
            steps {
                script {
                    sh """
                        microk8s kubectl rollout restart deployment/${SERVICE_NAME} -n ${NAMESPACE}
                        microk8s kubectl rollout status deployment/${SERVICE_NAME} -n ${NAMESPACE} --timeout=60s
                        microk8s kubectl get pods -n ${NAMESPACE} -o wide
                    """
                }
            }
        }
    }

    post {
        success {
            echo "✅ Deploy completato per ${SERVICE_NAME}:${IMAGE_TAG} nel namespace ${NAMESPACE}"
        }
        failure {
            echo "❌ Deploy fallito per ${SERVICE_NAME}:${IMAGE_TAG}"
        }
    }
}
