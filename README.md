# first-cicd



pipeline {
    agent any
    tools {
        maven 'maven3'
        jdk 'jdk17'
    }
  
    environment {
        SCANNER_HOME = tool "sonar-scanner"
    }
  
    stages {
        stage('Git checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/rahman588/Ekart.git' 
            }
        }
        
        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }
        
        stage('Unit tests') {
            steps {
                sh "mvn test -DskipTests=true"
            }
        }
        
        stage('Sonarqube analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectKey=EKART \
                        -Dsonar.projectName=EKART \
                        -Dsonar.java.binaries=.
                    '''
                }   
            }
        }
        
        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck additionalArguments: '--scan ./', odcInstallation: 'DC'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        
        stage('Build') {
            steps {
                sh "mvn package -DskipTests=true"
            }
        }
        
        stage('Deploy To Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'global-maven', jdk: 'jdk17', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
                    sh "mvn deploy -DskipTests=true"
                }
            }
        }  // <-- This closing brace was missing
        
        stage('Build & Tag Docker image') {
            steps {
                script {  
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "docker build -t abdul776/ekart:latest -f docker/Dockerfile ."
                    }
                }
            }
        }
        
        stage('Trivy scan') {
            steps {
                sh "trivy image abdul776/ekart:latest > trivy-report.txt"  // Changed .ext to .txt
            }
        }
        
        stage('Docker push image') {
            steps {
                script {  
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "docker push abdul776/ekart:latest"  // Removed space before :latest
                    }
                }
            }
        }
        
        stage('Kubernetes deploy') {
            steps {
                withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'k8-token', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://10.9.223.27:6443') {
                    sh "kubectl apply -f deploymentservice.yml"
                    sh "kubectl get svc -n webapps"  // Removed duplicate -n webapps
                }
            }
        }
    }
}
