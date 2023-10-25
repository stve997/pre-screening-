Approach
Prerequisites:
•	A server with Jenkins & Docker configured
•	Docker registry
•	GitHub repo initialized 
•	Local & Deployment server
•	Containerized application
I would use Jenkins as the main CI/CD tool for setting up a pipeline for an application that has been containerized using Docker and its repo setup on GitHub. 
The pipeline will undergo 4 stages; build, test, staging & deploy. 
I settled for Jenkins as it being open source, its easy to setup, supports most environments & provides plugins for integration with other systems.

Implementation
We first need to setup GitHub webhooks for the repo that we will use in the pipeline. 
Navigate to Settings > Webhooks to set it up. 
Provide Jenkins URL as the payload of the webhook adding ‘/gitHub-webhook’ at the end.
Choose application/json as Content-Type from the drop-down menu, select Just the push event, and click the Add webhook.
Next, we need to setup the Jenkins project. 
On the Jenkins dashboard, click on “New Item”. 
After that, enter the name of your project, choose “Pipeline” from the list.
Under build triggers check the box with GitHub hook trigger for GITScm polling.
In the Pipeline section set the pipeline script to be from SCM and specify SCM as git. 
Provide the repository URL and your GitHub credentials. 
Specify the branch which the pipeline will be executed and file path of the Jenkins file on your repo the save the project.
Now on a local server, access your repo and make a Jenkins file at the root. 
This file is the one that will be used to run the pipeline. 
Configure the pipeline with the stages build, test, staging and deploy. 
On commit, the webhook configured will trigger the Jenkins job where the pipeline will build the application, pass it through some tests, stage the docker images to a registry then deploy the application.
Each stage of the pipeline will have a on-failure post action that will be sending email alerts to personnel in charge of the pipeline and application.
The CI/CD implementation is now complete, with each push GitHub will trigger the Jenkins job.
Sample of the Jenkins file is as below:
pipeline {
    agent any

    environment{
        imageName = 'prodtrace_frontend'
        registryCredential = 'dockerhub'
        stagingRegistryUrl = 'http://xxx.xxx.xx.xx:5000'
        productionRegistryUrl = 'http://xxx.xxx.xx.xx:5000'
        dockerImage = ''
        STAGING_SERVER ='xxx.xxx.xx.xx'
        PRODUCTION_SERVER ='xxx.xxx.xx.xx'
        mailRecepients= 'example@exapmle.com, example2@example.com'
    }
    
    stages {
        stage('Cloning our Git') {
            steps {
                echo 'Pulling....' + env.gitlabBranch
                git(url:'http://192.168.0.207/prodtrace/prodtrace-frontend.git', branch: env.gitlabBranch , credentialsId: 'GIT_CRED')
            }
        }

        stage ('BUILD') {
            steps{
                script {
                       if (env.gitlabBranch == 'master') {
                            dockerImage = docker.build("$imageName:$BUILD_NUMBER"," -f Dockerfile.production .")
                            docker.withRegistry( productionRegistryUrl,  ) {
                            dockerImage.push()
                            dockerImage.push("latest")
                            }
                        } else {
                            dockerImage = docker.build("$imageName:$BUILD_NUMBER", "-f Dockerfile.staging .")
                            docker.withRegistry( stagingRegistryUrl,  ) {
                            dockerImage.push()
                            dockerImage.push("latest")
                        }
                    }
                }
                sh '''docker rmi $imageName:$BUILD_NUMBER'''
            }

            post{
                failure{
                    mail to: "${mailRecepients}",
                    subject: "jenkins build:${currentBuild.currentResult}: ${env.JOB_NAME}",
                    body: "${currentBuild.currentResult}: Job ${env.JOB_NAME}\nMore Info can be found here: ${env.BUILD_URL}"
                }
            }
        }

        stage ('SERVER SETUP') {
            steps{
                sshagent(credentials:['JENKINS']) {
                    script{
                        if (env.gitlabBranch == 'master') {
                            sh "scp prod-setup.sh  $PRODUCTION_SERVER:~/"
                            sh 'ssh  -o StrictHostKeyChecking=no  $PRODUCTION_SERVER "chmod +x prod-setup.sh ; ./prod-setup.sh"'
                            sh 'scp docker-compose-prod.yml $PRODUCTION_SERVER:~/prodtrace-frontend/deployment/current/'
                        } else {
                            sh "scp staging-setup.sh  $STAGING_SERVER:~/"
                            sh 'ssh  -o StrictHostKeyChecking=no  $STAGING_SERVER "chmod +x staging-setup.sh ; ./staging-setup.sh"'
                            sh 'scp docker-compose-staging.yml $STAGING_SERVER:~/prodtrace-frontend/deployment/current/'
                        }
                    }

                 }
            }

             post{
                failure{
                    mail to: "${mailRecepients}",
                    subject: "jenkins build:${currentBuild.currentResult}: ${env.JOB_NAME}",
                    body: "${currentBuild.currentResult}: Job ${env.JOB_NAME}\nMore Info can be found here: ${env.BUILD_URL}"
                }
            }
        }
        stage('deploying prodtrace-frontend') {
            steps{
                sshagent(credentials:['JENKINS']) {
                    script{
                        if (env.gitlabBranch == 'master') {
                        sh "ssh -o StrictHostKeyChecking=no  $PRODUCTION_SERVER docker-compose -f prodtrace-frontend/deployment/current/docker-compose-prod.yml up -d"

                        } else {
                        sh "ssh -o StrictHostKeyChecking=no  $STAGING_SERVER docker-compose -f prodtrace-frontend/deployment/current/docker-compose-staging.yml up -d"
                        }
                    }
                }  
            }

            post{
                success{
                    mail to: "${mailRecepients}",
                    subject: "jenkins build:${currentBuild.currentResult}: ${env.JOB_NAME}",
                    body: "${currentBuild.currentResult}: Job ${env.JOB_NAME}\nMore Info can be found here: ${env.BUILD_URL}"
                }
                failure{
                    mail to: "${mailRecepients}",
                    subject: "jenkins build:${currentBuild.currentResult}: ${env.JOB_NAME}",
                    body: "${currentBuild.currentResult}: Job ${env.JOB_NAME}\nMore Info can be found here: ${env.BUILD_URL}"
                }
            }
        }
    }   
}
