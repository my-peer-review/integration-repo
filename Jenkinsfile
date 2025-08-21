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
        set -euo pipefail
        IFS=$'\\n\\t'

        NS_LIST="assignment review user-manager submission"

        # 0) Pulizia ingress (idempotente)
        for ns in $NS_LIST; do
          ${MK8S} kubectl -n "$ns" delete ingress --all --ignore-not-found
        done

        # 1) Namespace (deve venire prima di tutto)
        ${MK8S} kubectl apply -f "${K8S_DIR}/namespaces.yaml"

        # 2) Databases (StatefulSet + Service)
        ${MK8S} kubectl apply -R -f "${K8S_DIR}/databases"

        # 3) Services applicativi
        ${MK8S} kubectl apply -R -f "${K8S_DIR}/services"

        # 4) Config, Secrets, Ingress
        ${MK8S} kubectl apply -f "${K8S_DIR}/config.yaml"
        ${MK8S} kubectl apply -f "${K8S_DIR}/secrets.yaml"
        ${MK8S} kubectl apply -f "${K8S_DIR}/ingress.yaml"

        # 4.1) Attendi che i DB (StatefulSet) siano Ready prima di rilanciare le app
        for ns in $NS_LIST; do
          ${MK8S} kubectl -n "$ns" rollout status statefulset --all --timeout=10m || true
        done

        # 5) Rollout restart di TUTTI i controller
        for ns in $NS_LIST; do
          echo "==> Restart in namespace $ns"
          ${MK8S} kubectl -n "$ns" rollout restart deploy --all || true
          ${MK8S} kubectl -n "$ns" rollout restart statefulset --all || true
          ${MK8S} kubectl -n "$ns" rollout restart daemonset --all || true
        done

        # 6) Attendi stabilizzazione (fallisci lo stage se i Deploy non tornano su)
        for ns in $NS_LIST; do
          ${MK8S} kubectl -n "$ns" rollout status deploy --all --timeout=5m || {
            echo "✖️ Rollout fallito in $ns, stato corrente:"
            ${MK8S} kubectl -n "$ns" get pods -o wide
            exit 1
          }
          ${MK8S} kubectl -n "$ns" rollout status daemonset --all --timeout=5m || true
          ${MK8S} kubectl -n "$ns" rollout status statefulset --all --timeout=10m || true
        done
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

    stage('Run API Tests with Newman user-manager') {
      steps {
        sh '''
          newman run ./test/postman/Autenticazione.postman_collection.json \
            -e ./test/postman/postman-env.postman_environment.json \
            --reporters cli,json
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
            -e ./test/postman/postman-env.postman_environment.json \
            --reporters cli,json
        '''
      }
    }

    stage('Run API Tests with Newman review') {
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
          mongosh "mongodb://localhost:27017/user" --quiet \
          --eval 'const r=db.users.deleteMany({}); print("deleted (post):", r.deletedCount);'
        '''
    }
    success { echo "✅ Done — MODE=${env.MODE}, SERVICE=${env.SVC}" }
    failure { echo "❌ Deploy fallito — controlla i log" }
  }
}
