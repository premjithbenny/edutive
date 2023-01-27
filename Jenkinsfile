pipeline {
    agent any
    environment {
        
        IMAGE_NAME = "mail-service"
        def ECR_FULL_REPO_NAME = "481597576047.dkr.ecr.ap-south-1.amazonaws.com/"+"$IMAGE_NAME"
        BUILD_NO="pre-prod-1.0.$BUILD_NUMBER"
        CONTAINER_PORT="8097:8097"
                    
    // Dev env 
    
    
        machine_username="ubuntu"
        machine_ip_address="10.0.0.85"
        //real dev server ip = "10.0.0.196"
        //jenkins-server ip= 10.0.0.85
        
    
        
// DO NOT EDIT
         docker_colm="\$"+"3"
         def IMAGE_TEMP = ""
         doubleq= "\""
         ECRURL = "https://481597576047.dkr.ecr.ap-south-1.amazonaws.com"
         ECR_CREDS="ecr:ap-south-1:AWS_ECR_CRED"
        
    }
    stages {
        //Clone code from Bitbucket dev-cicd-mail-service branch
        stage('Cloning Git') {
            steps {
                checkout([$class: 'GitSCM', branches: [[name: '*/pre-prod-cicd-mail-service']], extensions: [], userRemoteConfigs: [[credentialsId: 'EDUTIVE-REPO-CRED', url: 'https://edu-tive-admin@bitbucket.org/edu-tive/mail-service/']]])     
            }
        }
        
        // build code with gradle
        
        stage('gradle') {
            steps {
                sh '/opt/gradle/gradle-7.4/bin/gradle clean build -x test'
                sh 'ls -alh'
                sh 'ls -alh ./build/libs/'
            }
            
        }
        
        // Scan and test code with SonarQube-scannr
        
//          stage('Sonar-scannner') {
//           environment {
//           def scannerHome = tool 'SONAR_QUBE_SCANNER'
//          def squrl="http://10.0.0.85:9000"
//          }
//            steps {
//               withSonarQubeEnv(installationName: 'SONAR_QUBE_SCANNER',credentialsId: 'SONARQUBE-TOKEN') {
//               sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=$IMAGE_NAME  -Dsonar.sources=. -Dsonar.java.source=1.8 -Dsonar.java.binaries=build/classes "
//        
//                }
//             }
//         }
        
//        Check the quality of SonarQube scanner report
//        
//         stage("Quality gate") {
//             steps
//             {
//                 waitForQualityGate abortPipeline: true
//            }
//          }
        
        // If report passed then build docker image with jar
        
        stage('Docker build') {
            steps{
                script {
                    IMAGE_TEMP = docker.build(IMAGE_NAME)
                    echo "Docker build successfully completed."
            
                }
            }
        }
        
        // Push docker image into AWS ECR registory
        
        stage('Push docker image to ECR'){
            steps{
                script {
                   docker.withRegistry(ECRURL,ECR_CREDS) {             
                        IMAGE_TEMP.push(BUILD_NO)
                        IMAGE_TEMP.push("latest")                   
                    }
                    echo "Image uploaded to ECR sucessfully "
                }
            }   
        }
        
        // Delete temprory images from jenkins server
        
        stage('Remove Unused docker image from Jenkins server') {
            steps {
                script {
                    sh '''
                    
                        docker rmi -f $ECR_FULL_REPO_NAME:$BUILD_NO
                        docker rmi -f $IMAGE_NAME
                        docker rmi -f $ECR_FULL_REPO_NAME
                    
                    '''
                }
            }
        }
        
        // Deploy in Dev server if environment is 'Dev'
        
        stage('Pre-prod ENV Deployment '){
//          when {
//               environment name: 'ENV_FOR_DEPLOYMENTS', value: 'dev'
//                expression{env.ENV_FOR_DEPLOYMENTS=='dev'}
//          }
            steps {
                script{
                // Change sshagent value according to remote server like for 
                // Dev-     DEV_ENV_APP_SERVER_MACHINE_KEYPAIR
                //Jenkins-  JENKINS_MACHINE_KEYPAIR
                
                    sshagent(['JENKINS_MACHINE_KEYPAIR']) {
                        try {
                            sh "ssh -o StrictHostKeyChecking=no $machine_username@$machine_ip_address \"aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin 481597576047.dkr.ecr.ap-south-1.amazonaws.com && docker pull $ECR_FULL_REPO_NAME:$BUILD_NO \" "
                            sh "ssh -o StrictHostKeyChecking=no  $machine_username@$machine_ip_address \"docker-compose -f /home/ubuntu/edutive/docker-compose.yml down \" "
                        
                        }
                        catch (err) {
                            echo 'caught error: '
                        }
                        try {
                            sh "ssh -o StrictHostKeyChecking=no  $machine_username@$machine_ip_address \"yq -i '.services.\$IMAGE_NAME.image =\$doubleq\$ECR_FULL_REPO_NAME:$BUILD_NO\$doubleq' /home/ubuntu/edutive/docker-compose.yml \" "
    
                            sh "ssh -o StrictHostKeyChecking=no  $machine_username@$machine_ip_address \"docker rm -f $IMAGE_NAME \" "
                        }
                        catch (yq) {
                            echo 'caught error: yq'
                        }
                        sh "ssh -o StrictHostKeyChecking=no  $machine_username@$machine_ip_address \"docker-compose -f /home/ubuntu/edutive/docker-compose.yml up -d \" "
                    
                  
                        try {
                            sh '''
                        
                               ssh -o StrictHostKeyChecking=no $machine_username@$machine_ip_address \"docker images -a | grep $ECR_FULL_REPO_NAME | awk '{print \$docker_colm}'| xargs docker rmi \"
                        
                            '''
                        }
                        catch(err1){
                            echo 'caught error:'
                        }
                    
                    }
                }
               
            }
        }
           
    }
}

