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

        stage('Git Checkout') {
            steps {
                git branch: 'Jenkins-CICD', url: 'https://github.com/Pipe-Cruz/DevSecOps-Project.git' 
            }
        }

        //PRE-COMMIT TOOLS
        stage('Git-secrets') {
            steps {
                script {
                    
                    sh 'wget https://github.com/awslabs/git-secrets/archive/refs/tags/v1.3.0.tar.gz'
                    sh 'tar -xvf v1.3.0.tar.gz'
                    sh 'cd git-secrets-1.3.0 && make install'
                    
                    // Configuración de git-secrets
                    def gitSecretsOutput = sh(script: 'git secrets --scan', returnStdout: true).trim()
                    writeFile file: 'git-secrets-output.json', text: gitSecretsOutput

                    // Convertir la salida a formato HTML
                    sh 'jq . git-secrets-output.json > git-secrets-output.html'
                }
            }

        }
        /*
        stage('Git-hound') {

        }
        */
        //SECRET SCANNING
        stage('GitLeaks Scan') {
            steps {
                script {
                    sh 'docker run -v \${WORKSPACE}:/path --name gitleaks ghcr.io/gitleaks/gitleaks:latest -s="/path" -f=json > gitleaks-report.json'
                    sh "docker rm -f gitleaks"
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
                    withCredentials([string(credentialsId: 'DP-check-token', variable: 'apiKeyDP')]) {
                        dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit --nvdApiKey=\${apiKeyDP}', odcInstallation: 'DP-Check'
                        dependencyCheckPublisher pattern: 'dependency-check-report.xml'
                        
                        /*
                        def vulnerabilitiesXml = readFile('/var/lib/jenkins/workspace/devsecops-project/dependency-check-report.xml')
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
                        withCredentials([string(credentialsId: 'TMDB_API_KEY_CREDENTIAL_ID', variable: 'TMDB_V3_API_KEY')]) {
                            sh "docker build --build-arg TMDB_V3_API_KEY=\${TMDB_V3_API_KEY} -t netflix ."
                            sh "docker tag netflix pipe7cruz/netflix:latest "
                        }
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
        /*
        stage('Deploy to Minikube') {
            steps {
                script {
                    // Iniciar Minikube si no está en ejecución
                    sh 'minikube start'
                    // Imprimir información del clúster de Kubernetes y contextos configurados
                    sh 'kubectl config get-contexts'
                    sh 'kubectl config current-context'

                    sh 'kubectl apply -f kubernetes-deployment.yaml'
                    //Esperar a que los pods estén en funcionamiento antes de continuar
                    sh 'kubectl wait --for=condition=ready pod -l app=netflix --timeout=300s'
                    // Obtiene la dirección IP y el puerto del servicio expuesto
                    def serviceIp = sh(script: 'minikube service netflix-service --url', returnStdout: true).trim()
                    echo "La aplicación está disponible en: \${serviceIp}"
                }
            }
        }
        */

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
            archiveArtifacts 'git-secrets-output.html'
            archiveArtifacts 'gitleaks-report.json'
            archiveArtifacts 'dependency-check-report.xml'
            archiveArtifacts 'trivy-filesystem-report.json'
            archiveArtifacts 'trivy-image-report.json'
            archiveArtifacts 'owasp-zap-report.html'
        }
    }
    
}