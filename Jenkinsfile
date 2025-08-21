pipeline {
  agent { label 'integration-test' }

  parameters {
    // Se chiamato da un microservizio, passa SERVICE_NAME e TRIGGER_TYPE=single
    string(name: 'SERVICE_NAME', defaultValue: 'none', description: 'Nome del microservizio (solo per trigger single)')
    choice(name: 'TRIGGER_TYPE', choices: ['push', 'single'], description: 'push=deploy completo; single=rollout del servizio')
  }

  environment {
    MK8S    = "/snap/bin/microk8s"
    K8S_DIR = "k8s"
    // fallback robusti anche in multibranch
    SVC     = "${params.SERVICE_NAME ?: 'assignment'}"
    MODE    = "${params.TRIGGER_TYPE ?: 'push'}"
    NS_LIST = "assignment review user-manager submission"
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

        microk8s kubectl delete ingress assignments-ingress -n assignment --ignore-not-found
        microk8s kubectl delete ingress submissions-ingress -n submission --ignore-not-found
        microk8s kubectl delete ingress review-ingress -n review --ignore-not-found
        microk8s kubectl delete ingress user-manager-ingress -n user-manager --ignore-not-found

        # 1) Namespace
        ${MK8S} kubectl apply -f "${K8S_DIR}/namespaces.yaml" 

        # 2) Databases
        ${MK8S} kubectl apply -R -f "${K8S_DIR}/databases" 

        # 3) Services
        ${MK8S} kubectl apply -R -f "${K8S_DIR}/services" 

        # 4) Config, Secrets, Ingress
        ${MK8S} kubectl apply -f "${K8S_DIR}/config.yaml"  
        ${MK8S} kubectl apply -f "${K8S_DIR}/secrets.yaml" 
        ${MK8S} kubectl apply -f "${K8S_DIR}/ingress.yaml" 

        # 5) RabbitMQ
        helm install rabbitmq bitnami/rabbitmq -n rabbitmq -f rabbitmq-values.yaml

        # 5) Rollout deployments
        ${MK8S} kubectl rollout restart deploy -n user-manager -l app=user-manager
        ${MK8S} kubectl rollout restart deploy -n assignment -l app=assignment
        ${MK8S} kubectl rollout restart deploy -n submission -l app=submission
        ${MK8S} kubectl rollout restart deploy -n review -l app=review

        # 6) Atessa restart
        ${MK8S} kubectl rollout status deployment/user-manager -n user-manager --timeout=80s
        ${MK8S} kubectl rollout status deployment/submission -n submission --timeout=80s
        ${MK8S} kubectl rollout status deployment/assignment -n assignment --timeout=80s
        ${MK8S} kubectl rollout status deployment/review -n review --timeout=80s

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
        ${MK8S} kubectl rollout restart deployment -n "${NS}" -l app="${NS}"

        # Attendi il completamento del rollout
        ${MK8S} kubectl rollout status deployment/"${NS}" -n "${NS}" --timeout=80s || {
            echo "Rollout fallito"
            exit 1
        }

        '''
        }
    }

    stage('Run API Tests with Newman user-manager') {
      when { expression { env.SVC == 'user-manager' || env.MODE == "push" } }
      steps {
        sh '''
          newman run ./test/postman/Autenticazione.postman_collection.json \
            -e ./test/postman/postman-env.postman_environment.json \
            --reporters cli,json
        '''
      }
    }

    stage('Run API Tests with Newman Assignments') {
      when { expression { env.SVC == 'assignment' || env.MODE == "push" } }
      steps {
        sh '''
          newman run ./test/postman/Assignments.postman_collection.json \
            -e ./test/postman/postman-env.postman_environment.json \
            --reporters cli,json
        '''
      }
    }

    stage('Run API Tests with Newman Submissions') {
      when { expression { env.SVC == 'submission' || env.MODE == "push" } }
      steps {
        sh '''
          newman run ./test/postman/Submissions.postman_collection.json \
            -e ./test/postman/postman-env.postman_environment.json \
            --reporters cli,json
        '''
      }
    }

    stage('Run API Tests with Newman review') {
      when { expression { env.SVC == 'review' || env.MODE == "push" } }
      steps {
        sh '''
          newman run ./test/postman/Review.postman_collection.json \
            -e ./test/postman/postman-env.postman_environment.json \
            --reporters cli,json
        '''
      }
    }
  }
      
  post {
    always { 
      sh '''
        microk8s kubectl -n user-manager exec mongodb-0 -- \
          mongosh "mongodb://localhost:27017/users" --quiet \
          --eval 'const r = db.users.deleteMany({}); print("deleted from users.users:", r.deletedCount)'

        microk8s kubectl -n assignment exec mongodb-0 -- \
          mongosh "mongodb://localhost:27017/assignment" --quiet \
          --eval 'const r = db.assignments.deleteMany({}); print("deleted from assignment.assignments:", r.deletedCount)'

        microk8s kubectl -n submission exec mongodb-0 -- \
          mongosh "mongodb://localhost:27017/submission" --quiet \
          --eval '
            db.getCollectionNames().forEach(c => {
              const r = db.getCollection(c).deleteMany({});
              print("Svuotata collection:", c, "deleted:", r.deletedCount);
            })'

        microk8s kubectl -n review exec mongodb-0 -- \
          mongosh "mongodb://localhost:27017/review" --quiet \
          --eval 'const r = db.reviews.deleteMany({}); print("deleted from review.reviews:", r.deletedCount)'
        '''
    }
    success { echo "✅ Done — MODE=${env.MODE}, SERVICE=${env.SVC}" }
    failure { echo "❌ Deploy fallito — controlla i log" }
  }
}
