pipeline {
    agent { label 'integration' }

    parameters {
        string(name: 'SERVICE_NAME', defaultValue: 'assignment', description: 'Nome del microservizio')
        string(name: 'IMAGE_TAG', defaultValue: 'latest', description: 'Tag immagine Docker')
    }

    environment {
        SERVICE_NAME = "${params.SERVICE_NAME ?: 'assignment'}"
        IMAGE_TAG = "${params.IMAGE_TAG ?: 'latest'}"
        K8S_NAMESPACE = "${SERVICE_NAME}"
        DEPLOY_PATH = "k8s/services/${SERVICE_NAME}/deployment.yaml"
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
                sh """
                    microk8s status --wait-ready
                    microk8s kubectl apply -f ${DEPLOY_PATH} -n ${K8S_NAMESPACE}
                    microk8s kubectl rollout restart deployment ${SERVICE_NAME} -n ${K8S_NAMESPACE}
                    microk8s kubectl rollout status deployment ${SERVICE_NAME} -n ${K8S_NAMESPACE}
                """
            }
        }
    }
}
