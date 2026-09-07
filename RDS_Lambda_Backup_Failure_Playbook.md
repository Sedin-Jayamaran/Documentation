# Playbook: Debugging a Missed/Failed Scheduled Database Backup
### (EventBridge → Lambda → RDS → EFS → S3 pipeline)

**Origin case:** `qreport-production` MySQL backup pipeline, incident of **September 5, 2026**
**Pipeline:** EventBridge rule `cron_prod_sydney_db_dump` → Lambda `prod_mysql_dump_script` → RDS `qreport-production` → EFS `fs-0847d280bbf3a9fd9` → S3 `qreport-prod-dbbackup`
**Region:** `ap-southeast-2` (Sydney)

This is a step-by-step, layer-by-layer playbook for diagnosing **any** future failure in this pipeline. Every step below includes: what to check, the exact AWS CLI command, the console path, what a healthy result looks like, and what an unhealthy result means. The real Sept 5 walkthrough is included throughout as a worked example, and summarized end-to-end in the Appendix.

---

## 0. Mental model — the 5 layers to check, in order

A missed backup can fail at any of five layers. Always check them **in this order**, top to bottom — each layer's health determines whether the next layer's data is even worth looking at.

```
1. SCHEDULE      Did the cron rule fire on time?              (EventBridge)
2. TRIGGER       Did EventBridge successfully invoke Lambda?  (EventBridge → Lambda handoff)
3. COMPUTE       Did Lambda run, and did it error/timeout?    (Lambda)
4. STORAGE/DB    Did the actual mysqldump + write succeed?    (RDS + EFS)
5. DELIVERY      Did the file actually land in S3?             (S3)
```

Never skip straight to "check the database" — in this incident, the schedule and trigger layers were both perfectly healthy, and jumping to conclusions there would have wasted time. Confirm each layer before moving to the next.

---

## 1. Decode the schedule first

Before touching any metrics, know exactly what "on schedule" means for this pipeline.

```
Rule: cron_prod_sydney_db_dump
Expression: 0 00,03,06,09,12,13 * * ? *
```

AWS EventBridge cron/scheduled expressions are **always in UTC**, never local time. Break the expression down field by field:

```
0 00,03,06,09,12,13 * * ? *
│      │              │ │ └─ Year
│      │              │ └─── Day-of-week (unused, day-of-month is *)
│      │              └───── Month (every)
│      └──────────────────── Day-of-month (every)
└─────────────────────────── Minute: 0
       └─ Hours: 00, 03, 06, 09, 12, 13
```

**Fires at:** 00:00, 03:00, 06:00, 09:00, 12:00, 13:00 UTC daily.

⚠️ **Red flag found in this exercise:** the gap between 12:00 and 13:00 is only 1 hour, versus 3 hours everywhere else, and the gap from 13:00 back to 00:00 the next day is 11 hours. This uneven spacing is not just cosmetic — it meant the 13:00 slot had the least time to recover from any resource strain (see the EFS section below), making it structurally the most fragile slot in the schedule.

**Convert to your local time** (example: IST, UTC+5:30) before reading any console graph, since consoles frequently default to browser-local time:

| UTC | IST |
|---|---|
| 00:00 | 05:30 AM |
| 03:00 | 08:30 AM |
| 06:00 | 11:30 AM |
| 09:00 | 02:30 PM |
| 12:00 | 05:30 PM |
| 13:00 | 06:30 PM |

---

## 2. Layer 1 — SCHEDULE: did EventBridge fire on time?

### Metrics to check (namespace `AWS/Events`)

| Metric | Meaning if missing/zero |
|---|---|
| `TriggeredRules` | The cron itself never fired — a scheduling problem |
| `MatchedEvents` | Same rule-engine level check |
| `InvocationAttempts` | Rule fired, but EventBridge never attempted to invoke the target |
| `SuccessfulInvocationAttempts` | EventBridge attempted the invoke, but the handoff to Lambda itself failed (permissions, throttling) |

### CLI — pull all four for a given day

```bash
RULE_NAME="cron_prod_sydney_db_dump"
REGION="ap-southeast-2"
START="2026-09-05T00:00:00Z"
END="2026-09-06T00:00:00Z"

for METRIC in TriggeredRules MatchedEvents InvocationAttempts SuccessfulInvocationAttempts; do
  echo "=== $METRIC ==="
  aws cloudwatch get-metric-statistics \
    --namespace AWS/Events \
    --metric-name "$METRIC" \
    --dimensions Name=RuleName,Value="$RULE_NAME" \
    --start-time "$START" \
    --end-time "$END" \
    --period 60 \
    --statistics Sum \
    --region "$REGION" \
    --output table
  echo ""
done
```

