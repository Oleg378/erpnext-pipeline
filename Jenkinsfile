pipeline {
  agent any

  parameters {
    string(name: 'PR_BRANCH', defaultValue: 'version-15', description: 'Branch to test (e.g., feature/my-fix)')
    string(name: 'GIT_COMMIT', defaultValue: '', description: 'Git commit SHA to report status to; this is not empty if trigger source is GitHub')
    string(name: 'LABELS', defaultValue: '', description: 'defines what tests to trigger; empty value means full regression; ')
    string(name: 'JIRA_ISSUE_KEY', defaultValue: '', description: 'This is not empty if trigger source is JIRA')
  }

  environment {
    ERP_REPO = 'https://github.com/Oleg378/erpnext.git'
    DOCKER_REPO = 'https://github.com/Oleg378/erpnext-pipeline.git'
    TEST_AUTOMATION_REPO = 'https://github.com/Oleg378/erpnext-test-automation'
    DOCKER_DIR = 'erpnext-pipeline'
    TEST_DIR = 'test-automation'
    ERP_DIR = 'erpnext'
    SITE_NAME = 'localhost'
    CONTAINER_NAME = 'backend'
    TEST_REPORT = 'test-report.json'
    GITHUB_TOKEN = credentials('github-status-token')
    JIRA_TOKEN = credentials('jira-status-token')
    JIRA_USER_EMAIL = credentials('jira-email')
  }

  stages {
    stage('Detect trigger source') {
      steps {
        script {
          if (params.GIT_COMMIT?.trim()) {
            env.TRIGGER_SOURCE = 'github'
          } else if (params.JIRA_ISSUE_KEY?.trim()) {
            env.TRIGGER_SOURCE = 'jira'
          } else {
            env.TRIGGER_SOURCE = 'manual'
          }
            echo "Detected trigger source: ${env.TRIGGER_SOURCE}"
        }
      }
    }

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

    stage('Checkout Test Automation Repo') {
      steps {
        dir("${TEST_DIR}") {
          git url: "${TEST_AUTOMATION_REPO}", branch: 'main', changelog: false, poll: false
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
                  bench build --force &&
                  bench serve --port 8080 &
                  sleep 30
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
                  sleep 30
                '
              """
            }

            // Get Docker network for test containers
            sh '''
              NETWORK=$(docker network ls --filter name=frappe -q | head -1)
              echo "DOCKER_NETWORK=$NETWORK" > /tmp/network.env
            '''

            def networkProps = readProperties file: '/tmp/network.env'
            env.DOCKER_NETWORK = networkProps.DOCKER_NETWORK
            echo "Docker network for tests: ${env.DOCKER_NETWORK}"
          }
        }
      }
    }

    stage('Run tests in Playwright container') {
      steps {
        dir("${TEST_DIR}") {
          script {
            echo "Running tests in Playwright container..."
            echo "Labels: ${params.LABELS}"
            def testTags = ''
            def testGroups = [] as Set // run smoke by default
            def labels = []
            if (params.LABELS) {
              labels = params.LABELS.split(',')
            }
            if (labels.contains('api')) {
              testGroups.add('@api')
            }
            if (labels.contains('flows')) {
               testGroups.add('@sales')
            }
            if (labels.contains('smoke')) {
               testGroups.add('@api')
               testGroups.add('@sessions')
            }
            if (labels.contains('full-regression')) {
               testGroups.clear()  // Empty the set for full regression
            }
            if (!testGroups.isEmpty()) {
              testTags = ' -g \\"' + testGroups.collect { "${it}" }.join('|') + '\\"'
            }


            sh """
              echo "testTags: ${testTags}"
              docker run --rm \
                --network \${DOCKER_NETWORK} \
                -v \$(pwd):/app \
                -w /app \
                -e BASE_URL=http://backend:8080 \
                -e HEADLESS=true \
                mcr.microsoft.com/playwright:v1.58.0-jammy \
                bash -c "
                  echo '=== Container Environment ==='

                  echo '\n=== Installing dependencies ==='
                  npm ci --only=production

                  echo '\n=== Actual Playwright version ==='
                  npx playwright --version

                  echo '\n=== Running setup tests ==='
                  npx playwright test --project=chromium -g @setup

                  echo '\n=== Running main test suite ==='
                  npx playwright test --project=chromium${testTags}

                  echo '\nTests completed'
                "
            """
            def lastRun = readJSON file: 'test-results/.last-run.json'
            env.TEST_STATUS = lastRun.status
            echo "Test status: ${env.TEST_STATUS}"
          }
        }
      }
    }

    stage('Generate report in tobix container') {
      steps {
        dir("${TEST_DIR}") {
          script {
            echo "Generating report in Allure container..."
            sh '''
                REPORT_NAME="allure-report-$(date +%Y-%m-%d_%H-%M-%S)"

                docker run --rm \
                    -v $(pwd)/allure-results:/allure-results \
                    -v $(pwd):/app \
                    tobix/allure-cli:latest \
                    generate /allure-results -o "/app/${REPORT_NAME}" --clean

                if [ -d "${REPORT_NAME}" ]; then
                    echo "Report generated successfully"
                    echo "REPORT_FOLDER=${REPORT_NAME}" > report-info.env
                else
                    echo "Report generation failed"
                    echo "REPORT_FOLDER=unknown" > report-info.env
                    exit 1
                fi
            '''
            // Save result
            def reportInfo = readProperties file: 'report-info.env'
            env.REPORT_FOLDER = reportInfo.REPORT_FOLDER
            echo "Report folder: ${env.REPORT_FOLDER}"
          }
        }
      }
    }
    stage('Publish Test Report to GitHub') {
      steps {
        dir("${TEST_DIR}") {
          script {
            echo "Publishing report to GitHub Pages..."
            def reportFolder = env.REPORT_FOLDER
            withCredentials([
              string(credentialsId: 'github-allure-report-token', variable: 'GH_TOKEN')
              ]) {
              sh """
                rm -rf /tmp/allure-reports

                git clone \
                  --filter=blob:none \
                  --no-checkout \
                  https://github.com/Oleg378/allure-reports.git /tmp/allure-reports

                cd /tmp/allure-reports
                git sparse-checkout init --cone
                git sparse-checkout set docs
                git checkout main

                mkdir -p docs
                cp -r "\${WORKSPACE}/${TEST_DIR}/${reportFolder}" docs/

                git config user.email "jenkins@ci"
                git config user.name "Jenkins CI"

                git add docs/${reportFolder}
                git commit -m "Add test report ${reportFolder}" || echo "No changes to commit"

                git push https://x-access-token:\${GH_TOKEN}@github.com/Oleg378/allure-reports.git main
              """
              }
            env.REPORT_URL = "https://oleg378.github.io/allure-reports/${reportFolder}/"
            env.REPORT_FOLDER = reportFolder
            echo "Report URL: ${env.REPORT_URL}"
          }
        }
      }
    }

    stage('Report Status to GitHub PR') {
      when {
        expression { env.TRIGGER_SOURCE == 'github' }
      }
      steps {
        script {
          // Validate parameters
          if (!params.GIT_COMMIT?.trim()) {
            echo "Commit SHA parameter (GIT_COMMIT) is required"
            return
          }
          if (!params.GIT_COMMIT.matches(/^[a-f0-9]{40}$/)) {
            echo "Invalid GIT_COMMIT format (expected 40-character SHA)"
            return
          }
          if (!env.REPORT_URL?.trim() || env.REPORT_URL == "https://oleg378.github.io/allure-reports/unknown/") {
            echo "Report URL is invalid or unknown, skipping status update"
            return
          }

          // Convert test status to GitHub status
          def status = env.TEST_STATUS == 'passed' ? 'success' : 'failure'
          def description = "Playwright tests ${env.TEST_STATUS}"

          echo "  Reporting status to GitHub..."
          echo "  Commit: ${params.GIT_COMMIT}"
          echo "  Status: ${status}"
          echo "  Report: ${env.REPORT_URL}"

          def apiUrl = "https://api.github.com/repos/Oleg378/erpnext/statuses/${params.GIT_COMMIT}"
          def payload = """
          {
            "state": "${status}",
            "target_url": "${env.REPORT_URL}",
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
                [name: 'Authorization', value: "token " + GITHUB_TOKEN]
              ],
              requestBody: payload
            )
            echo "Status reported to GitHub: ${status}"
          } catch (Exception e) {
            echo "Failed to report status to GitHub: ${e.getMessage()}"
          }
        }
      }
    }
    stage('Report Status to Jira') {
      when {
        expression { env.TRIGGER_SOURCE == 'jira' }
      }
      steps {
        script {
          // Validate parameters
          if (!params.JIRA_ISSUE_KEY?.trim()) {
            echo "JIRA_ISSUE_KEY missing, skipping Jira reporting"
            return
          }

          if (!env.REPORT_URL?.trim() || env.REPORT_URL.endsWith('/unknown/')) {
            echo "Report URL is invalid or unknown, skipping Jira reporting"
            return
          }

          def status = env.TEST_STATUS == 'passed' ? 'PASSED' : 'FAILED'
          def emoji  = env.TEST_STATUS == 'passed' ? '✅' : '❌'

          def basicAuth = "Basic " + "${JIRA_USER_EMAIL}:${JIRA_TOKEN}".bytes.encodeBase64().toString()

          def commentPayload = groovy.json.JsonOutput.toJson([
            body: [
              type: 'doc',
              version: 1,
              content: [
                [
                  type: 'paragraph',
                  content: [
                    [
                      type: 'text',
                      text: "${emoji} Automated tests completed\n" +
                            "Status: ${status}\n" +
                            "Branch: ${params.PR_BRANCH}\n" +
                            "Labels: ${params.LABELS ?: 'full-regression'}\n"
                    ],
                    [
                      type: 'text',
                      text: "Report",
                      marks: [
                        [type: 'link', attrs: [href: env.REPORT_URL]]
                      ]
                    ]
                  ]
                ]
              ]
            ]
          ])

          echo "Reporting status to Jira issue ${params.JIRA_ISSUE_KEY}"

          try {
            httpRequest(
              httpMode: 'POST',
              url: "https://kalman-oleg.atlassian.net/rest/api/3/issue/${params.JIRA_ISSUE_KEY}/comment",
              contentType: 'APPLICATION_JSON',
              customHeaders: [
                [name: 'Authorization', value: basicAuth],
                [name: 'Accept', value: 'application/json']
              ],
              requestBody: commentPayload
            )
            echo "Comment posted to Jira"

            // Hardcoded transition ID 31 represents status Done
            def transitionPayload = groovy.json.JsonOutput.toJson([
              transition: [id: "31"]
            ])
            httpRequest(
              httpMode: 'POST',
              url: "https://kalman-oleg.atlassian.net/rest/api/3/issue/${params.JIRA_ISSUE_KEY}/transitions",
              contentType: 'APPLICATION_JSON',
              customHeaders: [
                [name: 'Authorization', value: basicAuth],
                [name: 'Accept', value: 'application/json']
              ],
              requestBody: transitionPayload
            )
            echo "Jira issue transitioned to Done"
          } catch (Exception e) {
            echo "Failed to report status to Jira: ${e.getMessage()}"
          }
        }
      }
    }
  }

  post {
    always {
      echo "========================================"
      echo " Test Status: ${env.TEST_STATUS ?: 'UNKNOWN'}"
      echo " Report Folder: ${env.REPORT_FOLDER ?: 'NOT_GENERATED'}"
      echo " Report URL: ${env.REPORT_URL ?: 'NOT_AVAILABLE'}"
      echo "========================================"

      // 1. Delete test-results and allure-results from TEST_DIR
      dir("${TEST_DIR}") {
        sh '''
          echo " Cleaning test artifacts..."
          rm -rf test-results allure-results 2>/dev/null || true
          echo " Test artifacts removed"
        '''
      }

      // 2. Delete tmp
      sh '''
        echo "️  Cleaning temporary files..."
        rm -rf /tmp/allure-reports* /tmp/network.env 2>/dev/null || true
        echo " Temporary files removed"
      '''

      // 3. Stop docker compose
      dir("${DOCKER_DIR}") {
        sh '''
          echo " Stopping Docker containers..."
          docker compose down 2>/dev/null || echo "Docker already stopped"
          echo " Docker containers stopped"
        '''
      }
    }
  }
}