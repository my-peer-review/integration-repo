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

    stage('Recreate namespaces & base manifests') {
      steps {
        sh '''
          set -e
          if [ "$MODE" = "push" ]; then
            echo "Applying manifests…"
            "$MK8S" kubectl apply -f "$K8S_DIR/namespaces.yaml"
            "$MK8S" kubectl apply -R -f "$K8S_DIR/databases"
            "$MK8S" kubectl apply -R -f "$K8S_DIR/services"
            "$MK8S" kubectl apply -f "$K8S_DIR/config.yaml"
            "$MK8S" kubectl apply -f "$K8S_DIR/secrets.yaml"
            "$MK8S" kubectl apply -f "$K8S_DIR/ingress.yaml"
          else
            # rollout del solo microservizio
            "$MK8S" kubectl apply -R -f "$K8S_DIR/services/$SVC"
          fi
        '''
      }
    }

    stage('Cancellazione deployments') {
      steps {
        sh '''
          set -e
          if [ "$MODE" = "push" ]; then
            for ns in $NS_LIST; do
              echo "Cancellazione deployment in $ns"
              "$MK8S" kubectl delete deploy --all -n "$ns" --ignore-not-found || true
            done
          elif [ "$MODE" = "single" ]; then
            echo "Cancellazione deployment del servizio $SVC"
            "$MK8S" kubectl -n "$SVC" delete deploy "$SVC" --ignore-not-found || true
          else
            echo "MODE non supportato: $MODE"; exit 1
          fi
        '''
      }
    }

    stage('Rollout SINGLE all Deployments') {
      when { expression { env.MODE == 'push' } }
      steps {
        sh '''
          set -e
          for ns in $NS_LIST; do
            deps=$("$MK8S" kubectl -n "$ns" get deploy -o name || true)
            [ -z "$deps" ] && { echo "Nessun deployment in $ns"; continue; }

            echo "$deps" | while read -r d; do
              echo "rollout $d in $ns"
              "$MK8S" kubectl -n "$ns" rollout status "$d" --timeout=180s
              # attende che diventi Available (usa le readinessProbe)
              "$MK8S" kubectl -n "$ns" wait "$d" --for=condition=available --timeout=180s
            done
          done
        '''
      }
    }

    stage('Rollout SINGLE service (trigger=single)') {
      when { expression { env.MODE == 'single' } }
      steps {
        echo "Rollout del servizio: ${env.SVC}"
        sh '''
          set -e
          NS="$SVC"
          "$MK8S" kubectl -n "$NS" rollout restart "deploy/$SVC"
          "$MK8S" kubectl -n "$NS" rollout status  "deploy/$SVC" --timeout=180s
          "$MK8S" kubectl -n "$NS" wait "deploy/$SVC" --for=condition=available --timeout=180s
        '''
      }
    }

    stage('Run API Tests with Newman user-manager') {
      when { expression { env.SVC == 'user-manager' && env.MODE == "push" } }
      steps {
        sh '''
          newman run ./test/postman/Autenticazione.postman_collection.json \
            -e ./test/postman/postman-env.postman_environment.json \
            --reporters cli,json
        '''
      }
    }

    stage('Run API Tests with Newman Assignments') {
      when { expression { env.SVC == 'assignment' && env.MODE == "push" } }
      steps {
        sh '''
          newman run ./test/postman/Assignments.postman_collection.json \
            -e ./test/postman/postman-env.postman_environment.json \
            --reporters cli,json
        '''
      }
    }

    stage('Run API Tests with Newman Submissions') {
      when { expression { env.SVC == 'submission' && env.MODE == "push" } }
      steps {
        sh '''
          newman run ./test/postman/Submissions.postman_collection.json \
            -e ./test/postman/postman-env.postman_environment.json \
            --reporters cli,json
        '''
      }
    }

    stage('Run API Tests with Newman review') {
      when { expression { env.SVC == 'review' && env.MODE == "push" } }
      steps {
        sh '''
          newman run ./test/postman/Review.postman_collection.json \
            -e ./test/postman/postman-env.postman_environment.json \
            --reporters cli,json
        '''
      }
    }

    stage('Run API Tests with Newman Report') {
      when { expression { env.SVC == 'Report' && env.MODE == "push" } }
      steps {
        sh '''
          newman run ./test/postman/Report.postman_collection.json \
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

        microk8s kubectl -n review exec mongodb-0 -- \
          mongosh "mongodb://localhost:27017/review" --quiet \
          --eval 'const r = db.getCollection("submission-consegnate").deleteMany({}); print("deleted from review.submission-consegnate:", r.deletedCount)'

        POD=$(microk8s kubectl -n report get pods -l app=postgresdb \
          -o jsonpath='{.items[0].metadata.name}')

        microk8s kubectl -n report exec "$POD" -- \
          psql -U app -d reports -v ON_ERROR_STOP=1 \
          -c "TRUNCATE TABLE public.teacher_assignments, public.assignments, public.submissions, public.reviews RESTART IDENTITY CASCADE;"
      '''
    }
    success { echo "✅ Done — MODE=${env.MODE}, SERVICE=${env.SVC}" }
    failure { echo "❌ Deploy fallito — controlla i log" }
  }
}
