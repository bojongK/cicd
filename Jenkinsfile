pipeline {
  agent any
  tools {
    maven 'M3'
  }
  environment {
    dockerHubRegistry = 'zoome95/spring-test'
    dockerHubRegistryCredential = 'dockerhub'
  }
  
 
  stages {

    stage('Checkout Application Git Branch') {
        steps {
            git credentialsId: '0914c7a7-afd9-43d4-9adf-4373a29ab51b',
                url: 'https://github.com/bojongK/cicd.git',
                branch: 'main'
        }
        post {
                failure {
                  echo 'Repository clone failure !'
                }
                success {
                  echo 'Repository clone success !'
                }
        }
    }
   stage('Maven Jar Build') {
        steps {
            sh 'mvn -DskipTests=true package'
        }
        post {
                failure {
                  echo 'Maven jar build failure !'
                }
                success {
                  echo 'Maven jar build success !'
                }
        }
    }
    stage('Docker Image Build') {
        steps {
            sh "cp target/ROOT.jar ./"
            sh "cp deploy/Dockerfile ./"
            sh "docker build . -t ${dockerHubRegistry}:${currentBuild.number}"
            sh "docker build . -t ${dockerHubRegistry}:latest"
        }
        post {
                failure {
                  echo 'Docker image build failure !'
                }
                success {
                  echo 'Docker image build success !'
                }
        }
    }
    stage('Docker Image Push') {
        steps {
            withDockerRegistry([ credentialsId: dockerHubRegistryCredential, url: "" ]) {
                                sh "docker push ${dockerHubRegistry}:${currentBuild.number}"
                                sh "docker push ${dockerHubRegistry}:latest"

                                sleep 10 /* Wait uploading */ 
                            }
        }
        post {
                failure {
                  echo 'Docker Image Push failure !'
                  sh "docker rmi ${dockerHubRegistry}:${currentBuild.number}"
                  sh "docker rmi ${dockerHubRegistry}:latest"
                }
                success {
                  echo 'Docker image push success !'
                  sh "docker rmi ${dockerHubRegistry}:${currentBuild.number}"
                  sh "docker rmi ${dockerHubRegistry}:latest"
                }
        }
    }
    stage('K8S Manifest Update') {
        steps {
            git credentialsId: 'c09bc592-b136-4df2-a87c-7cc82819e5bf',
                url: 'https://github.com/bojongK/k8s-manifest.git',
                branch: 'main'

            sh "sed -i 's/my-app:.*\$/my-app:${currentBuild.number}/g' deployment.yaml"
            sh "git add deployment.yaml"
            sh "git commit -m '[UPDATE] my-app ${currentBuild.number} image versioning'"
            sshagent(credentials: ['{k8s-manifest repository credential ID}']) {
                sh "git remote set-url origin git@github.com:bojongK/k8s-manifest.git"
                sh "git push -u origin main"
             }
        }
        post {
                failure {
                  echo 'K8S Manifest Update failure !'
                }
                success {
                  echo 'K8S Manifest Update success !'
                }
        }
    }
  }
  
}
