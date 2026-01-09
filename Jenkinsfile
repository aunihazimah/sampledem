
pipeline {
  agent { label 'windows' }

  options {
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '20'))
  }

  parameters {
    booleanParam(name: 'USE_DEPLOYER', defaultValue: false, description: 'Use Software AG Deployer (requires deployer.bat)')
    string(name: 'API_PATH', defaultValue: '/rad/employee.resources:rad_employee/insertEmployee',
           description: 'Full invoke path (e.g., /employee/insertEmployee or /rad/employee.resources:rad_employee/insertEmployee)')
    string(name: 'POST_BODY', defaultValue: '{"code":"E001","name":"Alice","price":"29.90","category":"01","visible":"1"}',
           description: 'Request body to send to the POST API in health check')
    string(name: 'CONTENT_TYPE', defaultValue: 'application/json',
           description: 'Content-Type for POST (application/json or application/x-www-form-urlencoded)')
    string(name: 'PACKAGE_ZIP', defaultValue: 'employee.zip', description: 'Zip file at repo root to deploy')
    string(name: 'PACKAGE_FOLDER', defaultValue: 'employee', description: 'Folder name under IS\\packages after unzip (for reload)')
    string(name: 'IS_HOST', defaultValue: 'localhost', description: 'Integration Server host')
    string(name: 'IS_PORT', defaultValue: '5555', description: 'Integration Server port')
    booleanParam(name: 'USE_AUTH', defaultValue: true, description: 'Use Basic Auth (Jenkins cred: is-basic)')
  }

  environment {
    IS_HOME       = 'C:\\Users\\vkraft_Auni\\Documents\\w\\IntegrationServer'
    PACKAGES_DIR  = "${IS_HOME}\\packages"
    PACKAGE_NAME  = "${params.PACKAGE_ZIP}"
    PACKAGE_PATH  = "${WORKSPACE}\\${params.PACKAGE_ZIP}"
    API_URL       = "http://${params.IS_HOST}:${params.IS_PORT}${params.API_PATH}"
    RELOAD_URL    = "http://${params.IS_HOST}:${params.IS_PORT}/invoke/wm.server.packages:reloadPackage?name=${params.PACKAGE_FOLDER}"
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
        bat """
          IF NOT EXIST "${PACKAGE_PATH}" (
            echo Package zip not found: ${PACKAGE_PATH}
            exit /b 1
          )
          echo Expanding "%PACKAGE_PATH%" into "%PACKAGES_DIR%" ...
          powershell -NoProfile -Command "Expand-Archive -Force -LiteralPath '${PACKAGE_PATH}' -DestinationPath '${PACKAGES_DIR}'"
          echo ✅ Unzip complete.
          IF NOT EXIST "${PACKAGES_DIR}\\${params.PACKAGE_FOLDER}" (
            echo WARNING: Expected folder "%PACKAGES_DIR%\\${params.PACKAGE_FOLDER}" not found. Verify zip structure.
          ) ELSE (
            echo Found: ${PACKAGES_DIR}\\${params.PACKAGE_FOLDER}
          )
        """
      }
    }

    stage('Deploy (Deployer CLI)') {
      when { expression { params.USE_DEPLOYER } }
      steps {
        echo "🚀 Deploy via Deployer CLI (placeholder)"
        bat """
          IF NOT EXIST "${IS_HOME}\\deployer\\bin\\deployer.bat" (
            echo Deployer not found at: ${IS_HOME}\\deployer\\bin\\deployer.bat
            exit /b 1
          )
          echo NOTE: Proper Deployer CLI requires project/target setup. Showing --help:
          "${IS_HOME}\\deployer\\bin\\deployer.bat" --help
        """
      }
    }

    stage('Reload Package in IS') {
      steps {
        script {
          echo "🔄 Reloading package: ${params.PACKAGE_FOLDER}"
          withCredentials([usernamePassword(credentialsId: 'is-basic', usernameVariable: 'IS_USER', passwordVariable: 'IS_PASS')]) {
            def cmd
            if (params.USE_AUTH) {
              cmd = """powershell -NoProfile -Command ^
                "$pair = '${IS_USER}:${IS_PASS}'; ^
                 $b64 = [Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes($pair)); ^
                 (Invoke-WebRequest -Method GET -Uri '${RELOAD_URL}' -Headers @{ Authorization = 'Basic ' + $b64 } -UseBasicParsing).StatusCode" """
            } else {
              cmd = """powershell -NoProfile -Command "(Invoke-WebRequest -Method GET -Uri '${RELOAD_URL}' -UseBasicParsing).StatusCode" """
            }
            def status = bat(returnStdout: true, script: cmd).trim()
            echo "Reload status: HTTP ${status}"
            if (!(status == '200' || status == '202')) {
              echo "⚠️ Package reload returned ${status}. Ensure credentials/ACLs allow reload."
            }
          }
        }
      }
    }

    stage('Verify API Health (POST)') {
      steps {
        script {
          echo "🔎 Health-check: POST ${env.API_URL}"
          def ready = false
          withCredentials([usernamePassword(credentialsId: 'is-basic', usernameVariable: 'IS_USER', passwordVariable: 'IS_PASS')]) {
            for (int i = 1; i <= 5; i++) {
              sleep time: 8, unit: 'SECONDS'
              def cmd
              if (params.USE_AUTH) {
                cmd = """powershell -NoProfile -Command ^
                  "$pair = '${IS_USER}:${IS_PASS}'; ^
                   $b64 = [Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes($pair)); ^
                   $body = '${params.POST_BODY}'; ^
                   try { ^
                     $resp = Invoke-WebRequest -Method POST -Uri '${API_URL}' -Headers @{ Authorization = 'Basic ' + $b64 } -ContentType '${params.CONTENT_TYPE}' -Body $body -UseBasicParsing; ^
                     $code = $resp.StatusCode; ^
                   } catch { ^
                     $code = $_.Exception.Response.StatusCode.Value__; ^
                   }; ^
                   Write-Output $code" """
              } else {
                cmd = """powershell -NoProfile -Command ^
                  "$body = '${params.POST_BODY}'; ^
                   try { ^
                     $resp = Invoke-WebRequest -Method POST -Uri '${API_URL}' -ContentType '${params.CONTENT_TYPE}' -Body $body -UseBasicParsing; ^
                     $code = $resp.StatusCode; ^
                   } catch { ^
                     $code = $_.Exception.Response.StatusCode.Value__; ^
                   }; ^
                   Write-Output $code" """
              }
              def status = bat(returnStdout: true, script: cmd).trim()
              echo "Attempt ${i}: HTTP ${status}"
              if (status == '200' || status == '202') { ready = true; break }
            }
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
