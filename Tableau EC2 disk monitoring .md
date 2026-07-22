# Tableau EC2 Disk Monitoring and Alerting Walkthrough

We have successfully created and configured an automated monitoring and alerting system for your Tableau EC2 instance (`AL2023`) that integrates with your Microsoft Teams channel via a Power Automate webhook workflow.

---

## What was Accomplished

1. **Microsoft Teams Webhook Workflow**: Leveraged the new Teams Workflows template to configure a webhook URL capable of processing modern JSON Adaptive Cards.
2. **Dynamic Monitoring Script**: Created a bash script `/usr/local/bin/disk_monitor.sh` with dual operation modes:
   * **Monitor Mode**: Silently checks partition usage and triggers an immediate Adaptive Card notification if it exceeds 75%.
   * **Report Mode (`--report`)**: Bypasses the threshold check to generate and send a detailed system health status card showing Hostname, Instance ID, RAM usage, and mount point spaces.
3. **AWS metadata collection**: Utilized AWS IMDSv2 to safely query the EC2 Instance ID from inside the OS to include in the notifications.
4. **Cron Scheduler Configuration**: Installed `cronie` on the Amazon Linux 2023 instance and automated execution.

---

## File Details

### Script: [disk_monitor.sh](file:///Users/jayamaran/.gemini/antigravity/scratch/tableau_disk_monitor/disk_monitor.sh)
Located on the EC2 instance at `/usr/local/bin/disk_monitor.sh`. Below is the complete codebase deployed:

