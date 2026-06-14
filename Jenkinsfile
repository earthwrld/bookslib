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
                # Scan Dockerfile
                docker run --rm -v "$(pwd):/work" bridgecrew/checkov:latest -d /work --framework dockerfile --soft-fail
                '''
            }
        }

        stage('Security Gate 1: Secret Scanning') {
            steps {
                echo 'Running Secret Scanning using TruffleHog...'
                sh '''
                docker run --rm -v "$(pwd):/pwd" trufflesecurity/trufflehog:latest filesystem /pwd --fail
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
                docker run --rm -v "$(pwd):/src" returntocorp/semgrep:latest semgrep scan --config p/ci --error /src
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
                chmod 777 .
                for service in auth-service books-service reviews-service frontend; do
                    docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v "\$(pwd):/workspace" aquasec/trivy image --format cyclonedx --output /workspace/sbom-\${service}.json bookslib-\${service}
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
                
                echo 'Menunggu layanan hidup sepenuhnya...'
                sleep 15
            }
        }

        stage('Security Gate 4: DAST (OWASP ZAP)') {
            when {
                branch 'main'
            }
            steps {
                echo 'Running Dynamic Application Security Testing menggunakan OWASP ZAP...'
                sh '''
                # Beri izin tulis agar ZAP bisa menyimpan hasil report HTML ke direktori kerja Jenkins
                chmod 777 .
                
                # Jalankan ZAP Baseline Scan melawan frontend (localhost:3000)
                # Opsi -I berarti mengabaikan warning agar pipeline tidak gagal merah, kita hanya ingin reportnya
                docker run --rm --network host -v "$(pwd)":/zap/wrk/:rw ghcr.io/zaproxy/zaproxy:stable zap-baseline.py -t http://localhost:3000 -r zap-report.html -I || true
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'zap-report.html', allowEmptyArchive: true, fingerprint: true
                }
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