### Console path
`CloudWatch → Metrics → All metrics → Events → By Rule Name → select your rule → check Invocations/FailedInvocations`
- Switch the time picker's timezone to **UTC** (top of the graph) before reading anything — this console defaults to browser-local time.
- Set the **period to 1 minute** in the Options tab if bars look merged.

### What we found (worked example, Sept 5)
All 4 metrics showed **exactly 6 datapoints each**, matching the cron schedule perfectly (`00:00, 02:59/03:00, 06:00, 08:59/09:00, 11:59/12:00, 12:59/13:00` — the `:59` entries are normal ~1 min scheduling jitter, not a bug).

**Conclusion at this layer: EventBridge was 100% healthy.** Move to Layer 2.

---

## 3. Layer 2 — COMPUTE: did Lambda run, and did it fail?

### Metrics to check (namespace `AWS/Lambda`)

| Metric | What to look for |
|---|---|
| `Invocations` | Should match the trigger count exactly (6/day here) |
| `Errors` | Any Sum > 0 means the function threw or crashed |
| `Duration` (Maximum) | If it equals your configured timeout exactly (e.g. `900000.0` ms = 15 min), the function was **force-killed**, not cleanly erroring |
| `Throttles` | Any Sum > 0 means concurrency limits were hit — different root cause entirely (not a code bug) |

### CLI

```bash
FUNCTION_NAME="prod_mysql_dump_script"
REGION="ap-southeast-2"
START="2026-09-05T00:00:00Z"
END="2026-09-06T00:00:00Z"

for METRIC in Invocations Errors Duration Throttles; do
  echo "=== Lambda $METRIC ==="
  aws cloudwatch get-metric-statistics \
    --namespace AWS/Lambda \
    --metric-name "$METRIC" \
    --dimensions Name=FunctionName,Value="$FUNCTION_NAME" \
    --start-time "$START" \
    --end-time "$END" \
    --period 60 \
    --statistics Sum Maximum \
    --region "$REGION" \
    --output table
done
```

### Console path
`Lambda → [function name] → Monitor tab → view metrics`, or use CloudWatch directly:
`CloudWatch → Metrics → Lambda → By Function Name`

