# 20 Real-Time Shell Scripts (80-20 Rule for Practical Scripting)

This reference contains **20 practical, real-time shell scripts** that cover the essential **80% of real-world shell scripting tasks** in DevOps, system administration, and file automation. Each script is production-ready with detailed explanations for beginners.

---

## 1. Monitor System Resources (CPU, Memory, Disk)

```bash
#!/bin/bash

# Script to monitor system resources and log to a file
LOG_FILE="/var/log/system_monitor.log"
THRESHOLD=80

# Get CPU usage
cpu_usage=$(top -bn1 | grep "Cpu(s)" | awk '{print int($2)}')

# Get Memory usage
mem_usage=$(free | grep Mem | awk '{printf("%.0f", $3/$2 * 100)}')

# Get Disk usage
disk_usage=$(df -h / | tail -1 | awk '{print int($5)}')

# Create log entry
timestamp=$(date "+%Y-%m-%d %H:%M:%S")
echo "$timestamp - CPU: $cpu_usage%, Memory: $mem_usage%, Disk: $disk_usage%" >> "$LOG_FILE"

# Alert if threshold exceeded
if [ $cpu_usage -gt $THRESHOLD ] || [ $mem_usage -gt $THRESHOLD ] || [ $disk_usage -gt $THRESHOLD ]; then
    echo "ALERT: Resource usage exceeds threshold at $timestamp" >> "$LOG_FILE"
fi
```
**Explanation**: This script monitors three key system resources using `top`, `free`, and `df` commands. It logs the data with timestamps and alerts when resources exceed 80%. Perfect for ops teams to track server health automatically.

---

## 2. Find and Delete Old Log Files (Log Rotation)

```bash
#!/bin/bash

# Script to find and delete log files older than 7 days
LOG_DIR="/var/log"
DAYS=7

find "$LOG_DIR" -type f -name "*.log" -mtime +$DAYS -exec rm {} \;

echo "Log rotation completed. Files older than $DAYS days deleted."
```
**Explanation**: The `find` command searches for files matching a pattern, and `-mtime +7` finds files modified more than 7 days ago. The `-exec rm` deletes them. This prevents disk space from filling up with old logs.

---

## 3. Backup Files to Remote Server (Secure Copy)

```bash
#!/bin/bash

# Script to backup files to a remote server using SSH
SOURCE_DIR="/home/user/documents"
REMOTE_USER="admin"
REMOTE_HOST="192.168.1.100"
REMOTE_BACKUP_DIR="/backups"

# Create timestamp for unique backup folder
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")

# Copy files to remote server
scp -r "$SOURCE_DIR" "$REMOTE_USER@$REMOTE_HOST:$REMOTE_BACKUP_DIR/backup_$TIMESTAMP"

echo "Backup completed: backup_$TIMESTAMP"
```
**Explanation**: `scp` (secure copy) transfers files over SSH. The script recursively copies a directory to a remote server with a timestamped folder name for version tracking.

---

## 4. Check URL Status and Send Alert Email

```bash
#!/bin/bash

# Script to check if websites are up
URLS=("https://google.com" "https://github.com" "https://stackoverflow.com")
EMAIL="admin@example.com"

for url in "${URLS[@]}"; do
    status=$(curl -s -o /dev/null -w "%{http_code}" "$url")
    
    if [ "$status" != "200" ]; then
        echo "Alert: $url returned status $status" | mail -s "Website Down Alert" "$EMAIL"
    fi
done
```
**Explanation**: `curl` checks HTTP status codes of URLs. If any returns a non-200 status (not OK), an email alert is sent. This automates uptime monitoring.

---

## 5. Create User Accounts with SSH Access

```bash
#!/bin/bash

# Script to create new user and setup SSH access
USERNAME=$1
if [ -z "$USERNAME" ]; then
    echo "Usage: $0 <username>"
    exit 1
fi

# Create user
useradd -m -s /bin/bash "$USERNAME"

# Create .ssh directory
mkdir -p /home/"$USERNAME"/.ssh

# Set permissions
chmod 700 /home/"$USERNAME"/.ssh
chown "$USERNAME":"$USERNAME" /home/"$USERNAME"/.ssh

echo "User $USERNAME created successfully with SSH access configured."
```
**Explanation**: This automates user account creation. `useradd` creates the user, and the script sets up SSH directory with proper permissions (700 means owner can read/write/execute only).

---

## 6. Parse Log File and Extract Specific Patterns

```bash
#!/bin/bash

# Script to parse log file and count error occurrences
LOG_FILE="/var/log/application.log"
OUTPUT_FILE="error_report.txt"

# Extract lines containing "ERROR" and get unique errors
grep "ERROR" "$LOG_FILE" | sort | uniq -c | sort -rn > "$OUTPUT_FILE"

echo "Error report generated: $OUTPUT_FILE"
```
**Explanation**: `grep` searches for "ERROR" lines, `sort` orders them, `uniq -c` counts occurrences, and `-rn` sorts by frequency. This quickly identifies most common errors.

---

## 7. Monitor Failed Login Attempts

