
pipeline {
  agent { label 'windows' }

  options {
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '20'))
  }

  parameters {
    // Toggle to use Deployer CLI when you install it later
    booleanParam(name: 'USE_DEPLOYER', defaultValue: false, description: 'Use Software AG Deployer (requires deployer.bat)')
    // Set the REST path that you actually expose (change default if you use RAD path)
    stringParam(name: 'API_PATH', defaultValue: '/employee/insertEmployee',
                description: 'REST path to health-check (e.g., /employee/insertEmployee or /rad/employee.resources:rad_employee/insertEmployee)')
  }

  environment {
    // 🔁 Adjust to your actual Integration Server home if needed
    IS_HOME       = 'C:/Users/vkraft_Auni/Documents/w/IntegrationServer'
    PACKAGE_NAME  = 'employee.zip'          // This zip must be at repo root
    PACKAGE_PATH  = "${WORKSPACE}\\${PACKAGE_NAME}"
    PACKAGES_DIR  = "${IS_HOME}\\packages"  // Target for manual deploy
    API_HOST      = 'localhost'             // Windows agent talks to local IS
    API_PORT      = '5555'
    API_URL       = "http://${API_HOST}:${API_PORT}${params.API_PATH}"
  }

  stages {
    stage('Clean Workspace') {
      steps {
        echo "🧹 Cleaning workspace..."
        deleteDir()
      }
    }

    stage('Checkout SCM') {
      steps {
        echo "📥 Checking out from Git..."
        checkout scm
        bat 'dir'
      }
    }

    stage('Deploy (Manual Unzip)') {
      when { expression { !params.USE_DEPLOYER } }
      steps {
        echo "📦 Manual deploy (no Deployer installed): ${env.PACKAGE_NAME}"
        // Expand the package zip straight under Integration Server/packages
        bat """
          IF NOT EXIST "${PACKAGE_PATH}" (
            echo Package zip not found: ${PACKAGE_PATH}
            exit /b 1
          )
          echo Expanding %PACKAGE_PATH% into %PACKAGES_DIR% ...
          powershell -NoProfile -Command "Expand-Archive -Force -LiteralPath '${PACKAGE_PATH}' -DestinationPath '${PACKAGES_DIR}'"
          echo ✅ Manual deploy complete. If package name changed, reload it in IS Admin.
        """
      }
    }

    stage('Deploy (Deployer CLI)') {
      when { expression { params.USE_DEPLOYER } }
      steps {
        echo "🚀 Deploy via Deployer CLI"
        bat """
          IF NOT EXIST "${IS_HOME}\\deployer\\bin\\deployer.bat" (
            echo Deployer not found at: ${IS_HOME}\\deployer\\bin\\deployer.bat
            exit /b 1
          )
          "${IS_HOME}\\deployer\\bin\\deployer.bat" -silent -package "${PACKAGE_PATH}"
        """
      }
    }

    stage('Verify API Health') {
      steps {
        script {
          echo "🔎 Health-check: POST ${env.API_URL}"
          def ready = false
          for (int i = 1; i <= 5; i++) {
            sleep time: 10, unit: 'SECONDS'
            // Use PowerShell Invoke-WebRequest for clean status code
            def status = bat(
              script: """powershell -NoProfile -Command "(Invoke-WebRequest -Method POST -Uri '${API_URL}' -UseBasicParsing).StatusCode" """,
              returnStdout: true
            ).trim()
            echo "Attempt ${i}: HTTP ${status}"
            if (status == '200' || status == '202') { ready = true; break }
          }
          if (!ready) {
            error "❌ API FAILED: ${env.API_URL} not ready after 5 attempts"
          } else {
            echo "✅ API healthy"
          }
        }
      }
    }
  }

  post {
    always {
      echo "🧽 Cleaning workspace..."
      cleanWs()
    }
    success { echo "🎉 Pipeline succeeded!" }
    failure { echo "❌ Pipeline failed. Check logs above." }
  }
}
