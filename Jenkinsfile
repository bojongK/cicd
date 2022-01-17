pipeline {
  agent any
  tools {
    maven 'M3'
  }

  stages {

    stage('Checkout Application Git Branch') {
        steps {
            git credentialsId: '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',
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
  }
  
}
