pipeline {
    agent any

    parameters {
        string(name: 'SERVICE_NAME', defaultValue: '', description: 'Nome del microservizio (es: assignment)')
        string(name: 'IMAGE_TAG', defaultValue: 'latest', description: 'Tag dell\'immagine Docker')
    }

    environment {
        K8S_NAMESPACE = "${params.SERVICE_NAME}"  // namespace uguale al servizio
        DEPLOY_PATH = "k8s/services/${params.SERVICE_NAME}/deployment.yaml"
    }

    stages {
        stage('Check Parameters') {
            steps {
                echo "Deploying service: ${params.SERVICE_NAME}"
                echo "Using image tag: ${params.IMAGE_TAG}"
            }
        }

        stage('Deploy to MicroK8s') {
            steps {
                script {
                    sh """
                        # Verifica accesso al cluster
                        microk8s status --wait-ready

                        # Applica sempre il manifest con imagePullPolicy: Always
                        microk8s kubectl apply -f ${DEPLOY_PATH} -n ${K8S_NAMESPACE}

                        # Forza il rollout per ricaricare i pod
                        microk8s kubectl rollout restart deployment ${SERVICE_NAME} -n ${K8S_NAMESPACE}

                        # Attende il completamento del rollout
                        microk8s kubectl rollout status deployment ${SERVICE_NAME} -n ${K8S_NAMESPACE}
                    """
                }
            }
        }
    }

    post {
        success {
            echo "✅ Deploy completato per ${params.SERVICE_NAME}:${params.IMAGE_TAG}"
        }
        failure {
            echo "❌ Deploy fallito per ${params.SERVICE_NAME}:${params.IMAGE_TAG}"
        }
    }
}
