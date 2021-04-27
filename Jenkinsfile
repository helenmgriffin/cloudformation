def instanceID = ''
def $elasticIP = ''

pipeline{
    agent any
    environment {
        ARTIFACT_NAME = 'CollegeProject.zip'
        AWS_S3_BUCKET = 'collegeproject-zip'
        AWS_EB_APP_NAME = 'CollegeProject'
        AWS_EB_ENVIRONMENT = 'CollegeProject-env'
        AWS_EB_APP_VERSION = "${env.BUILD_NUMBER}"
    }
    stages{
        stage('Get CloudFormation File') {
           steps {
             git url: 'https://github.com/helenmgriffin/cloudformation.git', branch: 'main'
             }
        }
        stage('Deploy to Elastic Beanstalk')
        {
            steps
            {
                script
                {
                    withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'AWSCredentails2', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                        bat 'aws s3 cp c:/CollegeProjectWebSite/%ARTIFACT_NAME% s3://%AWS_S3_BUCKET%/%ARTIFACT_NAME%'
                        bat 'aws elasticbeanstalk create-application-version --application-name %AWS_EB_APP_NAME% --version-label %AWS_EB_APP_VERSION% --source-bundle S3Bucket=%AWS_S3_BUCKET%,S3Key=%ARTIFACT_NAME%'
                        bat 'aws elasticbeanstalk update-environment --application-name %AWS_EB_APP_NAME% --environment-name %AWS_EB_ENVIRONMENT% --version-label %AWS_EB_APP_VERSION%'
                    }
                
                }
            }
            
        }
        stage('AWS: Tear Down Dev Server')
        {
            steps
            {
                script
                {
                    withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'AWSCredentails2', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                        bat 'aws cloudformation delete-stack --stack-name CollegeProjectDevServer'
                        bat 'aws cloudformation wait stack-delete-complete --stack-name CollegeProjectDevServer'
                    }
                }
            }
        }
        stage('AWS: Provision Dev Server')
        {
            steps
            {
                script
                {
                    withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'AWSCredentails2', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                        bat 'aws cloudformation create-stack --stack-name CollegeProjectDevServer --template-body file://CollegeProjectEC2.yaml --parameters ParameterKey=InstanceType,ParameterValue=t2.micro ParameterKey=KeyName,ParameterValue=DevWebServer'
                        bat 'aws cloudformation wait stack-create-complete --stack-name "CollegeProjectDevServer'
                    }
                }
            }
        }
        stage('AWS: Install Docker on Dev Server')
        {
            steps
            {
                script
                {
                    //See if the server was provisioned by using the name we gave the server in the clouformarion file 'CollegeProjectWebServer' to get the public ip address
                    def cmd = new StringBuilder()
                    cmd.append('aws ec2 describe-instances --filters "Name=tag:Name,Values=CollegeProjectWebServer" --query "Reservations[].Instances[].PublicIpAddress" --output text')
                    withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'AWSCredentails2', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                        timeout(2) {//retry the server until the timeout is reached.
                            waitUntil {  
                                stdout = bat(returnStdout: true,script: "${cmd.toString()}").trim()
                                elasticIP = stdout.readLines().drop(1).join(" ")  
                                echo "$elasticIP"
                                if (elasticIP == "" ) {
                                    return false
                                } else {
                                    return true
                                }   
                            }  
                        }
                    }
                    def serverUpdate = 'sudo yum update -y'
                    def dockerInstall = 'sudo amazon-linux-extras install docker'
                    def dockerPermissions = 'sudo usermod -a -G docker ec2-user'
                    def dockerStart = 'sudo service docker start'
                    sshagent(['dev-web-server']) {
                        bat "ssh -o StrictHostKeyChecking=no ec2-user@$elasticIP ${serverUpdate}; ${dockerInstall}; ${dockerPermissions}; ${dockerStart}" 
                    }
                }
            }
        }
        stage('AWS: Run Docker Container on Dev Server')
        {
            steps
            {
                script
                {
                    def dockerRun = 'docker run -d -p 80:80 helenmgriffin/collegeproject:latest'
                    sshagent(['dev-web-server']) {
                       bat "ssh -o StrictHostKeyChecking=no ec2-user@$elasticIP ${dockerRun}" 
                    }
                }
            }
        }
    }
    post{
      always{
        emailext body: "${currentBuild.currentResult}: Job   ${env.JOB_NAME} build ${env.BUILD_NUMBER}\n More info at: ${env.BUILD_URL}",
        recipientProviders: [[$class: 'DevelopersRecipientProvider'], [$class: 'RequesterRecipientProvider']], 
        subject: "Jenkins Build ${currentBuild.currentResult}: Job ${env.JOB_NAME}"
        }
      }
 }