### What we found (worked example)
`Invocations` showed **18 datapoints for the day, not 6** — because each of the 6 scheduled slots retried **twice** (Lambda's default async retry behavior: 1 original attempt + 2 automatic retries), giving 3 attempts per slot × 6 slots = 18. `Errors` showed the **exact same 18 timestamps**, meaning **every single attempt, including every retry, failed**. `Duration` showed several entries pinned at exactly `900000.0` ms — the configured Lambda timeout — confirming some attempts hung until force-killed rather than failing cleanly.

**Retry pattern to recognize:** scheduled slot `HH:00` retrying at roughly `HH:16` and `HH:25` (9–16 minutes apart) is the signature of Lambda's default async retry with backoff. If you see this fan-out pattern in `Invocations`, know that N failures in the metric = (scheduled slots) × 3, not N separate incidents.

**Conclusion at this layer: Lambda was being invoked correctly by EventBridge, but the function itself was failing on every attempt.** Move to Layer 3 — read the actual error.

---

## 4. Layer 3 — Read the actual error from CloudWatch Logs

### Find the log group
Lambda logs always live at:
```
/aws/lambda/<function-name>
```
Confirm the exact name via console (`Lambda → function → Monitor → View CloudWatch logs`) or:
```bash
aws logs describe-log-groups \
  --log-group-name-prefix "/aws/lambda/prod_mysql_dump_script" \
  --region ap-southeast-2
```

### CLI — pull logs for a specific time window

⚠️ **Common pitfall #1 — macOS vs Linux `date` command.** macOS ships BSD `date`, which does **not** support `-d`. This will fail on a Mac:
```bash
# THIS FAILS ON macOS:
date -d "2026-09-05T06:08:30Z" +%s%3N
# error: "illegal option -- d"
```
Fix — use BSD syntax with explicit UTC:
```bash
date -u -j -f "%Y-%m-%dT%H:%M:%SZ" "2026-09-05T06:08:30Z" "+%s"
```
**Or, simplest and most reliable: skip `date` math entirely and use precomputed epoch milliseconds directly.**

⚠️ **Common pitfall #2 — forgetting `-u` (UTC) with `date -j`.** Without `-u`, BSD `date -j -f` interprets your input string as **local time**, silently shifting your query window by your timezone offset (5.5 hours for IST) and making you search the wrong window entirely, often returning an empty result that looks like "no logs exist" when they actually do.

```bash
aws logs filter-log-events \
  --log-group-name "/aws/lambda/prod_mysql_dump_script" \
  --start-time 1788588510000 \
  --end-time 1788588780000 \
  --region ap-southeast-2 \
  --output text
```

### CLI — structured query via Logs Insights (better for scanning a whole day)
```bash
aws logs start-query \
  --log-group-name "/aws/lambda/prod_mysql_dump_script" \
  --start-time 1788566400 \
  --end-time 1788652799 \
  --region ap-southeast-2 \
  --query-string 'fields @timestamp, @message, @requestId | filter @message like /START|END|REPORT|Error|error|timeout/ | sort @timestamp asc'
```
(note: `start-query` takes **seconds**, `filter-log-events` takes **milliseconds** — easy to mix up)

Fetch the results:
```bash
aws logs get-query-results --query-id <id-from-above> --region ap-southeast-2 --output table
```

### Console path
`CloudWatch → Log groups → /aws/lambda/<function> → Logs Insights` — set time range to **UTC** explicitly (same timezone trap as everywhere else in this playbook), run the same query string above.

### What we found (worked example)
```
ERROR   Invoke Error    {"errorType":"Error","errorMessage":"Command failed: /mnt/dbdump/db_backup.sh
mysqldump: Error 2013: Lost connection to MySQL server during query when dumping table `conversations` at row: 27203"
...}
```

**This is your smoking gun log line.** It tells you: the exact command that failed, the exact table, and the exact approximate row. Move to Layer 4 to find out *why* the connection was lost.

---

## 5. Error code reference — MySQL / mysqldump errors you're likely to see

| Error | Meaning | Likely cause |
|---|---|---|
| **Error 2013** — `Lost connection to MySQL server during query` | Connection dropped mid-read/write | Network stall, idle-connection eviction, server-side timeout, or (as in this case) client-side I/O backpressure from a slow disk |
| **Error 2006** — `MySQL server has gone away` | Connection dropped *between* queries (not mid-query) | Usually `wait_timeout` expiry on an idle connection, or server restart |
| **Error 1153** — `Got a packet bigger than 'max_allowed_packet' bytes` | A row/result exceeded the packet size limit | Large blob/text columns exceeding `max_allowed_packet`; raise it on both client and server (parameter group) |
| **Error 1205** — `Lock wait timeout exceeded` | A query waited too long for a lock and gave up | Concurrent write holding a lock on the same table/rows |
| **Error 1040** — `Too many connections` | Server hit `max_connections` | Connection leak, or genuine traffic spike |
| **Error 1044/1045** — Access denied | Credential or privilege issue | Wrong password, revoked grant, or IAM auth misconfiguration |

**How to tell these apart when you only have "Error 2013" in hand:** it's a generic "the pipe broke" error — the real cause is *always* in a different data source (DB parameter group, Performance Insights, storage metrics), never in the error message itself. Don't stop at the error code — it's the start of the investigation, not the end.

---

## 6. Layer 4a — Database-side checks (RDS)

### 6.1 Check RDS Recent Events
**Console:** `RDS → Databases → [instance] → Logs & events tab → Recent events panel`
- Default view is **"Last 1 day"** — change to **"Last 7 days"** to see anything relevant to a past incident.
- Look for: failovers, maintenance windows, storage-full warnings, parameter group changes, automated backup windows (these can briefly increase I/O load on single-AZ instances).

### 6.2 Check the RDS/MySQL error log
**Console:** same tab → **Logs panel** → click a log file → **View**
- ⚠️ If **no log file exists for your incident date**, that itself is a finding: it means the MySQL engine itself never logged a hard error — the disconnect happened at a layer below the database engine (network, or client-side I/O), not from the DB actively rejecting/crashing.

### 6.3 Check timeout-related parameters
**Console:** `RDS → instance → Configuration tab → note the Parameter Group name → Parameter groups → search`

Or via CLI:
```bash
aws rds describe-db-parameters \
  --db-parameter-group-name <your-parameter-group-name> \
  --region ap-southeast-2 \
  --query "Parameters[?ParameterName=='wait_timeout' || ParameterName=='interactive_timeout' || ParameterName=='net_read_timeout' || ParameterName=='net_write_timeout' || ParameterName=='max_allowed_packet']" \
  --output table
```

A value of `-` means it's at the **MySQL engine default** (not customized):
| Parameter | Engine default |
|---|---|
| `wait_timeout` | 28,800 seconds (8 hours) |
| `interactive_timeout` | 28,800 seconds |
| `net_read_timeout` | 30 seconds |
| `net_write_timeout` | 60 seconds |
| `max_allowed_packet` | typically 4–64 MB depending on engine version |

**What we found:** all of these were at default — ruling out a misconfigured timeout as the cause. This eliminated one whole branch of possibilities and pointed the investigation elsewhere.

### 6.4 Check whether query-level logging is even available
```bash
aws rds describe-db-parameters \
  --db-parameter-group-name <your-parameter-group-name> \
  --region ap-southeast-2 \
  --query "Parameters[?ParameterName=='slow_query_log' || ParameterName=='general_log' || ParameterName=='long_query_time']" \
  --output table
```
If `Source: engine-default` for `slow_query_log`, **it has never been turned on** — meaning there is no way to retroactively see what queries ran during a past incident. This is a permanent forensic dead-end for anything before the day you enable it.

**Turn it on now, for next time** (dynamic parameters — no reboot needed):
```bash
aws rds modify-db-parameter-group \
  --db-parameter-group-name <your-parameter-group-name> \
  --parameters \
    "ParameterName=slow_query_log,ParameterValue=1,ApplyMethod=immediate" \
    "ParameterName=long_query_time,ParameterValue=5,ApplyMethod=immediate" \
  --region ap-southeast-2

aws rds modify-db-instance \
  --db-instance-identifier <your-db-instance-id> \
  --cloudwatch-logs-export-configuration '{"EnableLogTypes":["slowquery"]}' \
  --apply-immediately \
  --region ap-southeast-2
```
**Cost:** effectively negligible at normal query volumes (first 5GB/month of CloudWatch Logs is free; a slow-query log with a 5-second threshold typically generates only KB–MB/month unless something is genuinely wrong often).

### 6.5 CloudWatch Database Insights (Performance Insights) — the richest data source, if within retention

**Retention window:** typically **7 days on the free tier** — check yours:
```bash
aws rds describe-db-instances \
  --db-instance-identifier <your-db-instance-id> \
  --region ap-southeast-2 \
  --query "DBInstances[0].{Enabled:PerformanceInsightsEnabled, RetentionDays:PerformanceInsightsRetentionPeriod}" \
  --output table
```
If `Enabled: true` and you're within the retention window, **check this before it ages out** — it's the single richest source of "what was actually running at that exact minute."

**Console:** `RDS instance → CloudWatch Database Insights (left sidebar)` or `CloudWatch → Database Insights`
1. Set the **time range explicitly** to your incident window (e.g. `06:00:00` – `06:20:00`), **timezone = UTC**
2. Check **"DB Load" graph**, sliced by **Waits** — look for a spike and which wait-event color dominates it
3. Check **"Top SQL"** tab — ranks actual SQL statements by load during the selected window
4. Check **"Top waits"** tab — ranks wait event types by load

**Key wait-event categories to recognize:**
| Wait event | Meaning |
|---|---|
| `wait/io/socket/sql/client_connection` | Server is waiting to **send data to the client** — the client isn't reading fast enough. Points to a slow/stalled client-side consumer (e.g. a slow disk write on the client) |
| `wait/io/table/sql/handler` | Table-level I/O handler wait — often accompanies large scans/dumps |
| `wait/synch/sxlock/innodb/*` or any `Lock/*` | Actual row/table locking from a concurrent query |
| `CPU` | Genuine compute-bound query, not I/O |

**Recognizing your own `mysqldump` traffic:** by default, `mysqldump` prefixes every SELECT with `SQL_NO_CACHE`. Any `SELECT SQL_NO_CACHE * FROM <table>` in Top SQL is your own backup script, not an external competing query — don't mistake it for the "culprit."

**What we found (worked example):** `wait/io/socket/sql/client_connection` dominated at 0.49 AAS — nearly half the total load — with **no lock-wait event in the top 6 at all**. This ruled out "a concurrent app query locking the table" and pointed specifically at a **client-side consumption bottleneck** — the client (Lambda) wasn't reading data from its own DB connection fast enough. That pointed the investigation toward *why* the client was slow — which turned out to be Layer 4b, below.

---

## 7. Layer 4b — Storage-side checks (EFS, if your Lambda writes to a mounted filesystem)

### 7.1 Confirm the Lambda is using EFS, and get the filesystem ID
```bash
aws lambda get-function-configuration \
  --function-name prod_mysql_dump_script \
  --region ap-southeast-2 \
  --query "FileSystemConfigs" \
  --output table
```
This returns an **access point ARN**, not the filesystem ID directly. Resolve it:
```bash
aws efs describe-access-points \
  --access-point-id <fsap-xxxxxxxxxxxx-from-above> \
  --region ap-southeast-2 \
  --query "AccessPoints[0].FileSystemId" \
  --output text
```

### 7.2 Check throughput mode — this determines which metric matters
```bash
aws efs describe-file-systems \
  --file-system-id <fs-xxxxxxxxxxxx> \
  --region ap-southeast-2 \
  --query "FileSystems[0].{ThroughputMode:ThroughputMode, SizeBytes:SizeInBytes.Value}" \
  --output table
```
- **`bursting`** → check `BurstCreditBalance` (below). This mode has a finite credit pool that depletes under sustained write load and refills slowly when idle.
- **`elastic`** or **`provisioned`** → burst credits don't apply; check `PercentIOLimit` and `ThroughputUtilization` instead.

### 7.3 Check BurstCreditBalance across the incident window
```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/EFS \
  --metric-name BurstCreditBalance \
  --dimensions Name=FileSystemId,Value=<fs-xxxxxxxxxxxx> \
  --start-time 2026-09-05T00:00:00Z \
  --end-time 2026-09-06T00:00:00Z \
  --period 300 \
  --statistics Average Minimum \
  --region ap-southeast-2 \
  --output table
```

**How to read it:**
- Healthy: comfortably high (tens of billions of bytes for a large filesystem), gently rising/falling with usage
- Depleted: flat at **`0.0`** across a sustained window — this is throttling in effect, right now, not a future risk

**Always compare against a known-good day**, not just the incident day in isolation — the absolute number means little without a baseline:
```bash
# Re-run the same command with a normal day's date range for comparison
```

### 7.4 Check actual data volume moved (for cost estimation, or to confirm abnormal write volume)
```bash
for METRIC in DataReadIOBytes DataWriteIOBytes; do
  echo "=== $METRIC ==="
  aws cloudwatch get-metric-statistics \
    --namespace AWS/EFS \
    --metric-name "$METRIC" \
    --dimensions Name=FileSystemId,Value=<fs-xxxxxxxxxxxx> \
    --start-time 2026-09-04T00:00:00Z \
    --end-time 2026-09-06T00:00:00Z \
    --period 3600 \
    --statistics Sum \
    --region ap-southeast-2 \
    --output table
done
```

### What we found (worked example)
| Date | Lowest `BurstCreditBalance` observed | Result |
|---|---|---|
| Sept 3 (normal day) | ~36.9 billion bytes — never came close to zero | 6/6 backups succeeded, ~5 min each |
| Sept 4, from 13:10 onward | **0 bytes** | First slot (13:00 UTC) already showing elevated duration (~13.7 min vs normal ~5 min) |
| Sept 5, all day | **0 bytes**, sustained the entire day | 0/6 backups succeeded — full outage |
| Sept 6 onward | Fully recovered | 6/6 backups succeeded again, ~5 min each |

**This confirmed the mechanism:** credits crashed after an abnormally long run on Sept 4 (13:00 UTC), didn't fully recover overnight (only ~11 hours of idle time before the next day's runs began), and every subsequent run on Sept 5 started already throttled — causing writes to stall, which caused `mysqldump` to stop consuming its DB connection fast enough, which caused RDS to drop the connection (`Error 2013`), exactly matching the `wait/io/socket/sql/client_connection` spike seen in Database Insights.