```bash
#!/bin/bash

# Script to extract failed login attempts from authentication log
AUTH_LOG="/var/log/auth.log"
OUTPUT_FILE="failed_logins.txt"

# Extract failed logins and count by IP
grep "Failed password" "$AUTH_LOG" | grep -oP '\d+\.\d+\.\d+\.\d+' | sort | uniq -c | sort -rn > "$OUTPUT_FILE"

echo "Failed login report saved: $OUTPUT_FILE"
```
**Explanation**: Parses auth logs for "Failed password" entries and extracts IP addresses using `grep -oP` (pattern matching). Counts and sorts by frequency to identify potential brute-force attacks.

---

## 8. Compress Old Files and Archive

```bash
#!/bin/bash

# Script to compress files older than 30 days
SOURCE_DIR="/var/data"
ARCHIVE_DIR="/var/archive"
DAYS=30

# Find old files and compress
find "$SOURCE_DIR" -type f -mtime +$DAYS -exec gzip {} \;

# Move compressed files to archive
mv "$SOURCE_DIR"/*.gz "$ARCHIVE_DIR/"

echo "Old files compressed and archived."
```
**Explanation**: `find` locates old files, `gzip` compresses them, and `mv` moves them to an archive directory. Saves disk space while keeping data accessible.

---

## 9. Synchronize Directories Between Servers

```bash
#!/bin/bash

# Script to sync files between two servers
SOURCE_DIR="/var/www/app"
REMOTE_USER="deploy"
REMOTE_HOST="production.server.com"
DEST_DIR="/var/www/app"

# Use rsync for efficient sync
rsync -avz -e ssh "$SOURCE_DIR/" "$REMOTE_USER@$REMOTE_HOST:$DEST_DIR/"

echo "Directory sync completed."
```
**Explanation**: `rsync` only transfers changed files (efficient), `-avz` enables archive mode with verbose output and compression. Much faster than `scp` for large directories.

---

## 10. Automatically Deploy Latest Code

```bash
#!/bin/bash

# Script to pull latest code and restart service
REPO_DIR="/opt/myapp"
SERVICE_NAME="myapp"

cd "$REPO_DIR" || exit 1

# Pull latest changes
git pull origin main

# Build/run tests
npm install && npm test

# Restart service if tests pass
if [ $? -eq 0 ]; then
    systemctl restart "$SERVICE_NAME"
    echo "Deployment successful. Service restarted."
else
    echo "Deployment failed. Tests did not pass."
    exit 1
fi
```
**Explanation**: Pulls latest code with `git pull`, runs tests, and only restarts the service if tests pass. `$?` captures the exit status (0 = success).

---

## 11. Find Largest Files and Generate Report

```bash
#!/bin/bash

# Script to find 10 largest files in filesystem
OUTPUT_FILE="largest_files_report.txt"

echo "Top 10 Largest Files" > "$OUTPUT_FILE"
echo "===================" >> "$OUTPUT_FILE"

find / -type f -printf '%s %p\n' 2>/dev/null | sort -rn | head -10 | awk '{printf("%.2f MB: %s\n", $1/1048576, $2)}' >> "$OUTPUT_FILE"

echo "Report saved: $OUTPUT_FILE"
```
**Explanation**: `find` locates all files with size and path, `sort -rn` orders by size (largest first), `head -10` gets top 10, and `awk` converts bytes to MB.

---

## 12. Kill Zombie Processes

```bash
#!/bin/bash

# Script to find and report zombie processes
ZOMBIE_PIDS=$(ps aux | grep "Z" | grep -v grep | awk '{print $3}')

if [ -z "$ZOMBIE_PIDS" ]; then
    echo "No zombie processes found."
else
    echo "Zombie processes (parent PIDs): $ZOMBIE_PIDS"
    # Kill parent processes to clean up zombies
    kill -9 $ZOMBIE_PIDS 2>/dev/null
    echo "Zombie cleanup attempted."
fi
```
**Explanation**: Uses `ps aux` to list processes, `grep "Z"` finds those in zombie state, and `awk` extracts parent PIDs. Killing parents cleans up their zombie children.

---

## 13. Read File Line by Line and Process

```bash
#!/bin/bash

# Script to read a file line by line
INPUT_FILE="servers.txt"
OUTPUT_FILE="server_status.txt"

# Clear output file
> "$OUTPUT_FILE"

while IFS= read -r line; do
    # Skip empty lines
    [ -z "$line" ] && continue
    
    # Ping server and log result
    if ping -c 1 -W 1 "$line" &> /dev/null; then
        echo "$line - UP" >> "$OUTPUT_FILE"
    else
        echo "$line - DOWN" >> "$OUTPUT_FILE"
    fi
done < "$INPUT_FILE"

echo "Server status check completed."
```
**Explanation**: `while read` loops through each line of input file. `IFS=` preserves spaces. Script pings each server and logs status. `ping -W 1` sets timeout to 1 second.

---

## 14. Extract and Count Specific Data from Logs

