# Amazon ECS & DelayedJob PDF Queuing Incident
## Architecture, Post-Mortem, & Operational Playbook

> [!NOTE]
> **Document Purpose**: This comprehensive reference manual documents the system architecture, root cause analysis, code-level mechanics, pitfalls/wrong turns, production solution, and operational commands for managing background PDF generation in the `qreport` Rails application on AWS ECS.

---

## 🏛️ 1. Project & System Architecture Outline

```
                           ┌─────────────────────────────────────────┐
                           │               AWS ECS HOST              │
                           │              (EC2 Instance)             │
                           │                                         │
                           │  ┌───────────────────┐                  │
                           │  │   qreport-app     │ (Puma Web App)   │
                           │  └─────────┬─────────┘                  │
                           │            │ Inserts Jobs               │
                           │            ▼                            │
                           │  ┌───────────────────┐                  │
                           │  │   qreport-dj      │ (DelayedJob)     │
                           │  └─────────┬─────────┘                  │
                           └────────────┼────────────────────────────┘
                                        │
             ┌──────────────────────────┼──────────────────────────┐
             ▼                          ▼                          ▼
   ┌───────────────────┐      ┌───────────────────┐      ┌───────────────────┐
   │    Amazon RDS     │      │   Tableau Cloud   │      │     Amazon S3     │
   │   (MySQL DB)      │      │     REST API      │      │  (Storage Bucket) │
   └───────────────────┘      └───────────────────┘      └───────────────────┘
```

### Core System Components:
1. **`qreport-app` (Web Container)**: Rails application serving HTTP user requests. When a policy renewal or quote is requested, it inserts job records into the database queue.
2. **`qreport-dj` (DelayedJob Worker Container)**: Background ECS service running a Rails worker daemon via `./startup.sh` to execute queued asynchronous tasks.
3. **Database (`Amazon RDS - db.m5.xlarge`)**: Holds application data and manages the queue via the `delayed_jobs` table using row-level locking (`locked_at`, `locked_by`).
4. **Tableau Cloud REST API**: External analytics reporting platform. The Rails app issues HTTPS REST API requests to Tableau Cloud to render PDF dashboard pages.
5. **Amazon S3**: AWS object storage bucket where final, merged policy PDF documents are stored for user download.

---

## 🧠 2. Understanding DelayedJob

### The Core Terminology:
* **`delayed_job` (Software Library)**: The Ruby on Rails background task manager gem installed in the app.
* **`delayed_jobs` (Database Table)**: The actual SQL table in Amazon RDS where pending tasks wait in line.
* **`qreport-dj` (Worker Container)**: The ECS Docker service that runs the worker processes.

### Database Schema (`db/migrate/create_delayed_jobs.rb`):
```ruby
class CreateDelayedJobs < ActiveRecord::Migration
  def self.up
    create_table :delayed_jobs, :force => true do |table|
      table.integer  :priority, :default => 0      # Lower numbers run first
      table.integer  :attempts, :default => 0      # Incremented on retries before failing
      table.text     :handler                      # YAML-encoded Ruby object containing method to run
      table.text     :last_error                   # Stack trace/error message from last failure
      table.datetime :run_at                       # Scheduled execution time
      table.datetime :locked_at                    # Timestamp set when a worker picks up the job
      table.datetime :failed_at                    # Timestamp when all retries are exhausted
      table.string   :locked_by                    # Worker Host & PID string (e.g. host:59a83c249219 pid:45)
      table.string   :queue                        # Optional queue identifier
      table.timestamps
    end

    add_index :delayed_jobs, [:priority, :run_at], :name => 'delayed_jobs_priority'
  end
end
```

### Configuration (`config/initializers/delayed_job_config.rb`):
```ruby
# Maximum retries before a job is marked as failed
Delayed::Worker.max_attempts = 12
# Maximum execution time before a lock expires
Delayed::Worker.max_run_time = 4.hours
```

---

## 🔍 3. Code-Level Analysis (`app/models/policy.rb`)