**Open question this doesn't answer:** what specifically triggered the abnormal Sept 4, 13:00 run in the first place. CloudWatch data confirms the effect (credits crashing) but not the ultimate original trigger. Worth checking with the team for anything unusual (deploy, migration, bulk data operation) around that specific time if this recurs.

---

## 8. Layer 5 — Confirm actual delivery (or non-delivery) in S3

### Find the bucket and prefix pattern
Check the Lambda's environment variable:
```bash
aws lambda get-function-configuration \
  --function-name prod_mysql_dump_script \
  --region ap-southeast-2 \
  --query "Environment.Variables.S3_BUCKET_NAME" \
  --output text
```

### List backups for a given date
The filename pattern in this pipeline is `mysqlbackup-D-M-YYYY-H:M:S.sql.gz` (note: **no leading zeros** on day/month/hour — e.g. `mysqlbackup-7-9-2026`, not `07-09-2026`).

```bash
BUCKET_NAME="qreport-prod-dbbackup"
DAY="5"; MONTH="9"; YEAR="2026"

aws s3api list-objects-v2 \
  --bucket "$BUCKET_NAME" \
  --prefix "DB-BACKUP/mysqlbackup-${DAY}-${MONTH}-${YEAR}" \
  --region ap-southeast-2 \
  --query "Contents[].{Key:Key, SizeBytes:Size, LastModified:LastModified}" \
  --output table
```

