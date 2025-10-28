# Linux Commands Cheatsheet for Junior Operations Engineers

**Master these commands to crack your Toyota Operations Engineer interview!**

---

## Table of Contents

1. [Basic Navigation & File Operations](#basic-navigation--file-operations)
2. [System Information](#system-information)
3. [Process Management](#process-management)
4. [Log Viewing & Analysis](#log-viewing--analysis)
5. [System Monitoring](#system-monitoring)
6. [Network Commands](#network-commands)
7. [Service Management](#service-management)
8. [Disk & Storage Management](#disk--storage-management)
9. [File Permissions & Ownership](#file-permissions--ownership)
10. [Text Processing & Search](#text-processing--search)
11. [Compression & Archives](#compression--archives)
12. [User Management](#user-management)
13. [Environment & Variables](#environment--variables)
14. [Cron & Scheduling](#cron--scheduling)
15. [Advanced Troubleshooting](#advanced-troubleshooting)

---

## Basic Navigation & File Operations

### Directory Navigation
```bash
# Show current directory
pwd
# Example output: /home/user

# List files and directories
ls                    # Basic listing
ls -l                 # Long format with details
ls -la                # Include hidden files
ls -ltr               # Sort by time (oldest first)
ls -lh                # Human-readable file sizes
ls -R                 # Recursive listing

# Change directory
cd /path/to/directory # Absolute path
cd ../                # Go up one level
cd ~                  # Go to home directory
cd -                  # Go to previous directory
```

### File Operations
```bash
# Create files and directories
touch filename.txt           # Create empty file
mkdir directory_name         # Create directory
mkdir -p parent/child/dirs   # Create nested directories

# Copy files
cp source.txt destination.txt        # Copy file
cp -r source_dir destination_dir     # Copy directory recursively
cp -p file.txt backup.txt           # Preserve permissions and timestamps

# Move/rename files
mv old_name.txt new_name.txt    # Rename file
mv file.txt /new/location/      # Move file
mv *.log /logs/                 # Move all .log files

# Remove files
rm filename.txt          # Remove file
rm -r directory_name     # Remove directory and contents
rm -f locked_file.txt    # Force remove
rm -rf dangerous_dir/    # Force remove directory (BE CAREFUL!)

# View file contents
cat file.txt            # Display entire file
head file.txt           # First 10 lines
head -n 20 file.txt     # First 20 lines
tail file.txt           # Last 10 lines
tail -n 50 file.txt     # Last 50 lines
tail -f logfile.log     # Follow file (real-time updates)
less file.txt           # Page through file (q to quit)
more file.txt           # Page through file
```

**Interview Tip:** Explain the difference between `cp` and `mv` - copy creates a duplicate, move transfers the file.

---

## System Information

### Basic System Info
```bash
# System information
uname -a              # All system info
uname -r              # Kernel version
uname -m              # Architecture
hostname              # System hostname
uptime                # System uptime and load
date                  # Current date and time
cal                   # Calendar
whoami                # Current username
id                    # User and group IDs
who                   # Logged in users
w                     # Detailed user activity
```

### Hardware Information
```bash
# Hardware details
lscpu                 # CPU information
lsmem                 # Memory information
lsblk                 # Block devices (disks)
lsusb                 # USB devices
lspci                 # PCI devices
lshw                  # Comprehensive hardware info
dmidecode             # Hardware details from BIOS
```

**Interview Example:** "To check if a server has enough CPU cores for our application, I'd run `lscpu` to see the number of cores and `uptime` to check current load."

---

## Process Management

### Viewing Processes
```bash
# Process information
ps                    # Processes in current terminal
ps aux                # All processes with detailed info
ps aux | grep nginx   # Find specific process
ps -ef                # All processes in full format
ps -u username        # Processes by specific user
pstree                # Process tree hierarchy
```

### Process Control
```bash
# Managing processes
kill PID              # Terminate process by PID
kill -9 PID           # Force kill process
killall process_name  # Kill all processes by name
pkill pattern         # Kill processes matching pattern
nohup command &       # Run command in background
jobs                  # Show background jobs
fg %1                 # Bring job to foreground
bg %1                 # Send job to background

# Finding process IDs
pidof process_name    # Get PID of process
pgrep process_name    # Get PID using pattern
```

**Real-world Example:**
```bash
# Find and restart a hung nginx process
ps aux | grep nginx
# nginx     1234  0.0  0.1  12345  6789 ?  S  10:30  0:00 nginx: master
kill -9 1234
systemctl restart nginx
```

---

## Log Viewing & Analysis

### System Logs
```bash
# System log files
tail -f /var/log/syslog           # System messages
tail -f /var/log/auth.log         # Authentication logs
tail -f /var/log/kern.log         # Kernel messages
tail -f /var/log/cron.log         # Cron job logs
tail -f /var/log/apache2/error.log # Web server errors
```

### Journal Control (systemd)
```bash
# SystemD journal
journalctl                        # All journal entries
journalctl -f                     # Follow journal (like tail -f)
journalctl -u service_name        # Logs for specific service
journalctl -u nginx.service       # Nginx service logs
journalctl --since "1 hour ago"   # Logs from last hour
journalctl --since yesterday      # Yesterday's logs
journalctl -p err                 # Only error messages
journalctl --disk-usage          # Journal disk usage
```

### Log Analysis Commands
```bash
# Analyzing logs
grep "ERROR" /var/log/syslog      # Find error messages
grep -i "failed" /var/log/auth.log # Case-insensitive search
grep -v "INFO" app.log            # Exclude INFO messages
grep -C 5 "ERROR" app.log         # Show 5 lines context
grep -r "pattern" /var/log/       # Recursive search
```

**Interview Scenario:** "How would you troubleshoot a service that's not starting?"
```bash
# Check service status
systemctl status service_name
# Check recent logs
journalctl -u service_name --since "10 minutes ago"
# Check system logs for errors
tail -n 100 /var/log/syslog | grep -i error
```

---

## System Monitoring

### Real-time Monitoring
```bash
# Process monitoring
top                   # Real-time process viewer
htop                  # Enhanced top (if installed)
# Top shortcuts: 'q' quit, 'k' kill, 'M' sort by memory, 'P' sort by CPU

# System resources
watch -n 2 df -h      # Monitor disk space every 2 seconds
watch -n 1 free -h    # Monitor memory every second
```

### Memory Monitoring
```bash
# Memory information
free -h               # Memory usage (human readable)
free -m               # Memory usage in MB
vmstat 1 5           # Virtual memory stats (1 sec interval, 5 times)
vmstat -s            # Memory statistics summary
```

### Disk Monitoring
```bash
# Disk usage
df -h                # Disk space by filesystem
df -i                # Inode usage
du -h /path          # Directory size
du -sh /*            # Size of root directories
du -sh * | sort -hr  # Sorted by size (largest first)
```

### Performance Monitoring
```bash
# Performance tools
iostat 1 5           # I/O statistics
sar 1 5              # System activity report
mpstat 1 5           # Multi-processor statistics
pidstat 1 5          # Per-process statistics
iotop                # I/O monitoring by process
```

**Interview Question:** "Server is running slow, how do you investigate?"
```bash
# Step-by-step investigation
uptime               # Check load average
top                  # Check CPU and memory usage
df -h                # Check disk space
iostat 1 3           # Check I/O wait
```

---

## Network Commands

### Network Configuration
```bash
# Network interfaces
ip addr show         # Show all interfaces and IPs
ip addr show eth0    # Show specific interface
ifconfig            # Legacy interface configuration
ip route            # Show routing table
ip route show       # Same as above
route -n            # Legacy routing table
```

### Network Connectivity
```bash
# Connection testing
ping google.com      # Test connectivity
ping -c 4 8.8.8.8   # Ping 4 times only
traceroute google.com # Trace packet route
tracepath google.com  # Alternative to traceroute
mtr google.com       # Continuous traceroute
```

### DNS Resolution
```bash
# DNS tools
nslookup google.com  # DNS lookup
dig google.com       # Detailed DNS info
dig @8.8.8.8 google.com # Use specific DNS server
host google.com      # Simple DNS lookup
```

### Network Connections
```bash
# Active connections
netstat -tuln        # Show listening ports (TCP/UDP)
netstat -tulnp       # Include process names
ss -tuln            # Modern replacement for netstat
ss -tulnp           # With process info
ss -t state established # Show established TCP connections
lsof -i :80         # Show what's using port 80
lsof -i :22         # Show SSH connections
```

### Network Troubleshooting
```bash
# Advanced network tools
tcpdump -i eth0      # Packet capture on interface
tcpdump -i any port 80 # Capture HTTP traffic
arp -a               # Show ARP table
arp -d IP_ADDRESS    # Delete ARP entry
```

**Troubleshooting Scenario:** "Web service is not accessible"
```bash
# Step-by-step network debugging
ping web_server_ip           # Test basic connectivity  
telnet web_server_ip 80      # Test port connectivity
nslookup web_server_name     # Check DNS resolution
ss -tuln | grep :80          # Check if service is listening
curl -I http://web_server    # Test HTTP response
```

---

## Service Management

### SystemD Services
```bash
# Service control
systemctl status service_name     # Check service status
systemctl start service_name      # Start service
systemctl stop service_name       # Stop service
systemctl restart service_name    # Restart service
systemctl reload service_name     # Reload configuration
systemctl enable service_name     # Enable auto-start at boot
systemctl disable service_name    # Disable auto-start
systemctl list-units --type=service # List all services
systemctl list-units --failed     # Show failed services
```

### Service Information
```bash
# Service details
systemctl show service_name       # Show service properties
systemctl cat service_name        # Show service file content
systemctl list-dependencies service_name # Show dependencies
```

**Production Example:**
```bash
# Restart nginx safely
systemctl status nginx           # Check current status
nginx -t                        # Test configuration
systemctl reload nginx          # Reload without downtime
systemctl status nginx          # Verify it's running
```

---

## Disk & Storage Management

### Disk Usage
```bash
# Disk space analysis
df -h                # Human-readable disk usage
df -T                # Show filesystem type
du -h --max-depth=1 /home # Directory sizes one level deep
du -sh /var/log/*    # Size of log files
find /var -size +100M # Files larger than 100MB
```

### Disk Operations
```bash
# Disk management
fdisk -l             # List all disks
lsblk               # Block device tree
mount               # Show mounted filesystems
mount /dev/sdb1 /mnt # Mount device
umount /mnt         # Unmount
```

### File System Operations
```bash
# File system tools
fsck /dev/sdb1      # Check filesystem
tune2fs -l /dev/sdb1 # Show filesystem info
```

**Disk Space Emergency:**
```bash
# When disk is 95% full
df -h               # Identify full filesystem
du -sh /* | sort -hr | head -10  # Find largest directories
find /var/log -name "*.log" -mtime +30 # Old log files
find /tmp -mtime +7 -delete # Clean old temp files
```

---

## File Permissions & Ownership

### Permission Basics
```bash
# View permissions
ls -l file.txt      # Show file permissions
stat file.txt       # Detailed file info
```

### Changing Permissions
```bash
# Permission modification
chmod 755 file.txt           # rwxr-xr-x
chmod u+x file.txt           # Add execute for user
chmod g-w file.txt           # Remove write for group
chmod o=r file.txt           # Set only read for others
chmod +x script.sh           # Make script executable
```

### Changing Ownership
```bash
# Ownership modification
chown user:group file.txt    # Change user and group
chown user file.txt          # Change user only
chgrp group file.txt         # Change group only
chown -R user:group /dir     # Recursive ownership change
```

**Permission Chart:**
```
rwx = 4+2+1 = 7 (read, write, execute)
rw- = 4+2+0 = 6 (read, write)
r-x = 4+0+1 = 5 (read, execute)
r-- = 4+0+0 = 4 (read only)
-wx = 0+2+1 = 3 (write, execute)
-w- = 0+2+0 = 2 (write only)
--x = 0+0+1 = 1 (execute only)
--- = 0+0+0 = 0 (no permission)
```

---

## Text Processing & Search

### Text Search
```bash
# Pattern searching
grep "pattern" file.txt      # Search for pattern
grep -i "pattern" file.txt   # Case-insensitive
grep -r "pattern" /dir       # Recursive search
grep -n "pattern" file.txt   # Show line numbers
grep -v "pattern" file.txt   # Invert match (exclude)
grep -c "pattern" file.txt   # Count matches
grep -l "pattern" *.txt      # Show only filenames
```

### Advanced Text Processing
```bash
# Text manipulation
awk '{print $1}' file.txt    # Print first column
sed 's/old/new/g' file.txt   # Replace all occurrences
cut -d: -f1 /etc/passwd      # Extract first field (delimiter :)
sort file.txt                # Sort lines
sort -r file.txt             # Reverse sort
sort -n file.txt             # Numeric sort
uniq file.txt                # Remove duplicate lines
wc -l file.txt              # Count lines
wc -w file.txt              # Count words
```

### File Finding
```bash
# File search
find /path -name "*.log"     # Find files by name
find /path -type f -size +10M # Files larger than 10MB
find /path -mtime -1         # Modified in last 24 hours
find /path -user username    # Files owned by user
locate filename              # Quick file search (requires updatedb)
which command                # Find command location
whereis command              # Find command and manual pages
```

**Log Analysis Example:**
```bash
# Analyze web server logs
grep "ERROR" /var/log/apache2/error.log | tail -20
grep "404" /var/log/apache2/access.log | awk '{print $1}' | sort | uniq -c | sort -nr
```

---

## Compression & Archives

### Creating Archives
```bash
# TAR operations
tar -czf archive.tar.gz /path/to/dir    # Create compressed archive
tar -czf backup-$(date +%Y%m%d).tar.gz /home/user # Timestamped backup
tar -xzf archive.tar.gz                 # Extract compressed archive
tar -tzf archive.tar.gz                 # List archive contents
tar -xzf archive.tar.gz -C /target/dir  # Extract to specific directory
```

### Compression Tools
```bash
# Compression utilities
gzip file.txt           # Compress file (creates file.txt.gz)
gunzip file.txt.gz      # Decompress file
zip -r archive.zip /dir # Create ZIP archive
unzip archive.zip       # Extract ZIP archive
```

**Backup Script Example:**
```bash
#!/bin/bash
# Create timestamped backup
BACKUP_DIR="/backups"
DATE=$(date +%Y%m%d-%H%M%S)
tar -czf $BACKUP_DIR/app-backup-$DATE.tar.gz /opt/myapp
echo "Backup created: app-backup-$DATE.tar.gz"
```

---

## User Management

### User Information
```bash
# User details
whoami              # Current user
id                  # User and group IDs
groups              # User's groups
finger username     # User information
last                # Login history
lastlog             # Last login for all users
```

### User Operations (requires sudo)
```bash
# User management
sudo useradd username           # Add user
sudo usermod -aG group username # Add user to group
sudo passwd username            # Change password
sudo userdel username           # Delete user
sudo groupadd groupname         # Add group
sudo groupdel groupname         # Delete group
```

---

## Environment & Variables

### Environment Variables
```bash
# Environment management
env                 # Show all environment variables  
echo $PATH          # Show PATH variable
echo $HOME          # Show home directory
export VAR=value    # Set environment variable
unset VAR           # Remove variable
printenv            # Print environment variables
```

### Shell Configuration
```bash
# Configuration files
~/.bashrc           # User bash configuration
~/.bash_profile     # User login configuration
/etc/profile        # System-wide profile
source ~/.bashrc    # Reload configuration
```

---

## Cron & Scheduling

### Cron Management
```bash
# Cron jobs
crontab -l          # List user's cron jobs
crontab -e          # Edit user's cron jobs
crontab -r          # Remove all cron jobs
sudo crontab -l -u username # List another user's crons
```

### Cron Format
```
# Minute Hour Day Month DayOfWeek Command
# *      *    *   *     *         /path/to/script
  0      2    *   *     *         /backup/script.sh    # Daily at 2 AM
  */15   *    *   *     *         /monitor/script.sh   # Every 15 minutes
  0      0    1   *     *         /monthly/cleanup.sh  # Monthly
```

### SystemD Timers
```bash
# Modern scheduling
systemctl list-timers           # Show active timers
systemctl status timer_name     # Check timer status
```

---

## Advanced Troubleshooting

### System Analysis
```bash
# Kernel and hardware messages
dmesg                    # Kernel ring buffer
dmesg | tail -20         # Recent kernel messages
dmesg | grep -i error    # Kernel errors
```

### Process Analysis
```bash
# Detailed process information
lsof                     # List open files
lsof -p PID             # Files opened by specific process
lsof -i :port           # Processes using specific port
strace -p PID           # Trace system calls of process
ltrace -p PID           # Trace library calls of process
```

### Network Analysis
```bash
# Network troubleshooting
netstat -i              # Interface statistics
iftop                   # Real-time network usage
nload                   # Network load monitor
```

### System Load Analysis
```bash
# Load investigation
uptime                  # Load averages
cat /proc/loadavg      # Detailed load info
cat /proc/cpuinfo      # CPU information
cat /proc/meminfo      # Memory information
```

---

## Interview-Specific Command Combinations

### Scenario 1: High Memory Usage
```bash
# Investigation steps
free -h                                    # Check memory usage
ps aux --sort=-%mem | head -10            # Top memory consumers
top -o %MEM                               # Real-time memory monitoring
```

### Scenario 2: High Disk Usage
```bash
# Quick disk analysis
df -h                                     # Check disk space
du -sh /* | sort -hr | head -10           # Largest directories
find /var/log -name "*.log" -size +100M   # Large log files
```

### Scenario 3: Service Not Responding
```bash
# Service troubleshooting
systemctl status service_name             # Check service status
journalctl -u service_name --since "1 hour ago" # Recent logs
ss -tuln | grep :port_number              # Check if port is listening
```

### Scenario 4: Network Connectivity Issues
```bash
# Network debugging
ping -c 3 target_host                     # Test connectivity
traceroute target_host                    # Trace route
nslookup target_host                      # Check DNS
ss -tuln                                  # Check listening ports
```

---

## Command Chaining & Shortcuts

### Useful Command Combinations
```bash
# Pipe combinations
ps aux | grep nginx | grep -v grep       # Find nginx processes
df -h | grep -E "(8[0-9]|9[0-9])%"      # Disks over 80% full
netstat -tuln | grep :80 | wc -l        # Count connections on port 80
cat /var/log/syslog | grep ERROR | tail -5 # Last 5 errors

# Command chaining
cd /var/log && ls -la                     # Change dir AND list
mkdir -p backup/$(date +%Y%m%d) && cd $_ # Create dated dir and enter
```

### Keyboard Shortcuts
```bash
Ctrl+C          # Kill current process
Ctrl+Z          # Suspend current process  
Ctrl+D          # Exit shell/logout
Ctrl+L          # Clear screen (same as 'clear')
Ctrl+A          # Move to beginning of line
Ctrl+E          # Move to end of line
Ctrl+R          # Reverse history search
!!              # Repeat last command
!grep           # Repeat last command starting with 'grep'
```

---

## Quick Reference Cards

### File Permissions Quick Reference
```
rwx rwx rwx = 777 (read, write, execute for all)
rw- rw- r-- = 664 (read/write for user/group, read for others)
rw- r-- r-- = 644 (read/write for user, read for others)
rwx r-x r-x = 755 (full for user, read/execute for others)
```

### Common Log Locations
```
/var/log/syslog         # System messages
/var/log/auth.log       # Authentication
/var/log/kern.log       # Kernel messages  
/var/log/cron.log       # Cron jobs
/var/log/apache2/       # Apache web server
/var/log/nginx/         # Nginx web server
```

### Process Signals
```
SIGTERM (15)    # Graceful termination (default kill)
SIGKILL (9)     # Force termination (kill -9)
SIGHUP (1)      # Reload configuration
SIGSTOP (19)    # Pause process
SIGCONT (18)    # Resume process
```

---

## Interview Preparation Tips

### Commands to Master for Operations Role:
1. **Process Management:** `ps`, `top`, `kill`, `systemctl`
2. **Log Analysis:** `tail`, `grep`, `journalctl`, `less`
3. **Network:** `ping`, `ss`, `netstat`, `curl`
4. **Monitoring:** `df`, `free`, `uptime`, `iostat`
5. **Service Management:** `systemctl`, `service`

### Practice Scenarios:
1. "Server is running slow" → `top`, `free`, `df`, `iostat`
2. "Service won't start" → `systemctl status`, `journalctl`, log analysis
3. "Can't connect to server" → `ping`, `telnet`, `ss`, DNS checks
4. "Disk is full" → `df`, `du`, find large files, cleanup

### Key Points to Remember:
- Always explain your thought process when troubleshooting
- Show knowledge of multiple approaches to solve problems
- Demonstrate understanding of when to use each command
- Mention safety considerations (backups, testing, etc.)
- Show familiarity with both legacy and modern commands (`netstat` vs `ss`, `ifconfig` vs `ip`)

---

**Remember:** Practice these commands on a Linux system before your interview. Understanding the output and being able to explain what each command does is more important than memorizing syntax!