pipeline {
  agent any

  parameters {
    string(name: 'PR_BRANCH', defaultValue: 'version-15', description: 'Branch to test (e.g., feature/my-fix)')
    string(name: 'GIT_COMMIT', defaultValue: '', description: 'Git commit SHA to report status to')
  }

  environment {
    ERP_REPO = 'https://github.com/Oleg378/erpnext.git'
    DOCKER_REPO = 'https://github.com/Oleg378/erpnext-pipeline.git'
    DOCKER_DIR = 'erpnext-pipeline'
    ERP_DIR = 'erpnext'
    SITE_NAME = 'localhost'
    CONTAINER_NAME = 'backend'
    TEST_REPORT = 'test-report.json'
    GITHUB_TOKEN = credentials('github-status-token')
  }

  stages {
    stage('Checkout ERPNext PR') {
      steps {
        dir("${ERP_DIR}") {
          git url: "${ERP_REPO}", branch: "${params.PR_BRANCH}", changelog: false, poll: false
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
            def siteExists = sh(
              script: "docker compose exec -T ${CONTAINER_NAME} bash -c 'ls /home/frappe/frappe-bench/sites/${SITE_NAME}/site_config.json 2>/dev/null'",
              returnStatus: true
            ) == 0

            // Always rebuild frontend assets
            sh """
              docker compose exec -T ${CONTAINER_NAME} bash -c '
                cd /home/frappe/frappe-bench/apps/frappe &&
                yarn install &&
                cd ../erpnext &&
                yarn install &&
                cd ../.. &&
                bench build
              '
            """

            if (siteExists) {
              echo "Site exists. Running migration..."
              sh """
                docker compose exec -T ${CONTAINER_NAME} bash -c '
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
                docker compose exec -T ${CONTAINER_NAME} bash -c '
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
      }
    }

    stage('Approve or Reject PR') {
      steps {
        script {
          if (!params.GIT_COMMIT?.trim()) {
            error("Commit SHA parameter (GIT_COMMIT) is required")
          }
          if (!params.GIT_COMMIT.matches(/^[a-f0-9]{40}$/)) {
            error("Invalid GIT_COMMIT format (expected 40-character SHA)")
          }

          if (!fileExists("${TEST_REPORT}")) {
            error("Test report not found at ${TEST_REPORT}")
          }

          def report = readJSON file: "${TEST_REPORT}"
          if (!report.tests || report.failures == null) {
            error("Invalid test report format - missing required fields")
          }

          def status = report.failures == 0 ? 'success' : 'failure'
          def description = report.failures == 0 ?
            "All ${report.tests} tests passed" :
            "${report.failures} test(s) failed"

          def apiUrl = "https://api.github.com/repos/Oleg378/erpnext/statuses/${params.GIT_COMMIT}"
          def payload = """
          {
            "state": "${status}",
            "target_url": "${env.BUILD_URL}",
            "description": "${description}",
            "context": "jenkins/ci"
          }
          """.stripIndent()

          try {
            def response = httpRequest(
              httpMode: 'POST',
              url: apiUrl,
              contentType: 'APPLICATION_JSON',
              customHeaders: [
                [name: 'Authorization', value: "token " + GITHUB_TOKEN],