```bash
#!/bin/bash

# Script to extract HTTP status codes and count occurrences
LOG_FILE="/var/log/apache2/access.log"
OUTPUT_FILE="http_status_report.txt"

# Extract status codes and count
awk '{print $9}' "$LOG_FILE" | sort | uniq -c | sort -rn > "$OUTPUT_FILE"

echo "HTTP Status Report:"
cat "$OUTPUT_FILE"
```
**Explanation**: `awk '{print $9}'` extracts the 9th field (HTTP status code) from Apache logs. `sort | uniq -c` counts occurrences. Reports most common status codes.

---

## 15. Automated Database Backup

```bash
#!/bin/bash

# Script to backup MySQL database
DB_USER="root"
DB_PASS="password"
DB_NAME="myapp_db"
BACKUP_DIR="/backups/database"

# Create backup with timestamp
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
BACKUP_FILE="$BACKUP_DIR/${DB_NAME}_${TIMESTAMP}.sql"

mysqldump -u "$DB_USER" -p"$DB_PASS" "$DB_NAME" > "$BACKUP_FILE"

# Compress backup
gzip "$BACKUP_FILE"

echo "Database backup completed: ${BACKUP_FILE}.gz"
```
**Explanation**: `mysqldump` exports database to SQL file. Timestamp ensures unique backup names. `gzip` compresses for storage efficiency. Essential for disaster recovery.

---

## 16. Compare Two Files and Report Differences

```bash
#!/bin/bash

# Script to compare two configuration files
FILE1=$1
FILE2=$2

if [ -z "$FILE1" ] || [ -z "$FILE2" ]; then
    echo "Usage: $0 <file1> <file2>"
    exit 1
fi

echo "Differences between $FILE1 and $FILE2:"
diff -u "$FILE1" "$FILE2"
```
**Explanation**: `diff -u` shows unified diff format (easy to read). Useful for tracking configuration changes or comparing log outputs.

---

## 17. Automate System Updates

```bash
#!/bin/bash

# Script to update system and handle errors
set -e  # Exit on any error

echo "Starting system update..."

# Update package lists
apt-get update

# Upgrade packages
apt-get upgrade -y

# Auto-remove unused packages
apt-get autoremove -y

echo "System update completed successfully."
```
**Explanation**: `set -e` ensures script exits if any command fails. `apt-get update` refreshes package lists, `upgrade -y` automatically approves updates, `autoremove` cleans up.

---

## 18. Create Audit Trail of File Changes

```bash
#!/bin/bash

# Script to track which files were modified in a directory
TARGET_DIR="/opt/myapp"
AUDIT_LOG="/var/log/file_audit.log"

# Find files modified in last 24 hours
find "$TARGET_DIR" -type f -mtime -1 -printf '%T@ %Tc %p\n' | sort -n | while read timestamp datetime file; do
    echo "$datetime - Modified: $file" >> "$AUDIT_LOG"
done

echo "Audit trail updated."
```
**Explanation**: `-mtime -1` finds files modified within last day. `printf` formats output with timestamps. Useful for security and compliance tracking.

---

## 19. Send System Report via Email

```bash
#!/bin/bash

# Script to generate and email system report
REPORT_FILE="/tmp/system_report.txt"
EMAIL="admin@example.com"

# Generate report
{
    echo "System Report - $(date)"
    echo "========================"
    echo "Uptime:"
    uptime
    echo ""
    echo "Disk Usage:"
    df -h
    echo ""
    echo "Memory Usage:"
    free -h
} > "$REPORT_FILE"

# Send email
mail -s "Daily System Report" "$EMAIL" < "$REPORT_FILE"

echo "Report sent to $EMAIL"
```
**Explanation**: Generates a comprehensive system report using `uptime`, `df`, and `free` commands. `mail` sends the report as email body. Good for daily monitoring summaries.

---

## 20. Parallel Processing with Background Jobs

```bash
#!/bin/bash

# Script to process multiple files in parallel
INPUT_DIR="/data/files"
OUTPUT_DIR="/data/processed"
MAX_JOBS=4

count=0
for file in "$INPUT_DIR"/*.txt; do
    # Process file in background
    (
        echo "Processing: $file"
        # Your processing command here
        cat "$file" | tr ' ' '\n' | sort | uniq > "$OUTPUT_DIR/$(basename $file)"
    ) &
    
    # Limit concurrent jobs
    count=$((count + 1))
    if [ $count -ge $MAX_JOBS ]; then
        wait -n  # Wait for one job to complete
        count=$((count - 1))
    fi
done

# Wait for remaining jobs
wait

echo "All files processed."
```
**Explanation**: Processes multiple files concurrently using background jobs (`&`). `MAX_JOBS` limits parallelism to prevent resource exhaustion. `wait -n` waits for one job to complete before starting another.

---

## Summary

These **20 scripts represent 80% of practical shell scripting** used in real-world DevOps, system administration, and automation tasks. They cover:

- **System Monitoring**: CPU, memory, disk tracking
- **File Management**: Backup, compression, synchronization
- **Log Processing**: Parsing, extracting, analyzing logs
- **Automation**: Deployments, user management, updates
- **Alerts & Reports**: Email notifications, status checks
- **Security**: Monitoring logins, audit trails

Master these scripts to handle most operational challenges and ace DevOps interviews!

---