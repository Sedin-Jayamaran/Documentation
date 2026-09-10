# Complete Incident Investigation Runbook & Root Cause Analysis (RCA)
**Service:** `portal.qreport.com.au` (JewelCover / Q-Report Production Portal)  
**Incident Type:** "503 Service Temporarily Unavailable" Outage  
**Date of Incident:** 9 September 2026  
**Outage Window:** 11:25 AM – 11:38 AM IST (05:55 – 06:08 UTC) — **Duration: ~13 minutes**  
**Total Failed Requests:** 859 (ELB 5XX Metric)  
**Environment:** AWS Account `463949419568`, Region `ap-southeast-2` (Sydney)  
**Cluster:** `qreport-production` (ECS on EC2)  
**Container Host:** `i-0fc2ad03328b243c9` (`m5.xlarge`, 4 vCPUs, 15,623 MiB RAM)  
**Document Author:** Jayamaran S (Sedin Technologies)  

---

## Table of Contents
1. [Executive Summary & True Root Cause](#1-executive-summary--true-root-cause)
2. [Why Previous Investigations Failed (The Fatal Flaw)](#2-why-previous-investigations-failed-the-fatal-flaw)
3. [The Full Correlated Chronological Timeline](#3-the-full-correlated-chronological-timeline)
4. [Step-by-Step Investigation Runbook (Where, How & What to Conclude)](#4-step-by-step-investigation-runbook-where-how--what-to-conclude)
   - [Phase 1: Proving the Outage Window and Empty Target Group](#phase-1-proving-the-outage-window-and-empty-target-group)
   - [Phase 2: Uncovering Why the Healthy Old Task Died (The Latency Spike)](#phase-2-uncovering-why-the-healthy-old-task-died-the-latency-spike)
   - [Phase 3: The Ground Truth of ECS Scheduler Events](#phase-3-the-ground-truth-of-ecs-scheduler-events)
   - [Phase 4: Proving Host CPU Saturation (94.81%) & Memory Pressure](#phase-4-proving-host-cpu-saturation-9481--memory-pressure)
   - [Phase 5: Verifying Connection Draining Delay](#phase-5-verifying-connection-draining-delay)
   - [Phase 6: Profiling Normal Workload Baseline vs Peak (7-Day Audit)](#phase-6-profiling-normal-workload-baseline-vs-peak-7-day-audit)
5. [Architectural Deep-Dive: Core Concepts Explained](#5-architectural-deep-dive-core-concepts-explained)
   - [What `cpu: 0` Actually Means](#what-cpu-0-actually-means)
   - [Hard Limit (`memory`) vs. Soft Limit (`memoryReservation`)](#hard-limit-memory-vs-soft-limit-memoryreservation)
   - [What Happens When a Container Crosses Its Hard Limit (8 GB)?](#what-happens-when-a-container-crosses-its-hard-limit-8-gb)
   - [Does Hitting 8 GB Cause Downtime? (DesiredCount 1 vs 2)](#does-hitting-8-gb-cause-downtime-desiredcount-1-vs-2)
6. [The Comprehensive Fix & Hardening Plan](#6-the-comprehensive-fix--hardening-plan)
7. [Ready-to-Apply Task Definition Configurations](#7-ready-to-apply-task-definition-configurations)

---

## 1. Executive Summary & True Root Cause

The 13-minute production outage on `portal.qreport.com.au` was caused by a **Cascading Resource Exhaustion (CPU & Memory) and Deployment Deadlock** on a single, overcommitted `m5.xlarge` EC2 host.

```
[Trigger: Co-locating qreport-shopify on the same single host]
  Total Configured RAM: 16,120 MiB > Physical Host RAM: 15,623 MiB
  Baseline Host RAM pushed to 85.5% (~13.3 GB used, only ~2.2 GB free)
       │
       ▼
[CodePipeline triggers concurrent deploy of qreport-app + qreport-dj]
       │
       ├─ New App: runs RAILS_ASSETS_PRECOMPILE=true (grabs 100% of all 4 vCPUs)
       ├─ New DJ: crashes on mkdir race condition, restarts, spawns 5 workers
       ├─ Shopify + Old App + Old DJ: actively running on the same host
       │
       ▼
[Host CPU Saturates at 94.81% / Cluster RAM surges to 93.38%]
       │
       ├─> New App fails health check (0s grace period + slow asset compile)
       │
       └─> Old App web worker threads starved of CPU -> response time spikes to 36.95s!
       │
       ▼
[ALB 5-second health check times out 3 consecutive times]
       │
       ▼
[ALB marks Old App UNHEALTHY -> ECS kills & deregisters it at 11:25:18]
       │
       ▼
[TARGET GROUP EMPTY: 0 Targets -> ALB returns 503 Service Unavailable (859 requests fail)]
       │
       ▼
[12.5-Minute Deadlock]: Docker suffers OutOfMemoryError during stop retries.
The ECS Agent control socket stalls under severe host memory thrashing.
At 11:37:50, the socket reconnects; queued task terminations complete.
Host RAM plummets from 93.4% to 14.3%! Replacement tasks start at 11:38:02. Full recovery at 11:39.
```

---

## 2. Why Previous Investigations Failed (The Fatal Flaw)

Early investigations (including initial Claude sessions) blamed `healthCheckGracePeriodSeconds = 0` as the "Primary Cause". **This was only half the story:**

1. **In ECS rolling deployments, a new container failing health check DOES NOT cause downtime.**
   Under normal rolling deployment mechanics (`minimumHealthyPercent = 100`), when a new container fails health checks, ECS terminates the *new* container. The *old* container continues serving live traffic. The deployment rolls back, but visitors experience **zero seconds of downtime**.
2. **The site went down because the OLD, stable container suddenly died at 11:25:17.**
   The old container had been running happily for 6 days. The earlier report listed this under *"Section 5: Unresolved Open Questions: Why did the old task time out?"* 
   **Without answering why the healthy container died, any root cause conclusion is invalid.**
3. **The Shopify Service was the Real Trigger:**
   Before adding Shopify, the host had ~4.5 GB of free RAM headroom. Adding Shopify brought total configured limits to 16,120 MiB (exceeding host RAM) and pushed baseline memory usage to 85.5% (leaving only ~2.2 GB free). When the deployment ran, the host had zero buffer to absorb the compilation spike.

---

## 3. The Full Correlated Chronological Timeline

Every timestamp below has been cross-verified using AWS CloudWatch metrics and ECS service event logs:

| Time (IST) | Time (UTC) | EC2 CPU | Cluster RAM | ALB Latency | Chronological Event Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Aug 27, 11:11** | Aug 27, 05:41 | Normal | Normal | Normal | `production-qreport-shopify` pipeline ran. Shopify task `6f6b5caa` started. |
| **Sep 03, 21:46** | Sep 03, 16:16 | Normal | Normal | Normal | `production-qreport` pipeline ran. Old App task `4f6030bae` and Old DJ `6a0df5a7` started. |
| **Sep 09, 10:08** | Sep 09, 04:38 | 5.1% | 85.0% | 0.25s | Service in steady state. Host running Old App, Old DJ, and Shopify (~85% RAM used). |
| **11:10:44** | `05:40:44` | 8.0% | 85.5% | 0.20s | CodePipeline execution starts for `production-qreport`. |
| **11:21:12** | `05:51:12` | 11.1% | 85.5% | 0.18s | CodePipeline reports SUCCEEDED (pipeline does not wait for ECS deployment stability). |
| **11:21:15** | `05:51:15` | 11.1% | 85.5% | 0.18s | ECS starts new App task `5e8c5710e1ec`. |
| **11:21:18** | `05:51:18` | 11.1% | 85.5% | 0.18s | ECS starts new DJ task `0756cd208114`. |
| **11:22:14** | `05:52:14` | 61.3% | 89.2% | 0.31s | New App container starts on port 33033; begins `RAILS_ASSETS_PRECOMPILE=true`. |
| **11:22:18** | `05:52:18` | 61.3% | 89.2% | 0.31s | Target registered in target group `qreport-production`. ALB begins health check probes. |
| **11:22:53** | `05:52:53` | 61.3% | 89.2% | 0.31s | New DJ container crashes (`mkdir: File exists @ dir_s_mkdir - /app/tmp/files`). |
| **11:22:54** | `05:52:54` | 61.3% | 89.2% | 0.31s | ECS immediately restarts DJ task `2ce3ac633027`, spawning 5 parallel workers. |
| **11:23:00** | `05:53:00` | **94.81%** | **88.3%** | 0.32s | **CPU PEGS AT 95%.** All 4 vCPUs fully saturated by asset compilation + DJ workers. |
| **11:23:32** | `05:53:32` | 94.81% | 88.3% | 0.32s | New App marked UNHEALTHY (`Health checks failed` after 74s). Grace period was 0. ECS stops task `5e8c5710`. |
| **11:23:33** | `05:53:33` | 94.81% | 88.3% | 0.32s | Target deregistered. Draining begins. Old App is still healthy and serving traffic. |
| **11:24:00** | `05:54:00` | 26.3% | 85.5% | **28.13s (Max 36.95s)** | **The Collapse:** Under CPU starvation, Old App Puma web server stalls. Average latency jumps from 0.32s to 28.1s (peaking at 36.95s). |
| **11:25:17** | `05:55:17` | 12.7% | 84.9% | *Timed Out* | ALB 5-second health check times out 3 consecutive times. **ALB marks Old App UNHEALTHY (`Request timed out`).** |
| **11:25:17** | `05:55:17` | 12.7% | 84.9% | *Timed Out* | ECS scheduler immediately issues STOP to Old App task `4f6030bae896`. |
| **11:25:18** | `05:55:18` | 12.7% | 84.9% | *Empty* | **Target deregistered. TARGET GROUP IS NOW COMPLETELY EMPTY (0 Targets).** |
| **11:25:27** | `05:55:27` | 12.7% | 84.9% | *Empty* | Old App enters connection draining (30s timeout). **503 Outage starts. 132 requests fail in this minute.** |
| **11:26–11:37** | `05:56–06:07` | 28–40% | **93.38%** | *Empty* | **The 12.5-Minute Deadlock:** 0 targets registered. Host memory peaks at 93.38% (14.5 GB). Docker encounters memory errors during stop retries. ECS Agent control socket is idle/stalled. |
| **11:37:50** | `06:07:50` | 28.3% | 93.4% | *Empty* | AWS drops the idle ACS control channel. Agent reconnects in 295ms. Queued `STOPPED` events are delivered. |
| **11:37:52** | `06:07:52` | 28.3% | 93.4% | *Empty* | Containers for Old App (`4f6030bae`) and Old DJ (`6a0df5a7`) finally terminate. |
| **11:38:00** | `06:08:00` | 40.0% | **14.3%** | **0.44s** | **Old containers terminate: Host memory plummets from 93.4% to 14.3%!** |
| **11:38:02** | `06:08:02` | 40.0% | 14.3% | 0.44s | ECS instantly launches replacement tasks `366c1039` and `c5b132f1`. |
| **11:38:13** | `06:08:13` | 40.0% | 14.3% | 0.44s | New target registered in target group. Begins health check probing. |
| **11:38:18** | `06:08:18` | 40.0% | 14.3% | 0.44s | Task `c5b132f1` starts running. Passes health checks. |
| **11:39:00** | `06:09:00` | 21.2% | 24.4% | 0.56s | **Full Recovery: 503 errors drop to 0.** Duplicate task `366c1039` stopped. |
| **11:41:43** | `06:11:43` | 10.0% | 31.6% | 0.28s | Deployment reported completed. Service reaches steady state. |

---

## 4. Step-by-Step Investigation Runbook (Where, How & What to Conclude)

Use this runbook to investigate future ECS incidents or verify past outages.

### Phase 1: Proving the Outage Window and Empty Target Group

* **Goal:** Determine the exact minutes the outage started and ended, and confirm if ALB was returning 503 because the target group was empty.
* **Where to Look:** CloudWatch ApplicationELB Metrics.

#### Command:
```bash
aws cloudwatch get-metric-statistics \
  --region ap-southeast-2 \
  --namespace AWS/ApplicationELB \
  --metric-name HTTPCode_ELB_503_Count \
  --dimensions Name=LoadBalancer,Value=app/qreport-production-ALB/f918dde635f519f8 \
  --start-time 2026-09-09T05:20:00Z \
  --end-time 2026-09-09T06:15:00Z \
  --period 60 \
  --statistics Sum \
  --output table
```

#### What We Found:
* Outage began at `05:55 UTC` (11:25 AM IST) with 132 failed requests in the first minute.
* Continued at 30–132 errors/minute through `06:08 UTC` (11:38 AM IST). Total failed requests: **859**.
* At `06:09 UTC` (11:39 AM IST), errors dropped to exactly **0**.

---

### Phase 2: Uncovering Why the Healthy Old Task Died (The Latency Spike)

* **Goal:** Determine why the old task (which ran fine for 6 days) failed health checks right during deployment.
* **Where to Look:** CloudWatch `TargetResponseTime` on the target group.

#### Command:
```bash
aws cloudwatch get-metric-statistics \
  --region ap-southeast-2 \
  --namespace AWS/ApplicationELB \
  --metric-name TargetResponseTime \
  --dimensions Name=TargetGroup,Value=targetgroup/qreport-production/01d80dc3d8d26dac Name=LoadBalancer,Value=app/qreport-production-ALB/f918dde635f519f8 \
  --start-time 2026-09-09T05:20:00Z \
  --end-time 2026-09-09T06:15:00Z \
  --period 60 \
  --statistics Average Maximum \
  --output table
```

#### What We Found:
* Normal latency baseline: **0.18s – 0.32s**.
* At **`05:54:00 UTC` (11:24 AM IST)**, latency exploded:
  * **Average:** **28.13 seconds**
  * **Maximum:** **36.95 seconds**
* **Conclusion:** Because ALB’s `HealthCheckTimeoutSeconds` is **5 seconds**, an application taking 28–37s to respond will time out 100% of the time. Three consecutive timeouts triggered the `Request timed out` failure that killed the container.

---

### Phase 3: The Ground Truth of ECS Scheduler Events

* **Goal:** View every single action taken by the ECS scheduler second-by-second.
* **Where to Look:** ECS `describe-services` event log.

#### Command:
```bash
aws ecs describe-services \
  --region ap-southeast-2 \
  --cluster qreport-production \
  --services qreport-app qreport-dj \
  --query "services[].{service:serviceName,events:events[?createdAt>='2026-09-09T05:20:00Z']}" \
  --output json
```

#### What We Found:
* `11:23:32`: New task `5e8c5710` marked UNHEALTHY (`Health checks failed`) and stopped.
* `11:25:17`: Old task `4f6030bae` marked UNHEALTHY (`Request timed out`) and stopped.
* `11:25:18`: `deregistered 1 targets in (target-group ...)` $\rightarrow$ **Target group was now empty.**
* `11:25:27` to `11:38:02`: **Complete silence for 12.5 minutes.** ECS did not log placement failures; it was waiting for task termination confirmation.
* `11:38:02`: Instantly launched replacement tasks `366c1039` and `c5b132f1`.

---

### Phase 4: Proving Host CPU Saturation (94.81%) & Memory Pressure

* **Goal:** Prove that the latency spike was caused by CPU/Memory starvation on the EC2 host.
* **Where to Look:** EC2 `CPUUtilization` and ECS `MemoryUtilization`.

#### Command (EC2 Host CPU):
```bash
aws cloudwatch get-metric-statistics \
  --region ap-southeast-2 \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-0fc2ad03328b243c9 \
  --start-time 2026-09-09T05:20:00Z \
  --end-time 2026-09-09T06:15:00Z \
  --period 60 \
  --statistics Average Maximum \
  --output table
```

#### Command (Cluster Memory):
```bash
aws cloudwatch get-metric-statistics \
  --region ap-southeast-2 \
  --namespace AWS/ECS \
  --metric-name MemoryUtilization \
  --dimensions Name=ClusterName,Value=qreport-production \
  --start-time 2026-09-09T05:20:00Z \
  --end-time 2026-09-09T06:15:00Z \
  --period 60 \
  --statistics Average Maximum \
  --output table
```

#### What We Found:
* **CPU:** At `05:53 UTC` (11:23 AM IST), EC2 CPU pegged at **94.81%** (all 4 vCPUs 100% maxed).
* **Memory:** Baseline memory was **85.5% (13.35 GB)**. During the deploy, memory surged to **93.38% (14.5 GB of 15.6 GB)**.
* **The Smoking Gun on Memory:** At `06:08 UTC` (11:38 AM IST), the exact minute the old containers stopped, memory **dropped from 93.4% down to 14.3%**!

---

### Phase 5: Verifying Connection Draining Delay

* **Goal:** Verify if ALB connection draining was responsible for the 12.5-minute freeze.
* **Where to Look:** Target group attributes.

#### Command:
```bash
aws elbv2 describe-target-group-attributes \
  --region ap-southeast-2 \
  --target-group-arn arn:aws:elasticloadbalancing:ap-southeast-2:463949419568:targetgroup/qreport-production/01d80dc3d8d26dac \
  --query "Attributes[?Key=='deregistration_delay.timeout_seconds']" \
  --output table
```

#### What We Found:
* `deregistration_delay.timeout_seconds = 30`.
* **Conclusion:** Connection draining took only 30 seconds (finished by 11:25:57). It was **not** the cause of the 12.5-minute delay. The delay was Docker/Agent memory contention.

---

### Phase 6: Profiling Normal Workload Baseline vs Peak (7-Day Audit)

We audited 7 days of historical 1-hour metrics (`2026-09-02` to `2026-09-09`) to size each service accurately:

| Service | CloudWatch Average | CloudWatch Maximum Peak | Real-World Activity Profile |
| :--- | :--- | :--- | :--- |
| **`qreport-app` (Web)** | **3% – 8%** CPU | **88% – 94%** CPU (Every hour!) | Average usage is light (~0.3 vCPU), but has sharp periodic bursts (PDFs, reports, Ruby GC) where it consumes all available cores. |
| **`qreport-dj` (Worker)** | **0.04% – 0.4%** CPU | Typically **1% – 5%** (Burst: 77.9%) | Background jobs idle 99% of the time (<0.05 vCPU). Only once a week does a batch job spike it. |
| **`qreport-shopify`** | **0.31% – 0.36%** CPU | **1.05% – 1.28%** CPU | **Ultra-lightweight:** Uses only ~40 CPU units (0.04 vCPU) and **175 MB of RAM** consistently. |

---

## 5. Architectural Deep-Dive: Core Concepts Explained

### What `cpu: 0` Actually Means

In AWS ECS, CPU is measured in units ($1024\text{ units} = 1\text{ vCPU}$). Your `m5.xlarge` instance has **4,096 units (4 vCPUs)**.

Setting `"cpu": 0` does **NOT** mean zero CPU. It means:
1. **For Scheduler Placement:** ECS does not check CPU availability. It crams the task onto the instance as long as RAM is available.
2. **For Docker Runtime:** Docker sets no CPU quota (`cpu.cfs_quota_us`). The container is allowed to **consume 100% of all 4 vCPUs on the host**.
3. **The Danger:** If multiple containers have `cpu: 0`, there are no fair shares. A new container compiling assets can grab all 4 cores, completely starving the live web server of CPU cycles.

---

### Hard Limit (`memory`) vs. Soft Limit (`memoryReservation`)

| Limit Type | Parameter | Role in ECS | What Happens When Reached |
| :--- | :--- | :--- | :--- |
| **Soft Limit** | `memoryReservation` | **The Scheduler Contract:** ECS uses this number to decide if a task fits on the EC2 instance. It guarantees the container this minimum amount of RAM. | Container is allowed to exceed this number and grow freely if the host has unused RAM. |
| **Hard Limit** | `memory` | **The Executioner Ceiling:** Configured in Linux kernel `cgroups`. Strict, non-negotiable maximum ceiling. | The Linux kernel **OOM Killer** immediately terminates the container with **Exit Code 137** (`SIGKILL`). |

#### The Trap That Broke Your Server:
* Your host has **15,623 MiB**.
* You configured:
  * `qreport-app`: Hard limit **12,000 MiB** (Soft: 4,000)
  * `qreport-dj`: Hard limit **3,096 MiB** (Soft: 1,024)
  * `qreport-shopify`: Hard limit **1,024 MiB** (Soft: 1,024)
  * **Total Hard Limits: 16,120 MiB** (Oversubscribed beyond host capacity!).
* When the deployment started, ECS checked the soft limits ($4,000 + 1,024 + 1,024 + 4,000 + 1,024 = 11,072\text{ MiB} \le 15,623\text{ MiB}$) and thought everything would fit.
* But at runtime, actual RAM demanded reached **17+ GB**. Because no single container reached its own hard limit, the kernel couldn't kill one container neatly—**the entire operating system choked on memory exhaustion**.

---

### What Happens When a Container Crosses Its Hard Limit (8 GB)?

1. **At byte 8,000,000,001**, the Linux kernel refuses to allocate more memory.
2. The kernel's **OOM Killer** sends an immediate `SIGKILL` (Signal 9) to the main process inside that container.
3. The container exits immediately with **Exit Code 137** ($128 + 9$).
4. ECS marks the task: `StoppedReason: OutOfMemoryError: Container killed due to memory usage`.
5. Because `essential: true` is set, ECS scheduler **automatically boots a fresh container** starting back at its clean baseline (~2 GB).
6. **Key Benefit:** Only the leaking container restarts. The host, Delayed Job, Shopify, and the OS remain **100% healthy**.

---

### Does Hitting 8 GB Cause Downtime? (DesiredCount 1 vs 2)

* **With `desiredCount: 1` (Your Current Setup):**
  **YES, a brief ~60 to 90 second downtime.** While the single container restarts and passes health checks, visitors will see a 502 or 503 error.
* **With `desiredCount: 2` (High Availability Setup):**
  **ZERO DOWNTIME.** 
  * Container 1 crashes at 8 GB.
  * ALB instantly shifts 100% of user traffic to Container 2.
  * ECS boots a fresh replacement for Container 1 in the background.
  * When healthy, ALB splits traffic 50/50 again. **No customer ever experiences an error.**

---

## 6. The Comprehensive Fix & Hardening Plan

### Fix 1: Right-Size Container Memory Limits (Eliminate Overcommit)
Eliminate memory oversubscription so that all 3 services leave **~5,000 MiB permanently free** on the host:

| Service | Current Hard Limit | New Soft Reservation | New Hard Limit | Impact |
| :--- | :--- | :--- | :--- | :--- |
| **`qreport-app`** | 12,000 MiB | **4,000 MiB** | **8,000 MiB** | Caps app growth at 8 GB; prevents runaway memory leaks. |
| **`qreport-dj`** | 3,096 MiB | **1,024 MiB** | **2,048 MiB** | Ample RAM for 5 workers; stops DJ from hogging host memory. |
| **`qreport-shopify`** | 1,024 MiB | **256 MiB** | **512 MiB** | Real usage is only 175 MB. 512 MB gives 3x headroom. |
| **Host Reserve** | 0 MiB | — | **~5,063 MiB FREE** | **Guaranteed OS, Docker, and deployment headroom.** |

---

### Fix 2: Set Explicit CPU Shares (cpu: 2048 for Web)
Change container `"cpu"` from 0 to explicit shares. In ECS on EC2, container-level CPU sets Docker `--cpu-shares`:

| Service | Current CPU | New CPU Shares | vCPU Equivalent | Runtime Behavior |
| :--- | :--- | :--- | :--- | :--- |
| **`qreport-app`** | 0 (unbounded) | **`2048`** | **2.0 vCPUs (50%)** | Guaranteed 50% CPU priority under contention; can burst to 90%+ when idle. |
| **`qreport-dj`** | 0 (unbounded) | **`512`** | **0.5 vCPU (12.5%)** | Normal usage is <0.4%; guaranteed capacity for batch jobs. |
| **`qreport-shopify`** | 512 | **`256`** | **0.25 vCPU (6.25%)** | Real usage is <0.04 vCPU; 256 units gives 6x headroom. |
| **Host / ECS Agent** | 0 | **`1280`** | **1.25 vCPUs** | **Unallocated buffer** so Docker and ECS agent never stall. |

---

### Fix 3: Move Rails Asset Precompilation to Docker Build
* **The Root Cause:** Running `RAILS_ASSETS_PRECOMPILE=true` on container boot takes 74 seconds and pegs CPU at 95%.
* **The Fix:** Move precompilation to the Dockerfile / CodeBuild stage:
  ```dockerfile
  RUN bundle exec rake assets:precompile
  ```
* Set `RAILS_ASSETS_PRECOMPILE=false` in the task definition environment. Boot time drops from 74 seconds to **under 3 seconds**.

---

### Fix 4: Health Check Grace Period & Timeout Tuning
Apply immediately to protect rolling deployments:
```bash
aws ecs update-service \
  --region ap-southeast-2 \
  --cluster qreport-production \
  --service qreport-app \
  --health-check-grace-period-seconds 300
```
* Increase ALB `HealthCheckTimeoutSeconds` from **5 seconds** to **10 seconds** on the target group.

---

### Fix 5: Prevent Rails In-App Memory Bloat (`puma-worker-killer`)
Add the `puma-worker-killer` gem to `Gemfile` to recycle fat Puma worker processes gracefully before they ever approach 8 GB:
```ruby
# In config/puma.rb
before_fork do
  PumaWorkerKiller.config do |config|
    config.ram           = 1024 # MB per worker
    config.frequency     = 10   # seconds
    config.rolling_restart_frequency = 12 * 3600 # 12 hours
  end
  PumaWorkerKiller.start
end
```

---

### Fix 6: Deployment Safety Rails & Alarms
1. **Enable Deployment Circuit Breaker with Rollback:**
   ```bash
   aws ecs update-service \
     --region ap-southeast-2 \
     --cluster qreport-production \
     --service qreport-app \
     --deployment-configuration "deploymentCircuitBreaker={enable=true,rollback=true}"
   ```
2. **Add CloudWatch Alarm on HealthyHostCount:**
   * Metric: `HealthyHostCount < 1` on target group `qreport-production`.
   * Alarm Action: Send immediate P1 alert via SNS / PagerDuty / Slack.

---

## 7. Ready-to-Apply Task Definition Configurations

### `production-qreport-app`
```json
{
  "name": "qreport-app",
  "image": "463949419568.dkr.ecr.ap-southeast-2.amazonaws.com/qreport-production:latest",
  "cpu": 2048,
  "memoryReservation": 4000,
  "memory": 8000,
  "essential": true,
  "portMappings": [
    {
      "containerPort": 3000,
      "hostPort": 0,
      "protocol": "tcp"
    }
  ],
  "environment": [
    {
      "name": "RAILS_ASSETS_PRECOMPILE",
      "value": "false"
    }
  ]
}
```

### `production-qreport-dj`
```json
{
  "name": "qreport-dj",
  "image": "463949419568.dkr.ecr.ap-southeast-2.amazonaws.com/qreport-production:latest",
  "cpu": 512,
  "memoryReservation": 1024,
  "memory": 2048,
  "essential": true
}
```

### `production-qreport-shopify`
```json
{
  "name": "qreport-shopify",
  "image": "463949419568.dkr.ecr.ap-southeast-2.amazonaws.com/qreport-shopify:latest",
  "cpu": 256,
  "memoryReservation": 256,
  "memory": 512,
  "essential": true
}
```

---
*End of Investigation Runbook and Root Cause Analysis Document.*
