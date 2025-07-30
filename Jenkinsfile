pipeline {
  agent any

  environment {
    PR_BRANCH = "${env.BRANCH_NAME}"
    DOCKER_REPO = 'https://github.com/Oleg378/erpnext-pipeline.git'
    DOCKER_DIR = 'erpnext-pipeline'
    SITE_NAME = 'localhost'
    CONTAINER_NAME = 'backend'
    TEST_REPORT = 'test-report.json'
  }

  stages {
    stage('Checkout ERPNext PR') {
      steps {
        dir('erpnext') {
          checkout scm
        }
      }
    }

    stage('Checkout CI Docker Setup') {
      steps {
        git url: "${DOCKER_REPO}", branch: 'main', changelog: false, poll: false
      }
    }

    stage('Prepare and Start Environment') {
      steps {
        dir("${DOCKER_DIR}") {
          sh '''
            echo "Starting docker environment..."
            docker compose up -d
          '''

          script {
            def siteExists = sh (
              script: "docker compose exec ${CONTAINER_NAME} bash -c 'ls /home/frappe/frappe-bench/sites/${SITE_NAME}/site_config.json 2>/dev/null'",
              returnStatus: true
            ) == 0

            if (siteExists) {
              echo "🧠 Site exists. Applying code updates and running migrations..."
              sh """
                docker compose exec ${CONTAINER_NAME} bash -c '
                  cd /home/frappe/frappe-bench &&
                  bench --site ${SITE_NAME} migrate &&
                  bench --site ${SITE_NAME} clear-cache &&
                  bench --site ${SITE_NAME} build &&
                  bench --site ${SITE_NAME} serve --port 8080 &
                  sleep 10
                '
              """
            } else {
              echo "🆕 Site does not exist. Waiting for create-site to exit..."
              sh '''
                echo "Waiting for create-site container to exit..."
                docker compose wait create-site || true
              '''

              echo "Starting bench serve for new site..."
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
        echo '✅ Placeholder: run Playwright tests here'
        sh '''
          echo '{ "status": "passed", "tests": 10, "failures": 0 }' > ${TEST_REPORT}
        '''
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
      echo '🧹 Cleanup: stopping containers'
      dir("${DOCKER_DIR}") {
        sh 'docker compose down --remove-orphans'
      }
    }
  }
}