### 1. The Delayed Job Trigger:
```ruby
pdf_type_lists[pdf_type].count == 1 ? 
  Policy.store_letters_in_diary(...) : 
  Policy.delay.store_letters_in_diary(...)
```
* **Single Policy (`count == 1`)**: Executes `store_letters_in_diary` synchronously on the web server.
* **Batch of Policies (`count > 1`)**: Calls **`Policy.delay.store_letters_in_diary(...)`**. The `.delay` method intercepts the call, converts arguments into YAML text, and inserts a row into the `delayed_jobs` table.

### 2. The Worker Execution Method:
```ruby
def self.store_letters_in_diary(policies, partial, filename, addr_file_id, pdf_type, batch_file_id, mode, admin_id, tokens)
  policies && policies.each do |policy_id|
    policy = Policy.find_by_id(policy_id)
    Policy.generate_pdf([policy], "php/reports/#{partial}", filename + "_Single", addr_file_id, pdf_type, batch_file_id, mode, admin_id, tokens)
    add_reminder(pdf_type, policy, policy.status_code) if policy.status_code != POL_STATUS::AUTO_RENEWAL_PENDING
  end
end
```
* The worker loop receives an array of `policy_id` values and iterates through them sequentially.
* For each policy ID, `Policy.generate_pdf` calls Tableau Cloud (takes ~30s per page), appends S3 pages, and writes the output to S3.

> [!IMPORTANT]
> **Batch Math Bottleneck**: A batch of 30 policies inside a single delayed job takes $30 \text{ policies} \times 55 \text{ seconds} = \mathbf{27.5 \text{ minutes}}$ to complete. If only 1 worker process is running, all other queued jobs are blocked behind it for nearly half an hour!

---

## 🚨 4. Root Cause Analysis (RCA)

1. **Heavy External I/O Latency**: Log analysis proved each PDF required 2 HTTPS calls to Tableau Cloud, taking **36.1s for Page 1** and **18.8s for Page 2** (~55s per PDF total).
2. **Single-Worker Bottleneck**: `startup.sh` originally executed:
   ```bash
   bundle exec script/delayed_job run
   ```
   This ran only **1 single worker process (PID 50)** in foreground mode. Max throughput was capped at **~60 PDFs per hour**.
3. **Host Idle CPU**: Host metrics (`top`) showed **97.3% Idle CPU**. The server wasn't overloaded—the worker was simply sleeping/waiting on Tableau HTTPS responses.

---

## ⚠️ 5. Wrong Turns & Pitfalls Avoided

> [!WARNING]
> **Pitfall 1: `-n 3 run` in `script/delayed_job` Ignores Concurrency Flags**
> Running `bundle exec script/delayed_job -n 3 run` does **NOT** spawn 3 workers. In `delayed_job`, `run` mode (foreground) ignores `-n` and only runs 1 worker. `-n` requires `start` mode or background process spawning.

> [!WARNING]
> **Pitfall 2: `source: not found` in Container Shell**
> Standard POSIX shell (`/bin/sh`) inside Linux containers does not support the `source` keyword. Use the dot operator instead:
> `. /tmp/secrets_env.sh`

> [!WARNING]
> **Pitfall 3: Defining IRB-only `Struct` Jobs Fails in Workers**
> Defining `TestJob = Struct.new(...)` inside an interactive Rails console session creates a class in IRB memory only. When external background workers try to deserialize it from DB, they fail with `uninitialized constant TestJob`. To test workers, use built-in methods like `3.times { Kernel.delay.sleep(5) }`.

> [!WARNING]
> **Pitfall 4: False Alarm on Database Connection Pool Limits**
> A concern was raised that running 3 workers with `pool: 5` in `database.yml` would crash RDS connections ($3 \times 5 = 15$ connections). 
> **Fact**: The RDS instance is a `db.m5.xlarge` (16 GB RAM) with a `max_connections` limit of **~1,365 connections**. 15–20 connections is **~1.5% of total capacity** and 100% safe.

---

## 🛠️ 6. The Production Solution (`startup.sh`)

To run 3 true parallel worker processes in Docker with proper signal handling and container persistence:

```bash
#!/bin/bash

# Fetch secret JSON
SECRET_JSON=$(aws secretsmanager get-secret-value \
  --secret-id "$SECRET_ID" \
  --query SecretString \
  --output text)

if [ $? -ne 0 ]; then
  echo "Error fetching secret: $SECRET_ID"
  exit 1
fi

# Convert keys with invalid chars into valid shell variable names
echo "$SECRET_JSON" | jq -r '
  to_entries | .[] |
  .key |= gsub("[^A-Za-z0-9_]"; "_") |
  "export \(.key)=\(.value)"
' > /tmp/secrets_env.sh

# Load ENV variables (. works in /bin/sh and /bin/bash)
. /tmp/secrets_env.sh

echo "Secrets loaded into environment variables (invalid characters in keys replaced with _)."

if [ "$CONTAINER_NAME" == "qreport-app" ]; then
  echo "🚀 Starting Rails app server..."
  cron start;bundle exec rake db:migrate;bundle exec whenever --set environment=$ENV;bundle exec whenever --update-crontab;bundle exec puma --config config/puma.rb
elif [ "$CONTAINER_NAME" == "qreport-dj" ]; then
  echo "⏳ Starting 3 Parallel Delayed Job workers..."
  trap 'kill -TERM $(jobs -p)' TERM INT
  bundle exec script/delayed_job run &
  bundle exec script/delayed_job run &
  bundle exec script/delayed_job run &
  wait -n
else
  echo "❌ Invalid container type: $CONTAINER_TYPE (use app|delayedjob)"
  exit 1
fi
```

### Why this pattern works:
* `bundle exec script/delayed_job run &`: Backgrounds 3 independent foreground worker processes in Linux memory.
* `trap 'kill -TERM $(jobs -p)' TERM INT`: Forwards ECS termination signals to all 3 workers so jobs finish gracefully on container redeployments.
* `wait -n`: Keeps PID 1 alive and monitors background worker jobs.

---

## 💻 7. Operational Command Reference

### 1. Connecting to Container & Loading Environment
```bash
# Log into container from EC2 host:
sudo docker exec -it <CONTAINER_ID> /bin/bash

# Load secrets into subshell (POSIX format):
. /tmp/secrets_env.sh

# Open Rails Console:
bundle exec rails c
```

### 2. Checking Process Concurrency
```bash
# From EC2 host:
sudo docker top <CONTAINER_ID>

# From inside container:
ps aux | grep delayed_job
```

### 3. Safe Read-Only Database Verification Queries (Rails Console)
```ruby
# 1. Check total pending jobs in queue
Delayed::Job.count

# 2. View currently locked/running jobs & worker PIDs
Delayed::Job.where.not(locked_at: nil).pluck(:id, :locked_at, :locked_by)

# 3. Check for permanently failed jobs
Delayed::Job.where.not(failed_at: nil).count

# 4. Check ready jobs vs future scheduled jobs
Delayed::Job.where("run_at <= ?", Time.now).count
Delayed::Job.where("run_at > ?", Time.now).count
```

### 4. Non-Destructive 5-Second Parallel Concurrency Test
```ruby
# Enqueue 3 non-destructive 5-second sleep jobs:
3.times { Kernel.delay.sleep(5) }

# Run immediately within 5 seconds to verify 3 active PIDs:
Delayed::Job.where.not(locked_at: nil).pluck(:id, :locked_at, :locked_by)
```

**Expected Output:**
```ruby
=> [
     [935910, Tue, 21 Jul 02:19:23, "host:59a83c249219 pid:45"],
     [935911, Tue, 21 Jul 02:19:23, "host:59a83c249219 pid:44"],
     [935912, Tue, 21 Jul 02:19:24, "host:59a83c249219 pid:43"]
   ]
```

---

## 💡 8. Tips & Tricks for Future Troubleshooting

1. **Tracking a Specific Policy PDF**: Search logs using the `RP` reference number (e.g. `RP1046519`):
   ```bash
   sudo docker logs <CONTAINER_ID> | grep "RP1046519"
   ```
2. **Decoding Log Header Prefixes**:
   `[2026-07-20T16:53:19.782574 #42]` ➔ Timestamp (UTC) + Linux Process ID (`PID #42`).
3. **Clearing Stale Locks**:
   If a container crashed leaving a stale lock, `delayed_job` auto-clears it after `max_run_time` (4 hours). To clear manually in console:
   ```ruby
   Delayed::Job.where("locked_at < ?", 2.hours.ago).update_all(locked_at: nil, locked_by: nil)
   ```
