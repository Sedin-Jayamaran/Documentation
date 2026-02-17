# Jenkins Disk Cleanup:

## Introduction

- ```cleanup.sh``` script at Jenkins EC2
- This ```cleanup.sh``` will trigger the Clean-up pipeline by
```
curl -X POST \
  "$JENKINS_URL/job/$JOB_NAME/build" \
  --user "$JENKINS_USER:$API_TOKEN"

```
So , This will trigger the pipeline . If needed we can use a cron to run this script every 15 mins inorder to check the disk usage level .

## Script.sh 
### Inside Jenkins server

```
#!/bin/bash

# -------------------------------
# Configuration
# -------------------------------

THRESHOLD=5
JENKINS_URL="http://127.0.0.1:8080"
JOB_NAME="Cleanup"
JENKINS_USER="jayamaran"
API_TOKEN="11d44afa9ed18b0544e5158b4362802dd1"

LOG_FILE="/var/log/jenkins-disk-cleanup.log"

# -------------------------------
# Get Disk Usage
# -------------------------------

USAGE=$(df / | tail -1 | awk '{print $5}' | sed 's/%//')

echo "$(date) - Current disk usage: ${USAGE}%" >> $LOG_FILE

# -------------------------------
# Check Threshold
# -------------------------------

if [ "$USAGE" -lt "$THRESHOLD" ]; then
    echo "$(date) - Disk usage below threshold. Exiting." >> $LOG_FILE
    exit 0
fi

echo "$(date) - Disk usage above threshold. Triggering Jenkins job." >> $LOG_FILE

# -------------------------------
# Trigger Jenkins Pipeline
# -------------------------------

curl -X POST \
  "$JENKINS_URL/job/$JOB_NAME/build" \
  --user "$JENKINS_USER:$API_TOKEN"

echo "$(date) - Jenkins cleanup job triggered." >> $LOG_FILE

```

## Pipeline 

```

while(true){
    def runningBuilds = Jenkins.instance.computers.collectMany {it.executors}.count {it.isBusy()}
    def queuedBuilds = Jenkins.instanse.queue.items.size()
    echo "Running builds: ${runningBuilds}"
    echo "Queueing builds: ${queuedBuilds}"
    if ( runningBuilds <= 1 && queuedBuilds == 0 ){
        echo " Jenkins is idle , Clean up process can begin !
        break
    }
    echo " Jenkins is occupied! "
    sleep 60
    
}
```
The above is the key part , which ensure there is no job is in queueing or bulding phase . 

#### Now look at the full jenkinsfile :

```
pipeline {
    agent any

    environment {
        SLACK_WEBHOOK = "https://hooks.slack.com/services/XXXXXXXXXXXXX"
    }

    stages {

        stage('Wait Until Builds Finish') {
            steps {
                script {

                    echo "Checking Jenkins build and queue status..."

                    while (true) {

                        // Count running builds using Jenkins internal API
                        def runningBuilds = Jenkins.instance.computers
                            .collectMany { it.executors }
                            .count { it.isBusy() }

                        // Count queued builds
                        def queuedBuilds = Jenkins.instance.queue.items.size()

                        echo "Running builds: ${runningBuilds}"
                        echo "Queued builds : ${queuedBuilds}"

                        // <=1 because cleanup job itself is running
                        if (runningBuilds <= 1 && queuedBuilds == 0) {
                            echo "Jenkins is idle. Proceeding to cleanup."
                            break
                        }

                        echo "Builds running or queued. Waiting 60 seconds..."
                        sleep 60
                    }
                }
            }
        }

        stage('Docker Cleanup') {
            steps {
                echo "Cleaning EC2 disk..."

                sh '''
                    docker container prune -f
                    docker image prune -a -f
                    docker builder prune -f
                '''

                echo "Cleanup completed."
            }
        }

        stage('Collect Disk Info') {
            steps {
                script {

                    def disk = sh(
                        script: "df -h / | tail -1",
                        returnStdout: true
                    ).trim().split()

                    env.DISK_TOTAL = disk[1]
                    env.DISK_USED  = disk[2]
                    env.DISK_AVAIL = disk[3]
                    env.DISK_USAGE = disk[4]

                    env.SERVER_NAME = sh(
                        script: "hostname",
                        returnStdout: true
                    ).trim()
                }
            }
        }
    }

    post {

        success {
            sh """
            curl -X POST -H 'Content-type: application/json' \
            --data '{
              "blocks": [
                {
                  "type": "header",
                  "text": {
                    "type": "plain_text",
                    "text": "🧹 Jenkins Disk Cleanup - SUCCESS"
                  }
                },
                {
                  "type": "section",
                  "fields": [
                    { "type": "mrkdwn", "text": "*Total Disk:*\\n${DISK_TOTAL}" },
                    { "type": "mrkdwn", "text": "*Used Space:*\\n${DISK_USED}" },
                    { "type": "mrkdwn", "text": "*Available:*\\n${DISK_AVAIL}" },
                    { "type": "mrkdwn", "text": "*Usage:*\\n${DISK_USAGE}" }
                  ]
                },
                {
                  "type": "context",
                  "elements": [
                    {
                      "type": "mrkdwn",
                      "text": "Server: ${SERVER_NAME}"
                    }
                  ]
                }
              ]
            }' ${SLACK_WEBHOOK}
            """
        }

        failure {
            sh """
            curl -X POST -H 'Content-type: application/json' \
            --data '{
              "text": "❌ Jenkins Cleanup FAILED. Check Jenkins logs."
            }' ${SLACK_WEBHOOK}
            """
        }
    }
}
```