### What to check
- **Fewer than expected files** → confirms exactly which slot(s) failed to reach S3, by comparing timestamps against your cron schedule
- **File present but size = 0 or suspiciously small** → the script "succeeded" (no thrown error) but produced a corrupt/empty file — this is a gap in the current script (no validation before upload); treat this as equally bad as a missing file
- **All files present, consistent size** → that day's backups are genuinely healthy

### What we found (worked example)
Sept 5: **zero files returned** — full confirmation that no backup reached S3 that entire day, consistent with every layer above.
Sept 7 (post-fix baseline): 3 files present by mid-morning, consistent ~908MB sizes — confirms normal operation resumed.

---

## 9. Root-cause synthesis checklist

Once you've walked all 5 layers, use this checklist to write your conclusion:

- [ ] Layer 1 (Schedule): did `TriggeredRules`/`MatchedEvents` show the expected count? 
- [ ] Layer 2 (Trigger): did `SuccessfulInvocationAttempts` match `InvocationAttempts`?
- [ ] Layer 3 (Compute): did `Invocations` match expected count × (1 + retries)? Any `Errors`? Any `Duration` pinned at the timeout ceiling?
- [ ] Layer 3 (Logs): what's the exact error message and error code?
- [ ] Layer 4a (DB config): are `wait_timeout`/`net_read_timeout`/`net_write_timeout`/`max_allowed_packet` at safe values?
- [ ] Layer 4a (DB Insights): what wait event dominates the load during the exact failure window? Any lock waits?
- [ ] Layer 4b (Storage): if EFS-backed, what mode is it in, and was `BurstCreditBalance` healthy at the failure time vs. a normal day?
- [ ] Layer 5 (Delivery): did any files actually reach S3 that day, and at what sizes?

