//pipeline code
pipeline {
    agent any
    stages {
        stage('Networking Configuration') {
            steps {
                sh 'docker network ls'
            }
        }
        stage('Install Dependencies') {
            steps {
                sh 'cd $WORKSPACE'
                sh 'rm -rf project'
                git branch: "master",
                    url: "https://github.com/RaziAbbas1/Devsecops.git"
                sh 'ls'
            }
        }
        stage('Pre-Build Tests') {
            parallel {
                stage('Git Repository Scanner') {
                    steps {
                        sh 'cd $WORKSPACE'
                        sh 'trufflehog https://github.com/RaziAbbas1/Devsecops --json | jq "{branch:.branch, commitHash:.commitHash, path:.path, stringsFound:.stringsFound}" > trufflehog_report.json || true'
                        sh 'cat trufflehog_report.json'
                        sh 'echo "Scanning Repositories.....done"'
                        archiveArtifacts artifacts: 'trufflehog_report.json', onlyIfSuccessful: true                      
                       // emailext attachLog: true, attachmentsPattern: 'trufflehog_report.json',
                       // body: "${currentBuild.currentResult}: Job ${env.JOB_NAME} build ${env.BUILD_NUMBER}\n More info at: ${env.BUILD_URL}\n Thankyou,\n CDAC-Project Group-7", 
                       // subject: "Jenkins Build ${currentBuild.currentResult}: Job ${env.JOB_NAME} - success", mimeType: 'text/html', to: "raziabbasrizvi75@gmail.com"
                    }
                }
                stage('Image Security') {
                    steps {
                        sh 'cd $WORKSPACE'
                        sh 'trivy image --no-progress --light --exit-code 1 CRITICAL,HIGH centos > centos_CVE.txt'
                        archiveArtifacts artifacts: 'centos_CVE.txt', onlyIfSuccessful: true
      //                  emailext attachLog: true, attachmentsPattern: '*.json', 
     //                   body: "${currentBuild.currentResult}: Job ${env.JOB_NAME} build ${env.BUILD_NUMBER}\n More info at: ${env.BUILD_URL}\n Please Find Attachments for the following:\n Thankyou\n CDAC-Project Group-7",
     //                   subject: "${env.JOB_NAME} - Build # ${env.BUILD_NUMBER} - success", mimeType: 'text/html', to: "abbyvishnoi@gmail.com"
                    }
                }
            }
        }
        stage('Build Stage') {
            steps {
                sh 'mvn clean'
                sh 'mvn compile'
                sh 'mvn package'
            }
        }
       
        stage('SCA') {
            parallel {
                stage('Dependency Check') {
                    steps {
                        sh 'wget https://github.com/RaziAbbas1/Devsecops/blob/master/dc.sh'
                        sh 'chmod +x dc.sh'
                        sh './dc.sh'
                        archiveArtifacts artifacts: 'odc-reports/*.html', onlyIfSuccessful: true
                        archiveArtifacts artifacts: 'odc-reports/*.csv', onlyIfSuccessful: true
                        archiveArtifacts artifacts: 'odc-reports/*.json', onlyIfSuccessful: true
     //                   emailext attachLog: true, attachmentsPattern: '*.html', 
     //                   body: "${currentBuild.currentResult}: Job ${env.JOB_NAME} build ${env.BUILD_NUMBER}\n More info at: ${env.BUILD_URL}\n Please Find Attachments for the following:\n Thankyou\n CDAC-Project Group-7",
     //                   subject: "${env.JOB_NAME} - Build # ${env.BUILD_NUMBER} - success", mimeType: 'text/html', to: "abbyvishnoi@gmail.com"
                    }
                }
                stage('Junit Testing') {
                    steps {
                        sh 'echo "Junit Reports are created using archiveArtifacts"'
                        archiveArtifacts artifacts: '*junit.xml', onlyIfSuccessful: true
                   //     emailext attachLog: true, attachmentsPattern: '*junit.xml', 
                    //    body: "${currentBuild.currentResult}: Job ${env.JOB_NAME} build ${env.BUILD_NUMBER}\n More info at: ${env.BUILD_URL}\n Please Find Attachments for the following:\n Thankyou\n CDAC-Project Group-7",
                   //     subject: "${env.JOB_NAME} - Build # ${env.BUILD_NUMBER} - success", mimeType: 'text/html', to: "abbyvishnoi@gmail.com"
                    }
                }
            }
        }
           stage('SonarQube Analysis') {
             steps {
                  sh 'docker container stop sonarqube || true'
                  sh 'docker container rm -f sonarqube || true'
                  sh 'docker run -p 9000:9000 -d --name sonarqube owasp/sonarqube'
                  sh 'mvn sonar:sonar -Dsonar.projectKey=abbas -Dsonar.host.url=http://13.234.239.244:9000 -Dsonar.login=05f3a284f6dac43bda14023034422f540c5d10d5 || true'
           }
       }
        
               stage('Build Docker Images') {
                     steps {
                          sh 'docker build -t $JOB_NAME:v1.$BUILD_ID .'
                          sh 'docker image tag $JOB_NAME:v1.$BUILD_ID raziabbas1996/$JOB_NAME:v1.$BUILD_ID'
                          sh 'docker image tag $JOB_NAME:v1.$BUILD_ID raziabbas1996/$JOB_NAME:latest'
                  
              }
        }
       stage('Push Image To Docker Hub') { 
             steps {
                           withCredentials([string(credentialsId: 'dockerhubpassword', variable: 'dockerhubpassword')]) {
                           sh 'docker login -u raziabbas1996 -p ${dockerhubpassword}'
               }
                           sh 'docker image push raziabbas1996/$JOB_NAME:v1.$BUILD_ID'
                           sh 'docker image push raziabbas1996/$JOB_NAME:latest'
                           sh 'docker rmi $JOB_NAME:v1.$BUILD_ID raziabbas1996/$JOB_NAME:v1.$BUILD_ID raziabbas1996/$JOB_NAME:latest'
                   }
               }    
            stage('Deploying Containers') {
                  steps {  
                        script {
                           def dockerrun = 'docker run -p 8080:8080 -d --name Devsecops raziabbas1996/$JOB_NAME:latest'
                           def dockerrm = 'docker container rm -f Devsecops'
                           def dockerimg = 'docker rmi raziabbas1996/$JOB_NAME'
                           sshagent(['docker_Server']) {                     
                           sh "ssh -o StrictHostKeyChecking=no ec2-user@172.31.3.168 ${dockerrm}"
                           sh "ssh -o StrictHostKeyChecking=no ec2-user@172.31.3.168 ${dockerimg}"
                           sh "ssh -o StrictHostKeyChecking=no ec2-user@172.31.3.168 ${dockerrun}"
                       }     
                  }
                }   
            }
       }
}
