pipeline {
  agent { label 'unit-test' }

  environment {
    // ---- config progetto ----
    ENV             = 'unit-test'
    CI_IMAGE_NAME   = 'assignments-pytest'
    APP_IMAGE_NAME  = 'ale175/service-assignment'     // <user>/<repo>
    DOCKERHUB_CREDS = 'dockerhub-creds'               // ID credenziale Jenkins

    // job multibranch di integrazione (folder/repo/branch)
    INTEGRATION_JOB = 'peer-review-pipeline/integration-repo/main'
  }

  stages {
    stage('Build CI Image') {
      steps {
        echo "Costruisco immagine CI…"
        sh 'docker build -t "${CI_IMAGE_NAME}" -f Dockerfile.unit .'
      }
    }

    stage('Run Python Unit Tests') {
      steps {
        echo "Eseguo test unitari…"
        sh '''
          docker run --rm \
            --user "$(id -u)":"$(id -g)" \
            -e ENV="${ENV}" \
            -v "${PWD}:/work" -w /work \
            "${CI_IMAGE_NAME}" pytest -v test/pytest
        '''
      }
    }

    // Solo se questa build è una Pull Request il cui target è "main"
    stage('Build & Push App Image (:latest) — PR→main') {
      when {
        allOf {
          changeRequest()                               // è una PR
          expression { return env.CHANGE_TARGET == 'main' }  // target main
        }
      }
      steps {
        script {
          docker.withRegistry('https://index.docker.io/v1/', DOCKERHUB_CREDS) {
            def appImage = docker.build("${APP_IMAGE_NAME}", "-f Dockerfile .")
            appImage.push("latest")                    // <- richiesto: sempre :latest
          }
        }
      }
    }
  }

  post {
    success {
      script {
        // Trigger integrazione solo se PR→main
        if (env.CHANGE_ID?.trim() && env.CHANGE_TARGET == 'main') {
          echo "Trigger del job di integrazione: ${INTEGRATION_JOB}"
          build job: INTEGRATION_JOB,
                wait: false,
                parameters: [
                  string(name: 'SERVICE_NAME',  value: 'assignment'),
                  string(name: 'TRIGGER_TYPE',  value: 'single')
                  // IMAGE_TAG non necessario: in K8s usi imagePullPolicy: Always su :latest
                ]
        } else {
          echo "Nessun trigger integrazione (non è PR→main)."
        }
      }
    }
    always {
      sh '''
        chmod -R u+rwX .pytest_cache || true
        rm -rf .pytest_cache || true
      '''
      deleteDir()
    }
    failure {
      echo "Pipeline fallita."
    }
  }
}