**Only when every layer above the actual root cause checks out clean, and the layer at fault shows a clear anomaly vs. a known-good baseline, do you have a defensible conclusion** — not before.

---

## 10. Remediation checklist (apply after root cause is confirmed)

### Immediate (security)
- [ ] **Rotate any DB credentials** that appeared in plaintext in CloudWatch Logs (this pipeline's script passed the password via `--password=` on the command line, which gets logged verbatim on any command failure)
- [ ] Switch to a MySQL `--defaults-extra-file` option file instead of command-line credentials, to keep passwords out of both `ps aux` and error logs going forward

### Storage fix (if EFS bursting-mode exhaustion is the cause)
```bash
aws efs update-file-system \
  --file-system-id <fs-xxxxxxxxxxxx> \
  --throughput-mode elastic \
  --region ap-southeast-2
```
- No downtime required
- Removes the credit-exhaustion failure mode entirely
- New cost dimension introduced: ~$0.06/GB written + $0.03/GB read (versus bundled-into-storage cost under Bursting) — estimate your actual increase via `DataReadIOBytes`/`DataWriteIOBytes` (Section 7.4) before/after
- **Better long-term alternative to evaluate:** stream `mysqldump` output directly into S3 (multipart upload) without staging on a local/EFS disk at all — removes this failure class and the added cost entirely, at the cost of a larger code change

### Code hardening (Lambda script)
- [ ] Add an explicit `timeout` option to `execFile()`, shorter than the Lambda's own configured timeout, so failures produce a clean, readable error instead of a hard kill at the ceiling
- [ ] Add a `maxBuffer` option to `execFile()` to avoid silent buffer-overflow failures on verbose script output
- [ ] Add retry logic *inside* the script for transient failures, rather than relying solely on Lambda's default async retries (which just repeat the identical failure if the underlying condition — e.g. depleted storage credits — hasn't cleared)
- [ ] Validate the dump file's size (non-trivial, non-zero) before compressing/uploading, so a corrupt/partial dump never silently looks like a success
- [ ] Give the temporary shell script a unique filename per invocation, not a shared fixed path — prevents concurrent invocations from overwriting each other's in-flight script

### Schedule fix
- [ ] Fix the uneven cron gap — even spacing (e.g. every 4 hours: `0 0,4,8,12,16,20 * * ? *`) gives every slot equal recovery time, removing the structural weak point that made one slot more fragile than the rest

