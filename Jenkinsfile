pipeline {
    agent any
    tools { maven 'Maven3' }

    stages {
        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Build') {
            steps { sh 'mvn -B clean package -DskipTests' }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh 'mvn -B sonar:sonar -Dsonar.projectKey=workshop-sast -Dsonar.projectName=workshop-sast'
                }
            }
        }

        stage('Semgrep') {
            steps {
                sh '''
                /var/jenkins_home/semgrep-venv/bin/semgrep scan --config p/java --config p/owasp-top-ten --config p/security-audit --metrics=off --json-output=semgrep-report.json src/ || true
                /var/jenkins_home/semgrep-venv/bin/semgrep scan --config p/java --config p/owasp-top-ten --config p/security-audit --metrics=off src/ || true
                '''
            }
        }
    }

    post {
        always { archiveArtifacts artifacts: 'semgrep-report.json', allowEmptyArchive: true }
    }
}