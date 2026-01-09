pipeline {
    agent any

    environment {
        // Path to your local WebMethods Integration Server installation (on host)
        WMSERVER_HOME = "C:/Users/vkraft_Auni/Documents/w/IntegrationServer"
        PACKAGE_NAME  = "employee.zip"
        PACKAGE_PATH  = "${WORKSPACE}/${PACKAGE_NAME}"
        API_PORT      = "5555"
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
                echo "Checking out latest package from Git..."
                checkout scm
            }
        }

        stage('Deploy WebMethods Package') {
            steps {
                echo "Deploying ${PACKAGE_NAME} to local WebMethods IS..."
                script {
                    // Retry deployment up to 3 times if IS is busy
                    retry(3) {
                        bat """
                            "${WMSERVER_HOME}/deployer/bin/deployer.bat" -silent -package "${PACKAGE_PATH}"
                        """
                    }
                    echo "Deployment completed"
                }
            }
        }

        stage('Verify API Health') {
            steps {
                script {
                    def api = [method: 'POST', path: '/employee/insertEmployee']
                    echo "🔎 Checking API: ${api.method} ${api.path}"

                    def ready = false
                    for (int i = 1; i <= 10; i++) {
                        sleep 10
                        def status = bat(
                            script: """powershell -Command "(Invoke-WebRequest -Method ${api.method} -Uri http://host.docker.internal:${API_PORT}${api.path} -UseBasicParsing).StatusCode" """,
                            returnStdout: true
                        ).trim()

                        echo "Attempt ${i}: HTTP ${status}"

                        if (status == "200" || status == "202") {
                            ready = true
                            echo "✔ API is ready!"
                            break
                        }
                    }

                    if (!ready) {
                        error "API FAILED: ${api.method} ${api.path} not ready after 10 attempts"
                    }
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline finished. Cleaning workspace..."
            cleanWs()
        }
        success {
            echo "🎉 Pipeline succeeded!"
        }
        failure {
            echo "Pipeline failed. Check logs for details."
        }
    }
}