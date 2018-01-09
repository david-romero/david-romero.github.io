---
layout: post
title:  "Continuous integration with Jenkins Pipeline"
date:   2017-04-01 15:00:00
categories: [tutorial,jenkins,CI,maven]
comments: true
---
Since the beginning of 2017, I have been instilling in my company the importance and necessity of having a strong CI environment. I started installing Jenkins in a small server and right now I've got a project to implement continuous integration for all the development teams in the company.

In this post I would like to share many of the acquired knowledge.

## Workflow definition

First of all, we have designed a workflow for all projects. If each project pass the workflow we can achieve a good quality in the software factory.
Besides, we can automatize every deploy done to each environment.

Attached a diagram with the proposal workflow.

![Spring Initializr]({{ "/img/workflow.png" | absolute_url }})

The workflow contains the following stages:

1. Checkout the code
2. Compile the code
3. Execute tests
4. Quality code analisys, check coverage and OWASP analysis.
5. Deploy to pre-production environment
6. User confirmation
7. Tag the code
8. Deploy to production environment
9. Publish to Slack and Jira

## Workflow implementation

I think the version 2 of Jenkins is the most appropiate tool for this job, since they have incorporated a new feature to create "pipelines" with code. These pipelines are groovy scripts.

So I got down to work and started working with Jenkins Pipeline not without giving me the some headache.

I chose Maven as the main tool since with different plugins it allows me to carry out most of the actions of the workflow. I could also have used the Jenkins plugins but they are not updated to support the pipeline.

Attached code:

{% highlight groovy %}
#!groovy

