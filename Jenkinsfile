pipeline {
    agent any
    environment {
      // This can be nexus3 or nexus2
      NEXUS_VERSION = "nexus3"
      // This can be http or https
      NEXUS_PROTOCOL = "http"
      // Where your Nexus is running
      NEXUS_URL = "127.0.0.1:8081"
      // Repository where we will upload the artifact
      NEXUS_REPOSITORY = "npm-repo"
      // Jenkins credential id to authenticate to Nexus OSS
      NEXUS_CREDENTIAL_ID = "nexus-credentials"
    }
    stages {
        stage('Build') {
            steps {
                echo 'Running build automation'
                sh './gradlew build --no-daemon'
                archiveArtifacts artifacts: 'dist/trainSchedule.zip'
            }
        }
        
        stage('Build Docker Image') {
            when {
                branch 'master'
            }
            steps {
              script {
                  app = docker.build('willbla/train-schedule')
                  app.inside {
                      echo "curl localhost:8080"
                  }
              }
                
              nexusArtifactUploader(
                nexusVersion: NEXUS_VERSION,
                protocol: NEXUS_PROTOCOL,
                exusUrl: NEXUS_URL,
                groupId: pom.groupId,
                version: pom.version,
                repository: NEXUS_REPOSITORY,
                credentialsId: NEXUS_CREDENTIAL_ID,
                artifacts: [
                  // Artifact generated such as .jar, .ear and .war files.
                  [artifactId: tarfiles,
                    classifier: '',
                    file: 'gradlew.bat',
                    type: 'bat'],
                    // Lets upload the pom.xml file for additional information for Transitive dependencies
                  [artifactId: json,
                    classifier: '',
                    file: 'package-lock.json',
                    type: "json"]
                ]
              );
            }
        }
        stage('Push Docker Image') {
            when {
                branch 'master'
            }
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'docker_hub_login') {
                       app.push("${env.BUILD_NUMBER}")
                      app.push("latest")
                    }
                }
            }
        }
        stage('DeployToProduction') {
            when {
                branch 'master'
            }
            steps {
                input 'Deploy to Production?'
                milestone(1)
                withCredentials([usernamePassword(credentialsId: 'webserver_login', usernameVariable: 'USERNAME', passwordVariable: 'USERPASS')]) {
                    script {
                        sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@$Production \"docker pull willbla/train-schedule:${env.BUILD_NUMBER}\""
                        try {
                            sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@$Production \"docker stop train-schedule\""
                            sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@$Production \"docker rm train-schedule\""
                        } catch (err) {
                            echo: 'caught error: $err'
                        }
                        sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@$Production \"docker run --restart always --name train-schedule -p 8080:8080 -d willbla/train-schedule:${env.BUILD_NUMBER}\""
                    }
                }
            }
        }
    }
}
