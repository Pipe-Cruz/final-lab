pipeline {
    agent any
    
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }

    environment {
        SCANNER_HOME= tool 'sonar-scanner'
        EC2_URL = 'http://44.194.185.232'
    }

    stages {
        
        stage('Clean Workspace') {
            steps{
                cleanWs()
            }
        }

        //SECRET SCANNING
        stage('Git Checkout') {
            steps {
                git branch: 'Jenkins-CICD', url: 'https://github.com/Pipe-Cruz/DevSecOps-Project.git' 
            }
        }
        
        stage('GitLeaks Scan') {
            steps {
                script {
                    sh 'docker run -v \${WORKSPACE}:/path ghcr.io/gitleaks/gitleaks:latest -r="/path" -f=gitleaks-report.json'
                }
            }
        }

        //SAST
        stage('SonarQube Scan') {
            steps {
                script {
                    withSonarQubeEnv('sonar-server') {
                    sh "$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectKey=DevSecOps-project -Dsonar.projectName=DevSecOps-project"
                    }
                    /*
                    withCredentials([usernamePassword(credentialsId: 'sonarAPI-token', usernameVariable: 'SONARQUBE_USERNAME', passwordVariable: 'SONARQUBE_PASSWORD')]) {
                        def vulnerabilities = sh(script: """
                            curl -s -u \$SONARQUBE_USERNAME:\$SONARQUBE_PASSWORD -X GET \
                            "http://54.145.218.228:9000/api/issues/search?id=Final-lab&severities=MAJOR,CRITICAL,BLOCKER" \
                            | jq -r '.total'
                        """, returnStdout: true).trim()
                        echo vulnerabilities
                        if (vulnerabilities.toInteger() > 0) {
                            error "SAST: Pipeline failure due to Major, Critical or Blocked category vulnerabilities in SonarQube."
                        } else {
                            echo "Quality Gate passed."
                        }
                    }
                    */
                }
            }
        }
        
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
        
        //SCA
        stage('Dependency-Check Scan') {
            steps {
                script {
                    def apiKeyCredential = credentials('DP-check-token')
                    dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit --nvdApiKey=\${apiKeyCredential}', odcInstallation: 'DP-Check'
                    dependencyCheckPublisher pattern: 'dependency-check-report.xml'
                    /*
                    def vulnerabilitiesXml = readFile('/var/lib/jenkins/workspace/netflix/dependency-check-report.xml')
                    def criticalVulnerabilities = vulnerabilitiesXml.contains('<severity>CRITICAL</severity>') ? 1 : 0
                    def highVulnerabilities = vulnerabilitiesXml.contains('<severity>HIGH</severity>') ? 1 : 0
                    def mediumVulnerabilities = vulnerabilitiesXml.contains('<severity>MEDIUM</severity>') ? 1 : 0

                    if (criticalVulnerabilities >  0 || highVulnerabilities > 0 || mediumVulnerabilities > 0) {
                        error "SCA: Pipeline failure due to medium, high, or critical category vulnerabilities in Dependency-Check."
                    } else {
                        echo "Dependency-Check passed."
                    }
                    */
                }
            }
        }
        
        stage('Trivy FileSystem Scan') {
            steps {
                sh "trivy fs -f json -o trivy-filesystem-report.json ."   
            }
        }
        
        stage('Build & Tag Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'dockerhub-token', toolName: 'docker') {
                        sh "docker build --build-arg TMDB_V3_API_KEY=a39af0296e3f125c9e57ba803453c93a -t netflix ."
                        sh "docker tag netflix pipe7cruz/netflix:latest "
                    }
                }
            }
        }
        
        //IMAGE SECURITY
        stage('Trivy Image Scan') {
            steps {
                script {
                    sh "trivy image -f json -o trivy-image-report.json pipe7cruz/netflix:latest"
                    /*
                    def trivyReportJson = readFile(file: 'trivy-image-report.json')
                    def trivyReport = new groovy.json.JsonSlurper().parseText(trivyReportJson)
                    def severities = trivyReport.Results.Vulnerabilities.collect { it.Severity }.flatten()
                    if (severities.contains('CRITICAL') || severities.contains('HIGH') || severities.contains('MEDIUM')) {
                        error "Image Security: Pipeline failure due to medium, high, or critical category vulnerabilities in Trivy Image scan."
                    } else {
                        echo "Trivy Image Passed."
                    }
                    */
                }
            }
        }
        
        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'dockerhub-token', toolName:'docker'){              
                        sh "docker push pipe7cruz/netflix:latest"
                    }
                }
            }
        }
        
        stage('Deploy to container') {
            steps {
                script {
                    sh "docker rm -f netflix || true"
                    sh "docker run -d --name netflix -p 8081:80 pipe7cruz/netflix:latest"
                    sleep 30
                }
            }
        }
        
        //DAST
        stage('OWASP ZAP Scan') {
            steps {
                script {
                    sh "docker pull owasp/zap2docker-stable:latest"
                    sh "docker rm -f owasp || true"
                    sh "docker run -dt --name owasp owasp/zap2docker-stable /bin/bash"
                    sh "docker exec owasp mkdir /zap/wrk"
                    sh "docker exec owasp zap-baseline.py -t \${EC2_URL}:8081 -r owasp-zap-report.html -I"
                    sh "docker cp owasp:/zap/wrk/owasp-zap-report.html \${WORKSPACE}/owasp-zap-report.html"
                    sh "docker rm -f owasp"
                }
            }
        }
        
    }
    
    post {
        always {
            archiveArtifacts 'gitleaks-report.json'
            archiveArtifacts 'dependency-check-report.xml'
            archiveArtifacts 'trivy-filesystem-report.json'
            archiveArtifacts 'trivy-image-report.json'
            archiveArtifacts 'owasp-zap-report.html'
        }
    }
    
}