pipeline {
     agent any    //Docker agent. In the future, we will work with docker
     tools { //Jenkins installed tools
        maven 'M3' //Maven tool defined in jenkins configuration
        jdk 'JDK8' //Java tool defined in jenkin configuration
    }
    options {
        //If after 3 days the pipeline does not finish, please abort
        timeout(time: 76, unit: 'HOURS') 
    }
    environment {
        //Project name
        APP_NAME = 'My-App'
    }
    stages { //Stages definition
       stage ('Initialize') { //Send message to slack at the beginning
             steps {
                  slackSend (message: 'Start pipeline ' + APP_NAME, channel: '#jenkins', color: '#0000FF', teamDomain: 'my-company', token: 'XXXXXXXXXXXXXXXXXXX' )
            }
       }
       stage ('Build') { //Compile stage
            steps {
                 bat "mvn -T 4 -B --batch-mode -V -U -e -Dmaven.test.failure.ignore clean package -Dmaven.test.skip=true"
            }
       }
       stage ('Test') {
            //Tests stage. We use parrallel mode.
            steps {
                 parallel 'Integration & Unit Tests': {
                     bat "mvn -T 4 -B --batch-mode -V -U -e test"
                 }, 'Performance Test': {
                     bat "mvn jmeter:jmeter"
                 }
           }
       }
       stage ('QA') {
       //QA stage. Parallel mode with Sonar, Coverage and OWASP
           steps {
                parallel 'Sonarqube Analysis': {
                    bat "mvn -B --batch-mode -V -U -e org.jacoco:jacoco-maven-plugin:prepare-agent install -Dmaven.test.failure.ignore=true"
                    bat "mvn -B --batch-mode -V -U -e sonar:sonar"
               }, 'Check code coverage' : {
                    //Check coverage
                    //If coverage is under 80% the pipeline fails.
                    bat "mvn -B --batch-mode -V -U -e verify"
               }, 'OWASP Analysis' : {
                    bat "mvn -B -X --batch-mode -V -U -e dependency-check:check"
               }
          }
          //We store tests reports.
          post {
               success {
                    junit 'target/surefire-reports/**/*.xml' 
               }
          }
      }
      stage ('Deploy to Pre-production environment') {
      //We use maven cargo plugin to deploy in tomcat.
           steps {
                bat "mvn -B -P Desarrollo --batch-mode -V -U -e clean package cargo:redeploy -Dmaven.test.skip=true"
           }
      }
      stage ('Confirmation') {
      //In this stage, pipeline wait until user confirm next stage.
      //It sends slack messages
           steps {
                slackSend channel: '@boss',color: '#00FF00', message: '\u00BFDo you want to deploy to production environment?. \n Link: ${BLUE_OCEAN_URL}' , teamDomain: 'my-company', token: 'XXXXXXXXXXX'
                timeout(time: 72, unit: 'HOURS') {
                    input 'Should the project ' + APP_NAME + ' be deployed to production environment\u003F'
                }
           }
      }
      stage ('Tagging the release candidate') {
           steps {
               //Tagging from trunk to tag
               echo "Tagging the release Candidate";
               bat "mvn -B --batch-mode -V -U -e scm:tag -Dmaven.test.skip=true"
          }
      }
      stage ('Deploy to Production environment') {
           //We deploy in parrallel mode during 6 times. 
           steps {
                parallel 'Server 1': {
                    retry(6) {
                        bat "mvn -T 4 -B -P Produccion --batch-mode -V -U -e tomcat7:redeploy -Dmaven.test.skip=true"
                    }
                }, 'Server 2' : {
                    retry(6) {
                        bat "mvn -T 4 -B -P Produccion --batch-mode -V -U -e tomcat:redeploy -Dmaven.test.skip=true"
                    }
                }
           }
      }
      stage ('CleanUp') {
      //The pipeline remove all temporal files.
           steps {
                deleteDir()
           }
      }
    } //End of stages
    //Post-workflow actions.
    //The pipeline sends messages with the result of the execution
    post {
      success {
           slackSend channel: '#jenkins',color: '#00FF00', message: APP_NAME + ' executed successfully.', teamDomain: 'my-company', token: 'XXXXXXXXXXXXXXXXXXXX'
      }
      failure {
           slackSend channel: '#jenkins',color: '#FF0000', message: APP_NAME + ' is failure!!!. ${BLUE_OCEAN_URL}', teamDomain: 'my-company', token: 'XXXXXXXXXXXXXXXX'
      }
      unstable {
           slackSend channel: '#jenkins',color: '#FFFF00', message: APP_NAME + ' is unstable!!!. ${BLUE_OCEAN_URL}', teamDomain: 'my-company', token: 'XXXXXXXXXXXXXXXXXXXX'
      }
    }
   }
{% endhighlight %}

With Jenkins we can automate a lot of task that developer would have to do instead of develop code, in addition those task are prone to failures, with which we remove that possibility. We can also impose a series of quality rules in the code that have to be fulfilled if we want to deploy a new version.

Below is shown a real execution of the pipeline in Jenkin. Our Jenkins has installed the [Blue Ocen Plugin](https://jenkins.io/projects/blueocean/). This plugins improve the Jenkins UI. 

![Pipeline execution]({{ "/img/workflow2.png" | absolute_url }})

#### Next Steps:

* Docker integration. Execute stages in docker containers.
* Ansible integration. Deploy to multiple servers with one command.
* Jira Api Rest. Publish in Jira the release notes.

If you have any doubt, please contact me or leave a comment and so I can help anybody interested in Jenkins pipeline :)

Links:

* [https://jenkins.io/solutions/pipeline/](https://jenkins.io/solutions/pipeline/)
* [https://jenkins.io/blog/2017/02/07/declarative-maven-project/](https://jenkins.io/blog/2017/02/07/declarative-maven-project/)
* [https://github.com/jenkinsci/pipeline-examples](https://github.com/jenkinsci/pipeline-examples)
* [https://codehaus-cargo.github.io/cargo/Maven2+Plugin+Getting+Started.html](https://codehaus-cargo.github.io/cargo/Maven2+Plugin+Getting+Started.html)
* [https://github.com/jenkinsci/slack-plugin](https://github.com/jenkinsci/slack-plugin)