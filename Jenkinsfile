pipeline {
  agent any

  parameters {
    string(name: 'PR_BRANCH', defaultValue: 'version-15', description: 'Branch to test (e.g., feature/my-fix)')
    string(name: 'GIT_COMMIT', defaultValue: '', description: 'Git commit SHA to report status to')
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
    GITHUB_REPORT_TOKEN = credentials('github-allure-report-token')
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
            echo "🚀 Running tests in official Playwright container..."

            sh '''
              docker run --rm \
                --network ${DOCKER_NETWORK} \
                -v $(pwd):/app \
                -w /app \
                -e BASE_URL=http://backend:8080 \
                -e HEADLESS=true \
                mcr.microsoft.com/playwright:v1.54.2-jammy \
                bash -c "
                  echo '=== Container Environment ==='
                  echo 'Node.js: ' \$(node --version)
                  echo 'npm: ' \$(npm --version)
                  playwright --version

                  echo '\n=== Installing dependencies ==='
                  npm ci

                  echo '\n=== Actual Playwright version ==='
                  npx playwright --version

                  echo '\n=== Running setup tests ==='
                  npx playwright test --project=chromium -g @setup || echo 'Setup tests may have warnings'

                  echo '\n=== Running main test suite ==='
                  npx playwright test --project=chromium -g @api

                  echo '\n✅ Tests completed'
                "
            '''
            def lastRun = readJSON file: 'test-results/.last-run.json'
            env.TEST_STATUS = lastRun.status
            echo "Test status: ${env.TEST_STATUS}"
          }
        }
      }
    }

    stage('Generate report in Allure container') {
      steps {
        dir("${TEST_DIR}") {
          script {
            echo "📊 Generating report in Allure container..."

            // debug:
            sh 'ls -la allure-results/ 2>/dev/null || echo "No allure-results folder"'
            sh '''
                REPORT_NAME="allure-report-$(date +%Y-%m-%d_%H-%M-%S)"

                docker run --rm \
                    -v $(pwd)/allure-results:/allure-results \
                    -v $(pwd):/app \
                    tobix/allure-cli:latest \
                    generate /allure-results -o "/app/${REPORT_NAME}" --clean

                if [ -d "${REPORT_NAME}" ]; then
                    echo "✅ Report generated successfully"
                    echo "REPORT_FOLDER=${REPORT_NAME}" > report-info.env
                else
                    echo "❌ Report generation failed"
                    echo "REPORT_FOLDER=unknown" > report-info.env
                    exit 1
                fi
            '''

            // Check result
            if (fileExists("report-info.env")) {
              def reportInfo = readProperties file: 'report-info.env'
              env.REPORT_FOLDER = reportInfo.REPORT_FOLDER
              echo "📁 Report folder: ${env.REPORT_FOLDER}"

              // Also verify the folder exists locally
              if (env.REPORT_FOLDER != "unknown" && !fileExists(env.REPORT_FOLDER)) {
                echo "⚠️ Report folder ${env.REPORT_FOLDER} doesn't exist locally"
                env.REPORT_FOLDER = "unknown"
              }
            } else {
              env.REPORT_FOLDER = "unknown"
              echo "⚠️ report-info.env not created"
            }
          }
        }
      }
    }

    stage('Publish Test Report to GitHub') {
      steps {
        dir("${TEST_DIR}") {
          script {
            echo "🚀 Publishing report to GitHub Pages..."

            // Check if report was actually generated
            def reportFolder = env.REPORT_FOLDER ?: "unknown"
            if (reportFolder == "unknown") {
              // Try to find any report folder
              reportFolder = sh(
                script: 'find . -maxdepth 1 -name "allure-report-*" -type d | head -1 | xargs basename 2>/dev/null || echo "unknown"',
                returnStdout: true
              ).trim()
            }

            if (reportFolder == "unknown") {
              echo "⚠️ No report folder found, skipping GitHub Pages upload"
              env.REPORT_URL = "https://oleg378.github.io/allure-reports/"
              return
            }

            echo "📁 Using report folder: ${reportFolder}"

            sh """
              # Clone reports repo (shallow)
              git clone --depth 1 https://github.com/Oleg378/allure-reports.git /tmp/allure-reports
              cd /tmp/allure-reports

              # Create docs directory
              mkdir -p docs

              # Copy new report
              echo "Copying report: ${reportFolder}"
              cp -r "\${WORKSPACE}/${TEST_DIR}/${reportFolder}" "docs/" 2>/dev/null || echo "Report copy failed"

              # Commit and push
              git config user.email "jenkins@ci"
              git config user.name "Jenkins CI"
              git add .
              git commit -m "Add test report ${reportFolder}"
              git push https://x-access-token:${GITHUB_REPORT_TOKEN}@github.com/Oleg378/allure-reports.git


              echo "✅ Report published to GitHub"
            """

            // Generate report URL
            env.REPORT_URL = "https://oleg378.github.io/allure-reports/${reportFolder}/"
            env.REPORT_FOLDER = reportFolder
            echo "🌐 Report URL: ${env.REPORT_URL}"

            // Create test report JSON
            def reportData = [
              status: env.TEST_STATUS,
              report_url: env.REPORT_URL,
              timestamp: new Date().format('yyyy-MM-dd HH:mm:ss'),
              folder: reportFolder
            ]
            writeJSON file: "${TEST_REPORT}", json: reportData
          }
        }
      }
    }

    stage('Report Status to GitHub PR') {
      steps {
        script {
          // Validate parameters
          if (!params.GIT_COMMIT?.trim()) {
            error("❌ Commit SHA parameter (GIT_COMMIT) is required")
          }
          if (!params.GIT_COMMIT.matches(/^[a-f0-9]{40}$/)) {
            error("❌ Invalid GIT_COMMIT format (expected 40-character SHA)")
          }
          if (!env.REPORT_URL?.trim() || env.REPORT_URL == "https://oleg378.github.io/allure-reports/reports/unknown/") {
            echo "⚠️ Report URL is invalid or unknown, skipping status update"
            return
          }

          // Convert test status to GitHub status
          def status = env.TEST_STATUS == 'passed' ? 'success' : 'failure'
          def description = "Playwright tests ${env.TEST_STATUS}"

          echo "📤 Reporting status to GitHub..."
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
            echo "✅ Status reported to GitHub: ${status}"
          } catch (Exception e) {
            echo "⚠️ Failed to report status to GitHub: ${e.getMessage()}"
            // Don't fail the pipeline - status reporting is secondary
          }
        }
      }
    }
  }

  post {
    always {
      echo "========================================"
      echo "📊 Test Status: ${env.TEST_STATUS ?: 'UNKNOWN'}"
      echo "📁 Report Folder: ${env.REPORT_FOLDER ?: 'NOT_GENERATED'}"
      echo "🌐 Report URL: ${env.REPORT_URL ?: 'NOT_AVAILABLE'}"
      echo "========================================"

      // 1. Delete test-results and allure-results from TEST_DIR
      dir("${TEST_DIR}") {
        sh '''
          echo "🧹 Cleaning test artifacts..."
          rm -rf test-results allure-results 2>/dev/null || true
          echo "✅ Test artifacts removed"
        '''
      }

      // 2. Delete tmp
      sh '''
        echo "🗑️  Cleaning temporary files..."
        rm -rf /tmp/allure-reports* /tmp/network.env 2>/dev/null || true
        echo "✅ Temporary files removed"
      '''

      // 3. Stop docker compose
      dir("${DOCKER_DIR}") {
        sh '''
          echo "🐳 Stopping Docker containers..."
          docker compose down 2>/dev/null || echo "Docker already stopped"
          echo "✅ Docker containers stopped"
        '''
      }
    }
  }
}