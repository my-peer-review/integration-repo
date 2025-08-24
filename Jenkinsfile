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
    NS_LIST = "assignment review user-manager submission report"
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

    stage('Delete Deployments & Pods') {
      steps {
        sh """
          NS="user-manager assignment submission review report"

          for ns in ${NS_LIST}; do
            echo "Cancellazione deployment in $ns"
            ${MK8S} kubectl delete deploy --all -n "$ns" --ignore-not-found || true
          done
        """
      }
    }

    stage('Recreate namespaces & base manifests') {
      when { expression { env.MODE == 'push' } }
      steps {
        sh """
          echo "Applying manifests…"
          ${MK8S} kubectl apply -f "${K8S_DIR}/namespaces.yaml"
          ${MK8S} kubectl apply -R -f "${K8S_DIR}/databases"
          ${MK8S} kubectl apply -R -f "${K8S_DIR}/services"
          ${MK8S} kubectl apply -f "${K8S_DIR}/config.yaml"
          ${MK8S} kubectl apply -f "${K8S_DIR}/secrets.yaml"
          ${MK8S} kubectl apply -f "${K8S_DIR}/ingress.yaml"
        """
      }
    }

    stage('Wait for Deployments') {
      when { expression { env.MODE == 'push' } }
      steps {
        sh """
          NS="user-manager assignment submission review"

          for ns in $NS; do
            for d in $(${MK8S} kubectl get deploy -n "$ns" -o name); do
              echo "⏳ Attendo rollout $d in $ns"
              ${MK8S} kubectl rollout status -n "$ns" "$d" --timeout=80s
            done
          done
        """
      }
    }

    stage('Rollout SINGLE (trigger=single)') {
    when { expression { env.MODE == 'single' } }
    steps {
        echo "♻️ Rollout del servizio: ${env.SVC}"
        sh """
        set -e
        NS="${SVC}"
        DEP="${K8S_DIR}/services/${SVC}/deployment.yaml"

        if [ -f "${DEP}" ]; then
            ${MK8S} kubectl apply -f "${DEP}" -n "${NS}" || true
        fi

        microk8s kubectl scale deploy -n "${NS}" -l app="${NS}" --replicas=0

        microk8s kubectl wait --for=delete pod -n "${NS}" -l app="${NS}" --timeout=60s || true

        microk8s kubectl scale deploy -n "${NS}" -l app="${NS}" --replicas=1
        microk8s kubectl rollout status deploy -n "${NS}" -l app="${NS}" --timeout=80s || {
            echo "Rollout fallito"
            exit 1
        }

        """
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
      sh """
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

        microk8s kubectl -n review exec mongodb-0 -- \
          mongosh "mongodb://localhost:27017/review" --quiet \
          --eval 'const r = db.getCollection("submission-consegnate").deleteMany({}); print("deleted from review.submission-consegnate:", r.deletedCount)'

        POD=$(microk8s kubectl -n report get pods -l app=postgresdb \
          -o jsonpath='{.items[0].metadata.name}')

        microk8s kubectl -n report exec "$POD" -- \
          psql -U app -d reports -v ON_ERROR_STOP=1 \
          -c "TRUNCATE TABLE public.teacher_assignments, public.assignments, public.submissions, public.reviews RESTART IDENTITY CASCADE;"
      """
    }
    success { echo "✅ Done — MODE=${env.MODE}, SERVICE=${env.SVC}" }
    failure { echo "❌ Deploy fallito — controlla i log" }
  }
}
