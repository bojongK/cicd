pipeline {
    agent any
    stages {
        stage('Git Clone') {
            steps {
                script {
                    try {
                        git url: "https://$GIT_URL", branch: "master", credentialsId: "$GIT_CREDENTIALS_ID"
                        sh "sudo rm -rf ./.git"
                        env.cloneResult=true
                    } catch (error) {
                        print(error)
                        env.cloneResult=false
                        currentBuild.result = 'FAILURE'
                    }
                }
            }
        }
        stage('Build JAR') {
            when {
                expression {
                    return env.cloneResult ==~ /(?i)(Y|YES|T|TRUE|ON|RUN)/
                }                
            }
            steps {
                script{
                    try {
                        sh """
                        rm -rf deploy
                        mkdir deploy
                        """ 
                        sh "sudo sed -i \"s/module_name=.*/module_name=${env.JOB_NAME}\\:${env.BUILD_NUMBER}/g\" /var/lib/jenkins/workspace/${env.JOB_NAME}/src/main/resources/application.properties"
                        sh "cat /var/lib/jenkins/workspace/${env.JOB_NAME}/src/main/resources/application.properties"
                        sh 'mvn -Dmaven.test.failure.ignore=true clean install'
                        sh """
                        cd deploy
                        cp /var/lib/jenkins/workspace/${env.JOB_NAME}/target/*.jar ./${env.JOB_NAME}.jar
                        """
                        env.mavenBuildResult=true
                    } catch (error) {
                        print(error)
                        echo 'Remove Deploy Files'
                        sh "sudo rm -rf /var/lib/jenkins/workspace/${env.JOB_NAME}/*"
                        env.mavenBuildResult=false
                        currentBuild.result = 'FAILURE'
                    }
                }
            }
            post {
                success {
                    slackSend channel: '#pipeline-deploy', color: 'good', message: "The pipeline ${currentBuild.fullDisplayName} stage Build JAR successfully."
                }
                failure {
                    slackSend channel: '#pipeline-deploy', color: 'danger', message: "The pipeline ${currentBuild.fullDisplayName} stage Build JAR failed."
                }
            }
        }
        stage('Docker Build'){
            when {
                expression {
                    return env.mavenBuildResult ==~ /(?i)(Y|YES|T|TRUE|ON|RUN)/
                }
            }
            steps {
                script{
                    try {
                        sh"""
                        #!/bin/bash
                        cd ./deploy
                        cat>Dockerfile<<-EOF
FROM ${ECR_BASE_URL}:latest
ADD ${env.JOB_NAME}.jar /home/${env.JOB_NAME}.jar
CMD nohup java -jar /home/${env.JOB_NAME}.jar 1> /dev/null 2>&1
EXPOSE 9000
EOF"""
                        sh"""
                        cd ./deploy
                        \$(aws ecr get-login --no-include-email --region ap-northeast-2)
                        docker build -t ${env.JOB_NAME.toLowerCase()} .
                        docker tag ${env.JOB_NAME.toLowerCase()}:latest ${ECR_TASK_URL}:ver${env.BUILD_NUMBER}
                        docker push ${ECR_TASK_URL}:ver${env.BUILD_NUMBER}
                        """
                        echo 'Remove Deploy Files'
                        sh "sudo rm -rf /var/lib/jenkins/workspace/${env.JOB_NAME}/*"
                        env.dockerBuildResult=true
                    } catch (error) {
                        print(error)
                        echo 'Remove Deploy Files'
                        sh "sudo rm -rf /var/lib/jenkins/workspace/${env.JOB_NAME}/*"
                        env.dockerBuildResult=false
                        currentBuild.result = 'FAILURE'
                    }
                }
            }
            post {
                success {
                    slackSend channel: '#jenkins', color: 'good', message: "The pipeline ${currentBuild.fullDisplayName} stage Docker Build successfully."
                }
                failure {
                    slackSend channel: '#jenkins', color: 'danger', message: "The pipeline ${currentBuild.fullDisplayName} stage Docker Build failed."
                }
            }
        }
        stage('Push Yaml'){
            when {
                expression {
                    return env.dockerBuildResult ==~ /(?i)(Y|YES|T|TRUE|ON|RUN)/
                }
            }
            steps {
                script{
                    try {
                        git url: "https://$GIT_URL-yaml", branch: "master", credentialsId: "$GIT_CREDENTIALS_ID"
                        sh "rm -rf /var/lib/jenkins/workspace/${env.JOB_NAME}/*"
                        sh """
                        mkdir yaml
                        cd yaml
                        #!/bin/bash
                        cat>deployment.yaml<<-EOF
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: cb-test-api
  namespace: prd-api
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: cb-test-api
    spec:
      containers:
      - image: ${ECR_TASK_URL}:ver${env.BUILD_NUMBER}
        imagePullPolicy: Always
        name: cb-test-api
        ports:
        - containerPort: 9000
EOF"""
                        sh "cat /var/lib/jenkins/workspace/${env.JOB_NAME}/yaml/deployment.yaml"
                        withCredentials([[$class: "UsernamePasswordMultiBinding", credentialsId: "$GIT_CREDENTIALS_ID", usernameVariable: "GIT_AUTHOR_NAME", passwordVariable: "GIT_PASSWORD"]]) {
                            sh """
                            git add .
                            git commit -m "Deploy ${env.JOB_NAME} ${env.BUILD_NUMBER}"
                            git push https://${GIT_AUTHOR_NAME}:${GIT_PASSWORD}@${GIT_URL}-yaml
                            """
                        }                      
                        sh "sudo rm -rf /var/lib/jenkins/workspace/${env.JOB_NAME}/*"
                        env.pushYamlResult=true
                    } catch (error) {
                        print(error)
                        echo 'Remove Deploy Files'
                        withCredentials([[$class: "UsernamePasswordMultiBinding", credentialsId: "$GIT_CREDENTIALS_ID", usernameVariable: "GIT_AUTHOR_NAME", passwordVariable: "GIT_PASSWORD"]]) {
                            sh """
                            git reset --hard HEAD^
                            git push --force https://${GIT_AUTHOR_NAME}:${GIT_PASSWORD}@${GIT_URL}-yaml
                            """
                        }
                        sh "sudo rm -rf /var/lib/jenkins/workspace/${env.JOB_NAME}/*"
                        env.pushYamlResult=false
                        currentBuild.result = 'FAILURE'
                    }
                }
            }
            post {
                success {
                    slackSend channel: '#jenkins', color: 'good', message: "The pipeline ${currentBuild.fullDisplayName} stage Push Yaml successfully."
                }
                failure {
                    slackSend channel: '#jenkins', color: 'danger', message: "The pipeline ${currentBuild.fullDisplayName} stage Push Yaml failed."
                }
            }
        }
        stage('Argo Deploy'){
            when {
                expression {
                    return env.pushYamlResult ==~ /(?i)(Y|YES|T|TRUE|ON|RUN)/
                }
            }
            steps {
                script{
                    try {
                        withEnv(["PATH=/usr/local/bin:$PATH"]) {
                            sh"""
#!/bin/bash
expect << EOF
spawn argocd login --grpc-web $ARGOCD_DOMAIN


expect "Username:"
send "admin\r";    


expect "Password:"
send "$ARGOCD_PW\r";    
                                
expect eof
EOF
                                argocd app get $ARGOCD_APP_NAME
                                argocd app sync $ARGOCD_APP_NAME
                            """
                        }
                    } catch (error) {
                        print(error)
                        echo 'Remove Deploy Files'
                        sh "sudo rm -rf /var/lib/jenkins/workspace/${env.JOB_NAME}/*"
                        currentBuild.result = 'FAILURE'
                    }
                }
            }
            post {
                success {
                    slackSend channel: '#jenkins', color: 'good', message: "The pipeline ${currentBuild.fullDisplayName} successfully."
                }
                failure {
                    slackSend channel: '#jenkins', color: 'danger', message: "The pipeline ${currentBuild.fullDisplayName} failed."
                }
            }
        }
    }
}
