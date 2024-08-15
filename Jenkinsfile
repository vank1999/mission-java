pipeline {
    agent any

    tools{
        jdk 'jdk17'
        maven 'maven3'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {
        stage('Git checkout') {
            steps {
                git branch: 'main', credentialsId: 'git-cred', 
                url: 'https://github.com/vank1999/mission-java.git'
            }
        }

        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }

        stage('Test') {
            steps {
                sh "mvn test -DskipTests=true"
            }
        }

        stage('Trivy scan') {
            steps {
                sh 'trivy fs --format table -o trivy-fs-report.html .'
            }
        }

        stage('SonarQube analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner \
                    -Dsonar.projectKey=mission1 \
                    -Dsonar.projectName=mission1 \
                    -Dsonar.java.binaries=. '''
                }
            }
        }

        stage('Build') {
            steps {
                sh "mvn package -DskipTests=true"
            }
        }

        stage('Deploy Artifacts To Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'global-settings-xml') {
                sh "mvn deploy -DskipTests=true"
          }
        }
      }

      stage('Build & Tag Docker Image') {
        steps {
            script {
                withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                sh "docker build -t vank1999/mission:latest ."
        }
       }
      }
     }

     stage('Trivy Scan Image') {
        steps {
        sh "trivy image --format table -o trivy-image-report.html vank1999/mission:latest"
        }
      }

      stage('Publish Docker Image') {
        steps {
            script {
                withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                sh "docker push vank1999/mission:latest"
            }
          }
        }
      }

      stage('Deploy To K8s') {
        steps {
            withKubeConfig(caCertificate: '', clusterName: 'my-ak-eks',
            contextName: '', credentialsId: 'k8-token',
            namespace: 'webapps', restrictKubeConfigAccess: false,
            serverUrl: 'https://C5E2E20D71A8A5B3C146C1DAC3406AFF.gr7.us-east-1.eks.amazonaws.com') {
            sh "kubectl apply -f ds.yml -n webapps"
            sleep 60
        }
      }
    }

    stage('Verify Deployment') {
        steps {
            withKubeConfig(caCertificate: '', clusterName: 'my-ak-eks', 
            contextName: '', credentialsId: 'k8-token',
            namespace: 'webapps', restrictKubeConfigAccess: false,
            serverUrl: 'https://C5E2E20D71A8A5B3C146C1DAC3406AFF.gr7.us-east-1.eks.amazonaws.com') {
            sh "kubectl get pods -n webapps"
            sh "kubectl get svc -n webapps"
        }
      }
    }

    }

    post {
always {
script {
def jobName = env.JOB_NAME
def buildNumber = env.BUILD_NUMBER
def pipelineStatus = currentBuild.result ?: 'UNKNOWN'
def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red'
def body = """
<html>
<body>
<div style="border: 4px solid ${bannerColor}; padding: 10px;">
<h2>${jobName} - Build ${buildNumber}</h2>
<div style="background-color: ${bannerColor}; padding: 10px;">
<h3 style="color: white;">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3>
</div>
<p>Check the <a href="${BUILD_URL}">console output</a>.</p>
</div>
</body>
</html>
"""
emailext (
subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}",
body: body,
to: 'anilkumarvallepu1@gmail.com',
from: 'admin123@gmail.com',
replyTo: 'admin123@gmail.com',
mimeType: 'text/html',
attachmentsPattern: 'trivy-image-report.html'
          )
        }
      }
    }

}
