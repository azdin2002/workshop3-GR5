pipeline {
    agent any
    tools { maven 'Maven3' }

        parameters {
            string(name: 'TRIVY_SEVERITY', defaultValue: 'HIGH,CRITICAL',
                   description: 'Severites qui font echouer le pipeline (Trivy Gate)')
        }

        environment {
            TRIVY_BIN       = '/var/jenkins_home/bin/trivy'
            TRIVY_CACHE_DIR = '/var/jenkins_home/trivy-cache'
            IMAGE_NAME      = 'workshop-platform'
            IMAGE_TAG       = "${env.BUILD_NUMBER}"
        }

    stages {
        stage('Checkout') {
            steps { checkout scm }
        }
        stage('Semgrep') {
                    steps {
                        sh '''
                        /var/jenkins_home/semgrep-venv/bin/semgrep scan --config p/java --config p/owasp-top-ten --config p/security-audit --metrics=off --json-output=semgrep-report.json src/ || true
                        /var/jenkins_home/semgrep-venv/bin/semgrep scan --config p/java --config p/owasp-top-ten --config p/security-audit --metrics=off src/ || true
                        '''
                    }
        }
        stage('Build') {
            steps { sh 'mvn -B clean package -DskipTests' }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh 'mvn -B org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=workshop-sast -Dsonar.projectName=workshop-sast'
                }
            }
        }
        stage('Dependency-Check') {
            steps {
                sh '''
                mvn -B org.owasp:dependency-check-maven:check \
                  -Dformat=ALL \
                  -DdataDirectory=/var/jenkins_home/dependency-check-data \
                  -DautoUpdate=false \
                  -DfailBuildOnCVSS=11
                '''
            }
        }
        stage('Docker Build') {
             steps {
                        sh '''
                         test -f Dockerfile || { echo "Dockerfile introuvable"; exit 1; }
                          docker build -t "$IMAGE_NAME:$IMAGE_TAG" .
                          '''
                    }
        }

        stage('Trivy Scan') {
              steps {
                    sh '''
                        "$TRIVY_BIN" --version

                        "$TRIVY_BIN" image \
                          --scanners vuln,misconfig,secret \
                          --severity HIGH,CRITICAL \
                          --timeout 15m \
                          --format json --output trivy-image.json \
                          "$IMAGE_NAME:$IMAGE_TAG"

                        "$TRIVY_BIN" convert --format table --output trivy-image.txt trivy-image.json
                        "$TRIVY_BIN" convert --format template --template "@/var/jenkins_home/trivy-templates/html.tpl" --output trivy-image.html trivy-image.json
                        cat trivy-image.txt

                        mkdir -p trivy-config
                        cp Dockerfile trivy-config/Dockerfile
                        "$TRIVY_BIN" config --severity HIGH,CRITICAL --format table --output trivy-dockerfile.txt trivy-config
                        cat trivy-dockerfile.txt
                        '''
              }
        }

                stage('Trivy Gate') {
                    steps {
                        sh '''
                        "$TRIVY_BIN" image \
                          --scanners vuln,misconfig,secret \
                          --severity "$TRIVY_SEVERITY" \
                          --exit-code 1 \
                          --quiet \
                          "$IMAGE_NAME:$IMAGE_TAG"
                        '''
                    }
                }

    }
        post {
            always {
                archiveArtifacts artifacts: 'semgrep-report.json, target/dependency-check-report.*, trivy-image.*, trivy-dockerfile.txt', allowEmptyArchive: true
                sh 'docker rmi "$IMAGE_NAME:$IMAGE_TAG" || true'
            }
        }
}