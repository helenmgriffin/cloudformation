def instanceID = ''
def $elasticIP = ''

pipeline{
    agent any
    stages{
        stage('Get CloudFormation File') {
           steps {
             git url: 'https://github.com/helenmgriffin/cloudformation.git', branch: 'main'
             }
        }
        stage('Tear Down Dev Server')
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
        stage('Provision Dev Server')
        {
            steps
            {
                script
                {
                    withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'AWSCredentails2', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                        bat 'aws cloudformation create-stack --stack-name CollegeProjectDevServer --template-body file://CollegeProjectEC2.yaml --parameters ParameterKey=InstanceType,ParameterValue=t2.micro ParameterKey=KeyName,ParameterValue=DevWebServer ParameterKey=SSHLocation,ParameterValue=40.113.68.180/32'
                        bat 'aws cloudformation wait stack-create-complete --stack-name "CollegeProjectDevServer'
                    }
                }
            }
        }
        stage('Install Docker on Dev Server')
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
        stage('Run Docker Container on Dev Server')
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
