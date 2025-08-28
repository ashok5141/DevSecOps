pipeline {
    agent {
        label 'jenkins-agent'
    }

    environment {
        DVWA_SRC_DIR = 'dvwa-src'
        DVWA_IMAGE = "dvwa-app:${env.BUILD_ID}"
    }

    stages {
        stage('Checkout SCM') {
            steps {
                // Clean the workspace and checkout this repository (containing the Jenkinsfile)
                cleanWs()
                checkout scm
            }
        }

        stage('Clone DVWA Source') {
            steps {
                // Clone the actual DVWA application source code
                git url: 'https://github.com/digininja/DVWA.git', branch: 'master', dir: DVWA_SRC_DIR
            }
        }

        stage('Build DVWA Image') {
            steps {
                dir(DVWA_SRC_DIR) {
                    script {
                        // Use the Dockerfile from the cloned DVWA source
                        docker.build(DVWA_IMAGE, '.')
                    }
                }
            }
        }

        stage('SAST Scan (Semgrep)') {
            steps {
                dir(DVWA_SRC_DIR) {
                    sh 'semgrep --config="p/php" --error --verbose .'
                }
            }
        }

        stage('SCA Scan (Composer Audit)') {
            steps {
                dir(DVWA_SRC_DIR) {
                    // DVWA doesn't have a composer.lock, so we install deps first, then audit
                    // This is a more realistic scenario for a project without committed vendor files
                    sh 'composer install --no-dev --no-interaction'
                    sh 'composer audit'
                }
            }
        }

        stage('Container Scan (Trivy)') {
            steps {
                sh "trivy image --exit-code 1 --severity HIGH,CRITICAL ${DVWA_IMAGE}"
            }
        }

        stage('DAST Scan (OWASP ZAP)') {
            steps {
                script {
                    dir(DVWA_SRC_DIR) {
                        try {
                            // Start the DVWA application using its own compose file
                            sh 'docker-compose up -d'

                            // Wait for the application to be ready
                            sh 'echo "Waiting for DVWA to start..." && sleep 45'

                            // Run the ZAP baseline scan
                            // We use --network="host" to allow ZAP to access localhost:4280
                            sh """
                            docker run --rm --network="host" -v \${PWD}:/zap/wrk/:rw \
                            owasp/zap2docker-stable zap-baseline.py \
                            -t http://127.0.0.1:4280/ -g zap-report.conf -r zap-report.html
                            """
                        } finally {
                            // Tear down the application
                            sh 'docker-compose down'
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            // Archive the DAST report
            dir(DVWA_SRC_DIR) {
                archiveArtifacts artifacts: 'zap-report.html', allowEmptyArchive: true
            }
            // Clean up the workspace to remove the cloned source and other files
            cleanWs()
        }
    }
}
