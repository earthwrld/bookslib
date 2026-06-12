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

        stage('Security Gate 3: Container Scanning') {
            steps {
                echo 'Running Container Scanning using Trivy...'
                sh '''
                # Scan Auth Service
                docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image --severity CRITICAL bookslib-auth-service
                # Scan Books Service
                docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image --severity CRITICAL bookslib-books-service
                # Scan Reviews Service
                docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image --severity CRITICAL bookslib-reviews-service
                # Scan Frontend
                docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image --severity CRITICAL bookslib-frontend
                '''
            }
            post {
                failure {
                    script {
                        createGitHubIssue(
                            "Security Alert: Container Vulnerability Found", 
                            "Trivy mendeteksi celah keamanan CRITICAL pada Docker image. Tolong perbarui base image atau library."
                        )
                    }
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
