pipeline {
    agent any

    environment {
        REPO_NAME = "earthwrld/bookslib"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Security Gate 0: IaC Scanning') {
            steps {
                echo 'Running IaC Scanning using Checkov...'
                sh '''
                # Scan Dockerfile dan docker-compose.yaml
                docker run --rm -v "\$PWD:/work" bridgecrew/checkov:latest -d /work --framework dockerfile,docker_compose --soft-fail
                '''
            }
        }

        stage('Security Gate 1: Secret Scanning') {
            steps {
                echo 'Running Secret Scanning using TruffleHog...'
                sh '''
                docker run --rm -v "\$PWD:/pwd" trufflesecurity/trufflehog:latest filesystem /pwd --fail
                '''
            }
            post {
                failure {
                    script {
                        createGitHubIssue(
                            "Security Alert: Hardcoded Secrets Found", 
                            "TruffleHog mendeteksi adanya kredensial yang di-hardcode. Tolong periksa log pipeline Jenkins."
                        )
                    }
                }
            }
        }

        stage('Security Gate 2: SAST') {
            steps {
                echo 'Running SAST using Semgrep...'
                sh '''
                docker run --rm -v "\$PWD:/src" returntocorp/semgrep:latest semgrep scan --config p/ci --error /src
                '''
            }
            post {
                failure {
                    script {
                        createGitHubIssue(
                            "Security Alert: SAST Vulnerability Found", 
                            "Semgrep mendeteksi celah keamanan pada source code. Tolong periksa log pipeline Jenkins."
                        )
                    }
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                echo 'Building Docker Images...'
                sh 'docker compose build'
            }
        }

        stage('Security Gate 3: Container Scanning (Thresholds)') {
            steps {
                echo 'Running Container Scanning using Trivy...'
                
                // 1. Log MEDIUM & LOW (Tidak fail, hanya log)
                sh '''
                echo "=== Scanning for LOW & MEDIUM Vulnerabilities ==="
                for service in auth-service books-service reviews-service frontend; do
                    docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image --severity LOW,MEDIUM bookslib-\$service || true
                done
                '''
                
                // 2. Warning HIGH (Buat Issue tapi lanjut)
                script {
                    def services = ['auth-service', 'books-service', 'reviews-service', 'frontend']
                    for (service in services) {
                        def highScan = sh(script: "docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image --severity HIGH bookslib-${service} | grep -E 'HIGH: [1-9]' || true", returnStdout: true).trim()
                        if (highScan != "") {
                            createGitHubIssue(
                                "Security Warning: HIGH Vulnerability in ${service}", 
                                "Trivy mendeteksi celah keamanan HIGH pada Docker image bookslib-${service}. Harap diperiksa namun pipeline akan tetap dilanjutkan untuk saat ini."
                            )
                        }
                    }
                }
                
                // 3. Block CRITICAL (Fail pipeline)
                sh '''
                echo "=== Scanning for CRITICAL Vulnerabilities (BLOCKING) ==="
                for service in auth-service books-service reviews-service frontend; do
                    docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image --exit-code 1 --severity CRITICAL bookslib-\$service
                done
                '''
            }
            post {
                failure {
                    script {
                        createGitHubIssue(
                            "Security Alert: CRITICAL Container Vulnerability", 
                            "Trivy mendeteksi celah keamanan CRITICAL pada Docker image. Pipeline dihentikan. Tolong perbarui base image atau library."
                        )
                    }
                }
            }
        }

        stage('Generate SBOM (Supply Chain Security)') {
            steps {
                echo 'Generating SBOM menggunakan Trivy...'
                sh '''
                for service in auth-service books-service reviews-service frontend; do
                    docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v "\$PWD:/workspace" aquasec/trivy image --format cyclonedx --output /workspace/sbom-\$service.json bookslib-\$service
                done
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'sbom-*.json', fingerprint: true
                }
            }
        }

        stage('Deploy to Production') {
            when {
                branch 'main' // Hanya deploy jika branch adalah main
            }
            steps {
                echo 'Deploying application using Docker Compose...'
                sh 'docker compose up -d'
            }
        }
    }
}

// Fungsi pembantu untuk membuat GitHub Issue
def createGitHubIssue(String issueTitle, String issueBody) {
    // Membutuhkan 'github-token' yang disimpan di Jenkins Credentials (Secret Text)
    withCredentials([string(credentialsId: 'github-token', variable: 'GH_TOKEN')]) {
        sh """
        curl -X POST \
          -H "Accept: application/vnd.github.v3+json" \
          -H "Authorization: token \$GH_TOKEN" \
          https://api.github.com/repos/${REPO_NAME}/issues \
          -d '{"title":"${issueTitle}", "body":"${issueBody}"}'
        """
    }
}