```bash
#!/bin/bash

# Configuration
THRESHOLD=75
DISK_PARTITION="/"
WEBHOOK_URL="https://defaultd2624b5b89fd468eb5046570779aa3.b6.environment.api.powerplatform.com:443/powerautomate/automations/direct/workflows/1fd3da521d7549818936b7bb08703337/triggers/manual/paths/invoke?api-version=1&sp=%2Ftriggers%2Fmanual%2Frun&sv=1.0&sig=hdE8IkqFcUHR9riqJ0qdfohdSYIWHvwEn74EuJ3pJvs"

# Get server hostname and IP address
HOSTNAME=$(hostname)
IP_ADDR=$(hostname -I | awk '{print $1}')

# Fetch AWS EC2 Instance ID using metadata service (IMDSv2)
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 60")
INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id)

# If IMDSv2 fails (e.g. not on AWS), default to hostname
if [ -z "$INSTANCE_ID" ]; then
    INSTANCE_ID=$HOSTNAME
fi

# Check disk usage percentage (gets the number only, e.g. 42)
CURRENT_USAGE=$(df -h "$DISK_PARTITION" | grep -v Filesystem | awk '{print $5}' | sed 's/%//g')

# Fallback check if CURRENT_USAGE is empty
if [ -z "$CURRENT_USAGE" ]; then
    echo "Error: Could not retrieve disk usage for $DISK_PARTITION"
    exit 1
fi

# Function to send emergency alert
send_emergency_alert() {
    echo "Disk usage ($CURRENT_USAGE%) has exceeded the threshold ($THRESHOLD%). Sending alert to Teams..."

    # Construct the JSON payload using Adaptive Card format expected by Teams Workflows
    PAYLOAD=$(cat <<EOF
{
  "type": "AdaptiveCard",
  "version": "1.4",
  "\$schema": "http://adaptivecards.io/schemas/adaptive-card.json",
  "body": [
    {
      "type": "Container",
      "style": "attention",
      "items": [
        {
          "type": "TextBlock",
          "text": "🚨 Disk Space Warning: Tableau Server",
          "weight": "Bolder",
          "size": "Medium",
          "style": "heading"
        }
      ]
    },
    {
      "type": "FactSet",
      "facts": [
        { "title": "Instance ID:", "value": "${INSTANCE_ID}" },
        { "title": "Hostname:", "value": "${HOSTNAME}" },
        { "title": "IP Address:", "value": "${IP_ADDR}" },
        { "title": "Partition:", "value": "${DISK_PARTITION}" },
        { "title": "Current Usage:", "value": "${CURRENT_USAGE}%" },
        { "title": "Threshold:", "value": "${THRESHOLD}%" }
      ]
    },
    {
      "type": "TextBlock",
      "text": "Please log in and free up space immediately to avoid Tableau Server failure.",
      "weight": "Bolder",
      "wrap": true
    }
  ]
}
EOF
)

    curl -X POST -H "Content-Type: application/json" -d "$PAYLOAD" "$WEBHOOK_URL"
    echo "Alert sent successfully."
}

# Function to send scheduled status report
send_status_report() {
    echo "Generating and sending scheduled status report to Teams..."

    # Clean and escape RAM and Disk reports so they fit safely inside the JSON payload
    RAM_REPORT_ESC=$(free -h | sed 's/"/\\"/g' | awk '{printf "%s\\n", $0}')
    DISK_REPORT_ESC=$(df -h -x tmpfs -x devtmpfs -x squashfs | sed 's/"/\\"/g' | awk '{printf "%s\\n", $0}')

    # JSON Payload for Scheduled Status Report
    PAYLOAD=$(cat <<EOF
{
  "type": "AdaptiveCard",
  "version": "1.4",
  "\$schema": "http://adaptivecards.io/schemas/adaptive-card.json",
  "body": [
    {
      "type": "Container",
      "style": "good",
      "items": [
        {
          "type": "TextBlock",
          "text": "✅ Scheduled System Status Report",
          "weight": "Bolder",
          "size": "Medium",
          "style": "heading"
        }
      ]
    },
    {
      "type": "FactSet",
      "facts": [
        { "title": "Instance ID:", "value": "${INSTANCE_ID}" },
        { "title": "Hostname:", "value": "${HOSTNAME}" },
        { "title": "IP Address:", "value": "${IP_ADDR}" },
        { "title": "Disk Status:", "value": "${CURRENT_USAGE}% (Threshold: ${THRESHOLD}%)" }
      ]
    },
    {
      "type": "TextBlock",
      "text": "📊 **RAM Memory Status**",
      "weight": "Bolder"
    },
    {
      "type": "TextBlock",
      "text": "\`\`\`\\n${RAM_REPORT_ESC}\\n\`\`\`",
      "fontType": "Monospace",
      "wrap": true
    },
    {
      "type": "TextBlock",
      "text": "💾 **Disk Volumes Status**",
      "weight": "Bolder"
    },
    {
      "type": "TextBlock",
      "text": "\`\`\`\\n${DISK_REPORT_ESC}\\n\`\`\`",
      "fontType": "Monospace",
      "wrap": true
    }
  ]
}
EOF
)

    curl -X POST -H "Content-Type: application/json" -d "$PAYLOAD" "$WEBHOOK_URL"
    echo "Status report sent successfully."
}

# ==============================================================================
# Main Execution Logic
# ==============================================================================

if [ "$1" == "--report" ]; then
    # Mode 1: Send status report directly to Teams (triggered at 9:00 AM and 4:00 PM)
    send_status_report
else
    # Mode 2: Standard monitoring (triggered every 10 minutes)
    if [ "$CURRENT_USAGE" -gt "$THRESHOLD" ]; then
        # Threshold exceeded -> Send emergency alert
        send_emergency_alert
    else
        # Under threshold -> Print status report to local stdout only (terminal verification)
        echo "=========================================================="
        echo "               SYSTEM STATUS REPORT (OK)                  "
        echo "=========================================================="
        echo "Instance ID : $INSTANCE_ID"
        echo "Hostname    : $HOSTNAME"
        echo "IP Address  : $IP_ADDR"
        echo "Disk Status : $CURRENT_USAGE% (Threshold is $THRESHOLD%)"
        echo "----------------------------------------------------------"
        echo "RAM MEMORY STATUS:"
        free -h
        echo "----------------------------------------------------------"
        echo "ALL DISK VOLUMES STATUS:"
        df -h
        echo "=========================================================="
    fi
fi
```

---

## Active Cron Schedule

The active schedules running on the system (verified via `sudo crontab -l`):

```cron
# 1. Check disk space every 10 minutes (Alerts immediately if disk > 75%)
*/10 * * * * /bin/bash /usr/local/bin/disk_monitor.sh > /dev/null 2>&1

# 2. Send healthy status report twice a day (At 9:00 AM and 4:00 PM)
0 9,16 * * * /bin/bash /usr/local/bin/disk_monitor.sh --report > /dev/null 2>&1
```