### Observability (so next time is same-day, not 2 days later via a support ticket)
- [ ] Enable `slow_query_log` on the RDS parameter group (Section 6.4) — cheap, and gives exact query-level visibility for any future incident
- [ ] Set a CloudWatch Logs retention policy on the new slow query log group to avoid unbounded storage growth:
```bash
aws logs put-retention-policy \
  --log-group-name "/aws/rds/instance/<your-db-instance-id>/slowquery" \
  --retention-in-days 30 \
  --region ap-southeast-2
```
- [ ] Add a CloudWatch Alarm directly on the Lambda's `Errors` metric, so any future failure triggers same-day notification instead of relying on a downstream team noticing missing data days later:
```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "prod_mysql_dump_script-errors" \
  --namespace AWS/Lambda \
  --metric-name Errors \
  --dimensions Name=FunctionName,Value=prod_mysql_dump_script \
  --statistic Sum \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 0 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions <your-sns-topic-arn> \
  --region ap-southeast-2
```
- [ ] Consider a CloudWatch Alarm on `BurstCreditBalance` (if staying on Bursting mode) with a threshold well above zero, to catch depletion *before* it causes a failure, not after

---

## 11. Console navigation quick-reference

| What you need | Console path |
|---|---|
| EventBridge rule metrics | CloudWatch → Metrics → All metrics → Events → By Rule Name |
| Lambda invocation/error metrics | Lambda → function → Monitor tab, or CloudWatch → Metrics → Lambda |
| Lambda logs | CloudWatch → Log groups → `/aws/lambda/<function-name>` → Logs Insights |
| RDS instance overview | RDS → Databases → click instance |
| RDS recent events + error logs | RDS instance → Logs & events tab |
| RDS parameter values | RDS instance → Configuration tab → note parameter group name → Parameter groups → search |
| RDS query-level diagnostics | RDS instance → CloudWatch Database Insights (left sidebar), or CloudWatch → Database Insights |
| EFS filesystem details | Lambda function → Configuration → File systems (for the access point), then EFS console for the filesystem itself |
| EFS metrics | CloudWatch → Metrics → EFS → By File System Id |
| S3 backup files | S3 → bucket → navigate to prefix, sort by "Last modified" |

**Universal gotcha across every console page above:** always check the timezone toggle near the date/time picker and explicitly set it to **UTC** before reading any graph or setting any custom range — several consoles default to browser-local time, which silently shifts your entire investigation window and can make real data look like "nothing happened" during the actual incident time.

---

## 12. CLI cheat sheet — every command used in this investigation, by service

### EventBridge
```bash
aws cloudwatch get-metric-statistics --namespace AWS/Events --metric-name <TriggeredRules|MatchedEvents|InvocationAttempts|SuccessfulInvocationAttempts> --dimensions Name=RuleName,Value=<rule-name> --start-time <ISO8601> --end-time <ISO8601> --period 60 --statistics Sum --region <region> --output table
```

### Lambda
```bash
aws cloudwatch get-metric-statistics --namespace AWS/Lambda --metric-name <Invocations|Errors|Duration|Throttles> --dimensions Name=FunctionName,Value=<function-name> --start-time <ISO8601> --end-time <ISO8601> --period 60 --statistics Sum Maximum --region <region> --output table

aws lambda get-function-configuration --function-name <function-name> --region <region> --query "Environment.Variables.<VAR_NAME>" --output text

aws lambda get-function-configuration --function-name <function-name> --region <region> --query "FileSystemConfigs" --output table

aws logs filter-log-events --log-group-name "/aws/lambda/<function-name>" --start-time <epoch-ms> --end-time <epoch-ms> --region <region> --output text

aws logs start-query --log-group-name "/aws/lambda/<function-name>" --start-time <epoch-sec> --end-time <epoch-sec> --region <region> --query-string 'fields @timestamp, @message, @requestId | filter @message like /START|END|REPORT|Error|timeout/ | sort @timestamp asc'

aws logs get-query-results --query-id <id> --region <region> --output table
```

### RDS
```bash
aws rds describe-db-instances --region <region> --query "DBInstances[?Endpoint.Address=='<endpoint>'].DBInstanceIdentifier" --output text

aws rds describe-db-parameters --db-parameter-group-name <group-name> --region <region> --query "Parameters[?ParameterName=='<param>']" --output table

aws rds modify-db-parameter-group --db-parameter-group-name <group-name> --parameters "ParameterName=<param>,ParameterValue=<value>,ApplyMethod=immediate" --region <region>

aws rds modify-db-instance --db-instance-identifier <instance-id> --cloudwatch-logs-export-configuration '{"EnableLogTypes":["slowquery"]}' --apply-immediately --region <region>

aws rds describe-db-instances --db-instance-identifier <instance-id> --region <region> --query "DBInstances[0].{Enabled:PerformanceInsightsEnabled, RetentionDays:PerformanceInsightsRetentionPeriod}" --output table
```

