## Prerequisites:
AWS Account with Admin Privileges
GitHub Account


## Step 1:Configuring EC2 instance in AWS

Go to the AWS dashboard and then to the EC2 services. create an instance

![!\[alt text\](<Project 1/outputs/image.png>)](image.png)

## Step 2:Install Java on Ubuntu 22.04 LTS

After the successful SSH connection, firstly update the Linux machine. And install java using below commands:
```
sudo apt update
Now lets install java 17

sudo apt install openjdk-17-jre
Lets check the version of java

java -version
```
## Step 3:Install Jenkins on Ubuntu 22.04 LTS

Lets install jenkins using below commands
```
curl -fsSL https://pkg.jenkins.io/debian/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian binary/ | sudo tee \
 /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins
```
## Step 4:Enable and start Jenkins on Ubuntu 22.04 LTS

You can enable the Jenkins service to start at boot with the command
```
sudo systemctl enable jenkins
```
You can start the Jenkins service with the command
```
sudo systemctl start jenkins
```
You can check the status of the Jenkins service using the command
```
sudo systemctl status jenkins
```

## Step 5:Install git on Ubuntu 22.04 LTS

We need to Install git using below command
```
sudo apt install git
```
## Step 6:Access Jenkins on Browser

https//:<Instance_ip>:8080

After that On the browser, you should see the Jenkins interface that asks for the administrator password.


Now cat the following Jenkins file to retrieve the Administrator password and paste it to the Jenkins dashboard.

Here, create a Jenkins user, here I'm using Jenkins as admin user

After the configuration is completed, you should see the Jenkins dashboard.

