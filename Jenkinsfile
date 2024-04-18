pipeline {
    agent any
    
    environment {
        AWS_ACCOUNT_ID="992382689324"  <our aws id goes here>
        AWS_DEFAULT_REGION="us-east-1" <our region goeg here>
        IMAGE_REPO_NAME="samplewebappecr" <name of ecr repo>
        IMAGE_TAG="${BUILD_NUMBER}"
        REPOSITORY_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${IMAGE_REPO_NAME}"
        CONTAINER_PORT="8000"
    }
   
    stages {
        stage('Logging into AWS ECR') {
            steps {
                script {
                    sh """aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com"""
                }
            }
        }
        
        stage('Cloning Git') {
            steps {
                checkout([$class: 'GitSCM', branches: [[name: '*/main']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: '', url: 'https://github.com/shivagorasa/DemoRepo.git']]])     
            }
        }
  
        // Building Docker images
        stage('Building image') {
            steps {
                script {
                    dockerImage = docker.build("${IMAGE_REPO_NAME}:${IMAGE_TAG}", '.')
                }
            }
        }
   
        // Uploading Docker images into AWS ECR
        stage('Pushing to ECR') {
            steps {  
                script {
                    sh """docker tag ${IMAGE_REPO_NAME}:${IMAGE_TAG} ${REPOSITORY_URI}:${IMAGE_TAG}"""
                    sh """docker push ${REPOSITORY_URI}:${IMAGE_TAG}"""
                }
            }
        }

        // Running Docker container after pushing to ECR
        stage('Running Docker Container') {
            steps {
                script {
                    sh "docker run -d -p ${CONTAINER_PORT}:80 ${REPOSITORY_URI}:${IMAGE_TAG}"
                }
            }
        }

        // Email Notification stage
        stage('Email Notification') {
            steps {
                mail to: 'shiva789111@gmail.com',
                     subject: 'Build successful',
                     body: '''
                            Build successful!!!!

                            Thanks,
                            Shiva
                            '''
            }
        }
    }
}
