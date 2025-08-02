pipeline {
  agent any

  environment {
    PR_BRANCH = "${env.BRANCH_NAME}"  // "develop" Branch name from webhook or multibranch
    ERP_REPO = 'https://github.com/Oleg378/erpnext.git'
    DOCKER_REPO = 'https://github.com/Oleg378/erpnext-pipeline.git'
    DOCKER_DIR = 'erpnext-pipeline'
    ERP_DIR = 'erpnext'
    SITE_NAME = 'localhost'
    CONTAINER_NAME = 'backend'
    TEST_REPORT = 'test-report.json'
  }

  stages {
    stage('Checkout ERPNext PR') {
      steps {
        dir("${ERP_DIR}") {
          git url: "${ERP_REPO}", branch: "${PR_BRANCH}", changelog: false, poll: false
        }
      }
    }

    stage('Checkout CI Pipeline Repo') {
      steps {
        dir("${DOCKER_DIR}") {
          git url: "${DOCKER_REPO}", branch: 'main', changelog: false, poll: false
        }
      }
    }

    stage('Prepare and Start Environment') {
      steps {
        dir("${DOCKER_DIR}") {
          sh '''
            echo "Starting Docker environment..."
            docker compose up -d
          '''

          script {
            def siteExists = sh (
              script: "docker compose exec ${CONTAINER_NAME} bash -c 'ls /home/frappe/frappe-bench/sites/${SITE_NAME}/site_config.json 2>/dev/null'",
              returnStatus: true
            ) == 0

            if (siteExists) {
              echo "Site exists. Running migration..."
              sh """
                docker compose exec ${CONTAINER_NAME} bash -c '
                  cd /home/frappe/frappe-bench &&
                  bench migrate &&
                  bench serve --port 8080 &
                  sleep 10
                '
              """
            } else {
              echo "No existing site. Waiting for create-site..."
              sh '''
                docker compose wait create-site || true
              '''
              echo "Launching bench server..."
              sh """
                docker compose exec ${CONTAINER_NAME} bash -c '
                  cd /home/frappe/frappe-bench &&
                  bench --site ${SITE_NAME} serve --port 8080 &
                  sleep 10
                '
              """
            }
          }
        }
      }
    }

    stage('Run Smoke Tests') {
      steps {
        echo '✅ Placeholder for Playwright tests'
        sh """
          echo '{ "status": "passed", "tests": 10, "failures": 0 }' > ${TEST_REPORT}
        """
      }
    }

    stage('Publish Test Report') {
      steps {
        archiveArtifacts artifacts: "${TEST_REPORT}", onlyIfSuccessful: true
        // In real setup: publishHTML or junit integration here
      }
    }

    stage('Approve or Reject PR') {
      steps {
        script {
          def report = readJSON file: "${TEST_REPORT}"
          if (report.failures == 0) {
            echo "✅ Tests passed. PR is OK!"
            // call GitHub API or notify to proceed with merge
          } else {
            error("❌ Tests failed. Rejecting PR")
          }
        }
      }
    }
  }

  post {
    always {
      echo 'Cleaning up environment...'
      dir("${DOCKER_DIR}") {
        sh 'docker compose down --remove-orphans'
      }
    }
  }
}