![image](https://github.com/shivagorasa/DemoRepo/assets/97184376/f63e8238-92d5-45ec-baec-1c9f1ae1c1ab)


## Step 7:Add AWS credentials in Jenkins

We may also set up AWS credentials in Jenkins so that it facilitates the Docker push to the ECR repository.

GO to the Manage Jenkins>>Credentials>>system>>Global credentials

Then add credentials and here add AWS username and password and account ID

![image](https://github.com/shivagorasa/DemoRepo/assets/97184376/670bdc9b-334a-4d64-a902-27240e28d0a4)



## Step 8:Install Docker on Ubuntu 22.04 LTS

Now here we need to Install Docker

```
sudo apt  install docker.io
After Installing Docker we need to give some permission

sudo usermod -aG docker $USER
sudo chmod 666 /var/run/docker.sock
After installing docker lets Restart jenkins

sudo systemclt restart jenkins
```

## Step #9:Installing plugins in Jenkins

Go to the manage Jenkins>>Plugins>>Available Plugin

1. Docker
2. Docker Pipeline
3. Amazon ECR plugin

## Step #10:Creating ECR Repository in AWS

Lets Create AWS ECR repository to push this image so Go to AWS ECR repository and created with name **samplewebappecr**

![image](https://github.com/shivagorasa/DemoRepo/assets/97184376/f05bbda7-4ab1-4a62-a1e2-8b49c992aa3d)


## Step #11:Create AmazonEC2ContainerRegistryFullAccess IAM Role in AWS

Here in this step we need to create IAM role with below permission 

Attach permission policies : AmazonEC2ContainerRegistryFullAccess


## Step 12:Install AWS CLI on Ubuntu 22.04 LTS

You can go to the official site of AWS and Install to allow jenkins to authorize and use aws ecr service

```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip 
sudo ./aws/install
```

## Step 13:Push Docker image to AWS ECR using Jenkins pipeline

So lets create jenkins pipeline go to the Jenkins Dashboard Click on new Item 

select Pipeline 

then select github project and use github repo URl **https://github.com/shivagorasa/DemoRepo.git/**

Here we have following Dockerfile as follows

```
# Using official Nginx image as the base image
FROM nginx:1.19.10

COPY nginx.conf /etc/nginx/nginx.conf

# Copy our web application files into the Nginx document root
COPY . /var/www/html 

WORKDIR /var/www/html -

# Expose port 80 to the outside world
EXPOSE 80  

CMD ["nginx", "-g", "daemon off;"]
```

## nginx.conf replaced with following in same app directory

```
events {

}

http {
    # Configure HTTP server
    server {
        listen 80;              # Listen on port 8080
        server_name localhost;  # Set your server name here

        # Define the location of the root directory
        root /var/www/html;

        # Specify the default file to serve
        index index.html;

        # Configure error and access logs
        error_log /var/log/nginx/error.log;
        access_log /var/log/nginx/access.log;

        # Configure location rules
        location / {
            try_files $uri $uri/ /index.html;
        }

        # Handle requests for favicon.ico
        location = /favicon.ico {
            access_log off;
            log_not_found off;
        }
    }
}
```

## then under pipeline select Pipeline script and use following code :

```
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

```

## Let's add Email Notification:

add following under jenkins configure > email notification and extended email notification

Enter the SMTP server name under ‘Email Notification’. Click the ‘Advanced’ button and then click the checkbox next to the ‘Use SMTP Authentication’ option. Now, set the following fields.

* SMTP server name : smtp.gmail.com
* User name: user_email_id@gmail.com
* Password: 123456
* Use SSL : Checked
* SMTP Port: 465

![image](https://github.com/shivagorasa/DemoRepo/assets/97184376/edd9054f-2a57-4500-87b7-47f88da7f18a)


## After this click on build now in jenkins pipeline we see following output if everything goes well

```
Console Output
Started by user admin
[Pipeline] Start of Pipeline
[Pipeline] node
Running on Jenkins in /var/lib/jenkins/workspace/TestPipeline
[Pipeline] {
[Pipeline] withEnv
[Pipeline] {
[Pipeline] stage
[Pipeline] { (Logging into AWS ECR)
[Pipeline] script
[Pipeline] {
[Pipeline] sh
+ docker login --username AWS --password-stdin 992382689324.dkr.ecr.us-east-1.amazonaws.com
+ aws ecr get-login-password --region us-east-1
WARNING! Your password will be stored unencrypted in /var/lib/jenkins/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
[Pipeline] }
[Pipeline] // script
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Cloning Git)
[Pipeline] checkout
Selected Git installation does not exist. Using Default
The recommended git tool is: NONE
No credentials specified
 > git rev-parse --resolve-git-dir /var/lib/jenkins/workspace/TestPipeline/.git # timeout=10
Fetching changes from the remote Git repository
 > git config remote.origin.url https://github.com/shivagorasa/DemoRepo.git # timeout=10
Fetching upstream changes from https://github.com/shivagorasa/DemoRepo.git
 > git --version # timeout=10
 > git --version # 'git version 2.34.1'
 > git fetch --tags --force --progress -- https://github.com/shivagorasa/DemoRepo.git +refs/heads/*:refs/remotes/origin/* # timeout=10
 > git rev-parse refs/remotes/origin/main^{commit} # timeout=10
Checking out Revision d18f79c133f8b2c080954c6cae2b47d9d35a60df (refs/remotes/origin/main)
 > git config core.sparsecheckout # timeout=10
 > git checkout -f d18f79c133f8b2c080954c6cae2b47d9d35a60df # timeout=10
Commit message: "final commit"
 > git rev-list --no-walk d18f79c133f8b2c080954c6cae2b47d9d35a60df # timeout=10
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Building image)
[Pipeline] script
[Pipeline] {
[Pipeline] isUnix
[Pipeline] withEnv
[Pipeline] {
[Pipeline] sh
+ docker build -t samplewebappecr:30 .
#0 building with "default" instance using docker driver

#1 [internal] load build definition from Dockerfile
#1 transferring dockerfile: 341B 0.0s done
#1 DONE 0.1s

#2 [internal] load metadata for docker.io/library/nginx:1.19.10
#2 DONE 0.5s

#3 [internal] load .dockerignore
#3 transferring context: 2B done
#3 DONE 0.0s

#4 [1/4] FROM docker.io/library/nginx:1.19.10@sha256:df13abe416e37eb3db4722840dd479b00ba193ac6606e7902331dcea50f4f1f2
#4 DONE 0.0s

#5 [internal] load build context
#5 transferring context: 7.44kB 0.1s done
#5 DONE 0.1s

#6 [2/4] COPY nginx.conf /etc/nginx/nginx.conf
#6 CACHED

#7 [3/4] COPY . /var/www/html
#7 CACHED

#8 [4/4] WORKDIR /var/www/html -
#8 CACHED

#9 exporting to image
#9 exporting layers done
#9 writing image sha256:82a5166c374519ca0e8d7937e9c7a98aa455119ac5483903f3257b9d839e6ee2 0.0s done
#9 naming to docker.io/library/samplewebappecr:30 0.0s done
#9 DONE 0.1s
[Pipeline] }
[Pipeline] // withEnv
[Pipeline] }
[Pipeline] // script
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Pushing to ECR)
[Pipeline] script
[Pipeline] {
[Pipeline] sh
+ docker tag samplewebappecr:30 992382689324.dkr.ecr.us-east-1.amazonaws.com/samplewebappecr:30
[Pipeline] sh
+ docker push 992382689324.dkr.ecr.us-east-1.amazonaws.com/samplewebappecr:30
The push refers to repository [992382689324.dkr.ecr.us-east-1.amazonaws.com/samplewebappecr]
01109db3d6f5: Preparing
bfcf9d7830f3: Preparing
e0767a2d8a8f: Preparing
f0f30197ccf9: Preparing
eeb14ff930d4: Preparing
c9732df61184: Preparing
4b8db2d7f35a: Preparing
431f409d4c5a: Preparing
02c055ef67f5: Preparing
c9732df61184: Waiting
4b8db2d7f35a: Waiting
431f409d4c5a: Waiting
02c055ef67f5: Waiting
01109db3d6f5: Layer already exists
e0767a2d8a8f: Layer already exists
eeb14ff930d4: Layer already exists
bfcf9d7830f3: Layer already exists
f0f30197ccf9: Layer already exists
c9732df61184: Layer already exists
4b8db2d7f35a: Layer already exists
431f409d4c5a: Layer already exists
02c055ef67f5: Layer already exists
30: digest: sha256:5eeb5b9d4c9191e845a4acf66d86594ec84edb0b8e2b9349ee5263d0e627fa3b size: 2195
[Pipeline] }
[Pipeline] // script
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Running Docker Container)
[Pipeline] script
[Pipeline] {
[Pipeline] sh
+ docker run -d -p 8000:80 992382689324.dkr.ecr.us-east-1.amazonaws.com/samplewebappecr:30
236ded7d693f044a2fe095b7d80a7ed7d87c0817974d31e8933ab8ff5354fe25
[Pipeline] }
[Pipeline] // script
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Email Notification)
[Pipeline] mail
[Pipeline] }
[Pipeline] // stage
[Pipeline] }
[Pipeline] // withEnv
[Pipeline] }
[Pipeline] // node
[Pipeline] End of Pipeline
Finished: SUCCESS

```

## Jenkins Output 

![image](https://github.com/shivagorasa/DemoRepo/assets/97184376/71922b9f-3e8a-47f4-8e1d-ec582928ad90)


## Modify security groups to allow access to required ports

![image](https://github.com/shivagorasa/DemoRepo/assets/97184376/4f33e303-0bd1-4a29-92d5-7785a3eb5225)


## Now lets Check ECR Repo our image push or not

![image](https://github.com/shivagorasa/DemoRepo/assets/97184376/4c83c38e-60d9-49fd-99a3-76e51bd31e5e)


## Email Notification:

![image](https://github.com/shivagorasa/DemoRepo/assets/97184376/770a6ae3-d24a-435b-95bb-e7f51bc514cf)


## we can access our app running on port 8000

![image](https://github.com/shivagorasa/DemoRepo/assets/97184376/d20de53f-4329-4458-9a1a-71c899cb851e)


## App running on port 8000 can be accessed as follows:

![image](https://github.com/shivagorasa/DemoRepo/assets/97184376/30c4853c-9be2-4c42-b716-0d93f3d01458)



## Conclusion:

Hence we can conclude we have built CI/CD pipeline using Jenkins and deployed our e-com app in AWS cloud and our image onto Amazon ECR.