### EFS
```bash
aws efs describe-access-points --access-point-id <fsap-id> --region <region> --query "AccessPoints[0].FileSystemId" --output text

aws efs describe-file-systems --file-system-id <fs-id> --region <region> --query "FileSystems[0].{ThroughputMode:ThroughputMode, SizeBytes:SizeInBytes.Value}" --output table

aws cloudwatch get-metric-statistics --namespace AWS/EFS --metric-name <BurstCreditBalance|PercentIOLimit|DataReadIOBytes|DataWriteIOBytes|ClientConnections> --dimensions Name=FileSystemId,Value=<fs-id> --start-time <ISO8601> --end-time <ISO8601> --period 300 --statistics Average Minimum Sum --region <region> --output table

aws efs update-file-system --file-system-id <fs-id> --throughput-mode elastic --region <region>
```

### S3
```bash
aws s3api list-objects-v2 --bucket <bucket-name> --prefix "<path-prefix>" --region <region> --query "Contents[].{Key:Key, SizeBytes:Size, LastModified:LastModified}" --output table

aws s3api list-objects-v2 --bucket <bucket-name> --prefix "<path-prefix>" --region <region> --query "length(Contents[])" --output text
```

### CloudWatch Logs housekeeping
```bash
aws logs put-retention-policy --log-group-name "<log-group>" --retention-in-days 30 --region <region>

aws cloudwatch put-metric-alarm --alarm-name "<alarm-name>" --namespace AWS/Lambda --metric-name Errors --dimensions Name=FunctionName,Value=<function-name> --statistic Sum --period 300 --evaluation-periods 1 --threshold 0 --comparison-operator GreaterThanThreshold --alarm-actions <sns-topic-arn> --region <region>
```

---

## Appendix — Full worked timeline: Sept 5, 2026 incident, morning-to-resolution

| Step | Question asked | Tool/Command | Finding |
|---|---|---|---|
| 1 | What's the schedule? | Decoded cron expression manually | 6x/day UTC, uneven gap between 12:00–13:00 slots |
| 2 | Did EventBridge fire correctly? | `get-metric-statistics` on `AWS/Events` (4 metrics) | All 6 slots fired correctly — layer healthy |
| 3 | Did Lambda get invoked and did it error? | `get-metric-statistics` on `AWS/Lambda` (4 metrics) | 18 invocations (6 slots × 3 attempts), all 18 errored, several hit the 900s timeout ceiling |
| 4 | What's the actual error? | `filter-log-events` on the Lambda log group | `Error 2013: Lost connection ... dumping table conversations at row: 27203` |
| 5 | Is a DB timeout setting too aggressive? | `describe-db-parameters` for `wait_timeout`, `net_read_timeout`, etc. | All at engine defaults — ruled out |
| 6 | Was there a DB-side crash logged? | RDS console → Logs & events → Logs panel | No log file existed for the incident date — ruled out a server-side crash |
| 7 | Was gp2 storage burst-throttled? | `get-metric-statistics` on `AWS/RDS BurstBalance` | Stayed at 99% the whole time — ruled out |
| 8 | What was actually running on the DB at the failure moment? | CloudWatch Database Insights, Top SQL / Top waits, narrowed to the exact window | `wait/io/socket/sql/client_connection` dominant, no lock waits — pointed at a client-side stall, not a DB-side lock |
| 9 | Is the Lambda's local storage (EFS) the bottleneck? | Resolved access point → filesystem ID → checked `ThroughputMode` | Confirmed `bursting` mode |
| 10 | Were EFS burst credits exhausted at the failure time? | `get-metric-statistics` on `AWS/EFS BurstCreditBalance`, narrow window | **Flat 0.0 bytes for the entire incident window** — root cause found |
| 11 | Was this a one-off, or does it happen every day? | Same query on a known-good day (Sept 3) for comparison | Sept 3 never dropped below ~36.9 billion bytes — confirmed this was a genuine anomaly, not routine behavior |
| 12 | When did it actually start? | Compared `Duration` metric across Sept 1–6 | Sept 4, 13:00 UTC slot already showed elevated duration — the real starting point, 11 hours before the "reported" incident |
| 13 | Did any backups reach S3 that day? | `list-objects-v2` scoped to the date prefix | Zero files for Sept 5 — confirmed total, not partial, outage |
| 14 | What's the fix? | — | Switch EFS to Elastic Throughput mode; harden Lambda script; fix cron spacing; enable slow query log; add CloudWatch alarms |

**Total investigation time:** roughly half a day, across ~5 layers, using only CLI commands and console navigation — no code changes were needed to reach the root cause; it was entirely a metrics-and-logs forensic exercise.
