# Jenkins Automated Pipeline for Checkmarx Code scan
This is a Jenkins Pipeline script for automated Checkmarx code scan and notification.


## Overview

[Jenkins](https://www.jenkins.io/) is a an open source automation server mainly used to implement Continuous Integration (CI) and Continuous Delivery (CD) for any development project. CI/CD, a key component of a DevOps strategy, allows you to develop the application lifecycle while maintaining quality by automating tasks like building, testing, code scan / code compliance, end-to-end delivery.

Checkmarx (https://checkmarx.com/) allows you to discover the open source in your code and map discovered components to known vulnerabilities. This tutorial is show how Checkmarx scan can be integrated into your dev-ops process for a jenkins pipeline and to monitor your projects in the background and raise alert you as new threats arise.

[Jenkins Pipeline as Code](https://www.jenkins.io/solutions/pipeline/) with the use of Pipeline plugin, jenkins allows to implement a project’s entire build/test/deploy pipeline in a Jenkinsfile that stores alongside the code repository, the pipeline as another piece of code checked into source control.


## Guide

- This is configured as a seperate pipeline since we had to run this scan on a daily basis. 
- This can be integrated in the normal jenkinsFile pipeline script as well, but just to note the requirments as if you don't want to trigger application builds everyday or if you don't need to fail the builds based on the scan results then it is best to keep this a seperate pipeline.

[Jenkins Cron job Syntax](https://www.jenkins.io/doc/book/pipeline/syntax/cron-syntax)

Configure the daily scan values basically using cron job syntax.

```
 // trigger job daily
     triggers {
     pollSCM('H H * * *')
    }
```

## Pipeline Parameters 

If you use either dev or prod environments then you can create custom tag parameters for input in the jenkins gui as below. Also you can configure any other parameters that is required. Email address to be passed as a string vaule was used to send out email notifications once the scan has been completed. Also included is the parameters for the full code scan and incremental scan which can selected when triggering the pipeline job in Jenkins. Note: Default value is incremental scan.

```groovy
      agent any
    parameters {
        string(name: 'REF', defaultValue: 'develop', description: "Commit hash or tag to scan")
        string(name: 'EMAIL', defaultValue: 'emailaddress.mydomain.com', description: 'Email notification')
        booleanParam(name: 'INCREMENTAL', defaultValue: true, description: "Perform an incremental code scan")
        booleanParam(name: 'FORCE_SCAN', defaultValue: false, description: "Force full scan (cannot be run while INCREMENTAL is true)")
    }

```

## Pipeline Stages 

The pipeline stages are

1. clear dir: This clears the workspace directory everytime the pipeline is ran.
2. Checkout: This is mainly generic to trigger the checkout source code management repository.
3. Scan Code: This takes the Git Branch values and intiates the scan. 
4. Scan Code: Also contains has powershell code that just creates the logs folder location.
5. Post Email Notification: Sends out the email notification once the scan has been completed.


```groovy
 withCredentials([string(credentialsId: "checkmarx-api-token", variable: 'CHECKMARX_API_TOKEN')]) {
                bat """ 
                    runCxConsole.cmd AsyncScan \
                    -verbose -CxServer "https://dcxcheckmarx.service.mydomain" \
                    -ProjectName "CxServer/YourProjectNameandPathInCheckMarx" \
                    -CxToken ${CHECKMARX_API_TOKEN} \
                    -LocationType folder \
                    -LocationPath %PROJECT_PATH% \
                    -preset all \
                    -reportpdf "${env.WORKSPACE}\\codescan\\logs\\${PROJECT_STR}_${env.BUILD_NUMBER}.pdf" \
                    -log "${env.WORKSPACE}\\codescan\\logs\\${PROJECT_STR}_${env.BUILD_NUMBER}.log" \
                    """  + "${INCREMENTAL_STR}"
                }
```
**NOTE: Remove the above syntax if it is not required in your pipeline**


Pipeline stage for code scan 
- -verbose -CxServer "https://dcxcheckmarx.service.mydomain" \ # Pass the url of the checkmarx server here.
- -ProjectName "CxServer/YourProjectNameandPathInCheckMarx" \ # Pass the project path name configured in checkmarx here
- -CxToken ${CHECKMARX_API_TOKEN} \ # your check marx api token here # note the variable is also pass as credentials in the script context of code scan
- -LocationType folder \ # location type is mainly folder 
- -LocationPath %PROJECT_PATH% \ # The location path of your project,  normally the workspace of the pipeline is used here.
- -reportpdf "${env.WORKSPACE}\\codescan\\logs\\${PROJECT_STR}_${env.BUILD_NUMBER}.pdf" \ # The path of the scan results pdf location.
- - log "${env.WORKSPACE}\\codescan\\logs\\${PROJECT_STR}_${env.BUILD_NUMBER}.log" \ # logs path


### Function to get your Git Branch Name
```groovy
def getGitBranchName() {
    return scm.branches[0].name
}
```

## Email Notification

Amend the following 

- to: to email groups or addresses normally this is configured by the pipeline parameters as stated above.
- from: from the email if your jenkins server has been configured with the smptp allow to send emails option from the address.
- subject: Modify the subject to your liking.
- body: Modify the body to your linking.
- 
### Post email notification script.
```groovy
 post {
        always {
            emailext (
                to: "${params.EMAIL}",
                from: "MyJenkins@mydomain.com",
                subject: "JENKINS BUILD - ${PROJECT_STR}: ${ENVIRONMENT_STR} - ${currentBuild.currentResult}",
                attachmentsPattern: "codescan/**/logs/*",
                body: """<p><h3><u> JENKINS BUILD DETAIL - ${PROJECT_STR}_${env.BUILD_NUMBER} </u></h3><br>
                        <b>JOB: </b> ${env.JOB_NAME} <br><br>
                        <b>BUILD STATUS: </b> ${currentBuild.currentResult} <br><br>
                        <b>DURATION: </b> ${currentBuild.durationString} <br><br>
                        <b>GIT BRANCH: </b> <a href='${env.GIT_BRANCH_URL}'>${env.GIT_BRANCH}</a> <br><br>
                        <b>CODESCAN RESULT: </b> Refer attached Checkmarx and Checkmarx codescan logs <br>
                        <br>Check console output at <a href='${env.BUILD_URL}'>${env.JOB_NAME} [${env.BUILD_NUMBER}]</a></p>""",
            )  
            
        }
    }
```

## Finally
Have fun implementing this and please don't forget to give credit or star on my github profile.
