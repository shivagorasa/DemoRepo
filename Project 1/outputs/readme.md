## Project Description:
Build CI / CD Pipeline using Jenkins and deploy the real world Web Application in AWS Cloud
## Goals:
CI/CD Pipelines will help you learn
Server automation, continuous integration, building pipelines, configuration of tools, automated testing, code quality improvement, and distributed systems in Jenkins through intensive, hands-on practical assignments.
## Technologies Used:-
1. Jenkins
2. Groovy
3. AWS Cloud
4. Git
5. Docker

## Steps:
1. Create jenkins file using our in-house code repo [should be cloned from
git/bitbucket]
2. Create Docker file in the same repository
3. Build-Docker image with tagging as build version, unit test cases should pass if
any for the code
4. The Image should be available in ECR with build version as TAG
5. The Docker Image should be deployed to EC2 Machine
6. The EC2 Machine Need to open specific Inbound Port and restrict Access only for
admin user to login
7. Jenkins Jobs should do validation and display successful message
8. Report should be sent to e-mail and it should contain status of each JOB
9. Domain should be registered with AWS

