pipeline {
  agent { label 'integration-test' }

  parameters {
    // Se chiamato da un microservizio, passa SERVICE_NAME e TRIGGER_TYPE=single
    string(name: 'SERVICE_NAME', defaultValue: 'assignment', description: 'Nome del microservizio (solo per trigger single)')
    choice(name: 'TRIGGER_TYPE', choices: ['push', 'single'], description: 'push=deploy completo; single=rollout del servizio')
  }

  environment {
    MK8S    = "/snap/bin/microk8s"
    K8S_DIR = "k8s"
    // fallback robusti anche in multibranch
    SVC     = "${params.SERVICE_NAME ?: 'assignment'}"
    MODE    = "${params.TRIGGER_TYPE ?: 'push'}"
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Cluster Ready') {
      steps {
        sh """
          ${MK8S} status --wait-ready
          ${MK8S} kubectl version --client=true
        """
      }
    }

    stage('Deploy FULL (trigger=push)') {
    when { expression { env.MODE == 'push' } }
    steps {
        echo "🚀 Deploy completo dell'ambiente di test"

        sh """
        set -e

        # 1) Namespace
        ${MK8S} kubectl apply -f "${K8S_DIR}/namespaces.yaml" || true

        # 2) Databases
        ${MK8S} kubectl apply -R -f "${K8S_DIR}/databases" || true

        # 3) Services
        ${MK8S} kubectl apply -R -f "${K8S_DIR}/services" || true

        # 4) Config, Secrets, Ingress
        ${MK8S} kubectl apply -f "${K8S_DIR}/config.yaml"   || true
        ${MK8S} kubectl apply -f "${K8S_DIR}/secrets.yaml"  || true
        ${MK8S} kubectl apply -f "${K8S_DIR}/ingress.yaml"  || true
        """
        }
    }

    stage('Rollout SINGLE (trigger=single)') {
    when { expression { env.MODE == 'single' } }
    steps {
        echo "♻️ Rollout del servizio: ${env.SVC}"
        sh '''
        set -e
        NS="${SVC}"
        DEP="${K8S_DIR}/services/${SVC}/deployment.yaml"

        if [ -f "${DEP}" ]; then
            ${MK8S} kubectl apply -f "${DEP}" -n "${NS}" || true
        fi

        # Forza il restart: con imagePullPolicy: Always scaricherà la nuova immagine
        ${MK8S} kubectl rollout restart deployment "${SVC}" -n "${NS}"
        '''
        }
    }

    stage('Run API Tests with Newman Assignments') {
      steps {
        sh '''
          newman run ./test/postman/Assignments.postman_collection.json \
            -e ./test/postman/postman-env.postman_environment.json \
            --reporters cli,json
        '''
      }
    }

    stage('Run API Tests with Newman Submissions') {
      steps {
        sh '''
          newman run ./test/postman/Submissions.postman_collection.json \
            -e ./test/postman/postman-env.postman_environment.json
            --reporters cli,json
        ''' 
      }
    }
  }
      
  post {
    success { echo "✅ Done — MODE=${env.MODE}, SERVICE=${env.SVC}" }
    failure { echo "❌ Deploy fallito — controlla i log" }
  }
}
