# Complete Prometheus Tutorial: Comprehensive Guide

## Table of Contents
1. [Introduction to Prometheus](#introduction-to-prometheus)
2. [What is Prometheus?](#what-is-prometheus)
3. [Why Use Prometheus?](#why-use-prometheus)
4. [Prometheus Architecture](#prometheus-architecture)
5. [Core Components](#core-components)
6. [Installing Prometheus](#installing-prometheus)
7. [Configuration](#configuration)
8. [Prometheus Exporters](#prometheus-exporters)
9. [PromQL - Prometheus Query Language](#promql-prometheus-query-language)
10. [Metric Types](#metric-types)
11. [Alerting with AlertManager](#alerting-with-alertmanager)
12. [Service Discovery](#service-discovery)
13. [Push Gateway](#push-gateway)
14. [Recording Rules](#recording-rules)
15. [Federation](#federation)
16. [Grafana Integration](#grafana-integration)
17. [Advantages of Prometheus](#advantages-of-prometheus)
18. [Disadvantages and Limitations](#disadvantages-and-limitations)
19. [Best Practices](#best-practices)

---

## Introduction to Prometheus

Prometheus is an open-source systems monitoring and alerting toolkit originally built at SoundCloud in 2012. It has since become one of the most popular monitoring solutions, particularly in cloud-native and microservices environments. In 2016, Prometheus joined the Cloud Native Computing Foundation (CNCF) as the second hosted project after Kubernetes.

Prometheus excels at collecting and storing metrics as time series data, making it ideal for understanding system behavior and quickly diagnosing problems during outages.

---

## What is Prometheus?

Prometheus is an open-source monitoring solution written in Go that collects metrics data and stores that data in a time series database. It uses a pull-based model where it periodically scrapes data from HTTP endpoints and pushes that data into a database using a multidimensional model.

**Key Characteristics:**
- Written in Go language
- Pull-based metrics collection
- Built-in time-series database (TSDB)
- Powerful query language (PromQL)
- No dependency on distributed storage
- Autonomous single server nodes
- Service discovery support
- Built-in alerting capabilities

---

## Why Use Prometheus?

### Use Cases

Prometheus is particularly well-suited for:

1. **Cloud-Native Monitoring**: Perfect for Kubernetes and containerized environments
2. **Microservices Architecture**: Handles dynamic service-oriented architectures
3. **Real-time Monitoring**: Designed for quick problem detection and diagnosis
4. **Infrastructure Monitoring**: Monitors servers, databases, and network devices
5. **Application Monitoring**: Tracks application-specific metrics
6. **Performance Monitoring**: Analyzes system performance over time

### When to Use Prometheus

- Recording numeric time series data
- Machine-centric monitoring
- Highly dynamic service-oriented architectures
- Multi-dimensional data collection and querying
- Reliable system diagnostics during outages
- Need for operational independence (no network storage dependencies)

### When NOT to Use Prometheus

- 100% accuracy requirements (billing/financial data)
- Long-term data retention (beyond weeks/months)
- Event logging (use ELK stack instead)
- Detailed request tracing (use Jaeger or Zipkin)

---

## Prometheus Architecture

Prometheus architecture consists of several key components working together:

```
┌─────────────────────────────────────────────────────────────┐
│                     Prometheus Server                        │
│  ┌──────────────────┐  ┌──────────────┐  ┌───────────────┐ │
│  │  Data Retrieval  │→ │ Time Series  │→ │  HTTP Server  │ │
│  │     Worker       │  │   Database   │  │   (PromQL)    │ │
│  └──────────────────┘  └──────────────┘  └───────────────┘ │
└─────────────────────────────────────────────────────────────┘
           ↓ scrape                              ↑ query
    ┌──────────────┐                      ┌─────────────┐
    │  Exporters   │                      │   Grafana   │
    │ - Node       │                      │  Dashboard  │
    │ - MySQL      │                      └─────────────┘
    │ - Redis      │
    │ - Custom     │                      ┌─────────────┐
    └──────────────┘                      │ AlertManager│
           ↑                              └─────────────┘
    ┌──────────────┐                             ↑
    │  Targets     │                             │ alerts
    │ - Apps       │─────────────────────────────┘
    │ - Services   │
    │ - Servers    │
    └──────────────┘
           ↑
    ┌──────────────┐
    │Push Gateway  │← short-lived jobs
    └──────────────┘
           ↑
    ┌──────────────┐
    │Service       │
    │Discovery     │
    │- Kubernetes  │
    │- Consul      │
    │- DNS         │
    └──────────────┘
```

---

## Core Components

### 1. Prometheus Server

The Prometheus server is the core component responsible for:

**Components:**
- **Data Retrieval Worker**: Scrapes metrics from targets via HTTP
- **Time Series Database (TSDB)**: Stores metrics efficiently with compression
- **HTTP Server**: Provides API and web UI for querying data

**Key Features:**
- Scrapes metrics at configurable intervals (default: 15s)
- Stores data locally on disk
- Evaluates recording and alerting rules
- Exposes web UI on port 9090 by default

### 2. Time Series Database (TSDB)

Prometheus includes a highly optimized time-series database:

- **Efficient Storage**: Compression reduces storage requirements
- **Fast Queries**: Optimized for time-range queries
- **Local Storage**: No dependency on distributed storage
- **Retention Policies**: Configurable data retention periods

**Data Model:**
```
metric_name{label1="value1", label2="value2"} value timestamp
```

Example:
```
http_requests_total{method="POST", endpoint="/api/users"} 1234 1698765432
```

### 3. Exporters

Exporters bridge the gap between systems that don't expose Prometheus-compatible metrics and Prometheus itself.

**How Exporters Work:**
1. Collect metrics from the target system
2. Transform data into Prometheus format
3. Expose metrics via `/metrics` HTTP endpoint
4. Prometheus scrapes the exporter

### 4. Service Discovery

Service discovery automatically detects targets to monitor without manual configuration.

**Supported Mechanisms:**
- Kubernetes
- Consul
- DNS (SRV records)
- AWS EC2
- Azure
- Google Cloud Platform (GCP)
- Docker
- File-based discovery

### 5. Push Gateway

The Push Gateway allows short-lived jobs to push metrics to Prometheus:

- Used for batch jobs and cron tasks
- Jobs push metrics to the gateway
- Prometheus scrapes from the gateway
- Metrics persist until explicitly deleted

### 6. AlertManager

AlertManager handles alerts sent by Prometheus:

- **Deduplication**: Prevents duplicate notifications
- **Grouping**: Combines related alerts
- **Routing**: Sends alerts to appropriate receivers
- **Silencing**: Temporarily mutes specific alerts
- **Inhibition**: Suppresses alerts based on other active alerts

---

## Installing Prometheus

### Installation on Linux (Ubuntu/Debian)

#### Step 1: Update System
```bash
sudo apt update && sudo apt upgrade -y
```

#### Step 2: Create Prometheus User
```bash
sudo groupadd --system prometheus
sudo useradd -s /sbin/nologin --system -g prometheus prometheus
```

#### Step 3: Create Directories
```bash
sudo mkdir /etc/prometheus
sudo mkdir /var/lib/prometheus
sudo chown prometheus:prometheus /etc/prometheus
sudo chown prometheus:prometheus /var/lib/prometheus
```

#### Step 4: Download Prometheus
```bash
cd /tmp
wget https://github.com/prometheus/prometheus/releases/download/v2.48.0/prometheus-2.48.0.linux-amd64.tar.gz
tar -xvf prometheus-2.48.0.linux-amd64.tar.gz
cd prometheus-2.48.0.linux-amd64
```

#### Step 5: Install Binaries
```bash
sudo cp prometheus /usr/local/bin/
sudo cp promtool /usr/local/bin/
sudo chown prometheus:prometheus /usr/local/bin/prometheus
sudo chown prometheus:prometheus /usr/local/bin/promtool
```

#### Step 6: Copy Configuration Files
```bash
sudo cp -r consoles /etc/prometheus
sudo cp -r console_libraries /etc/prometheus
sudo chown -R prometheus:prometheus /etc/prometheus/consoles
sudo chown -R prometheus:prometheus /etc/prometheus/console_libraries
```

#### Step 7: Create Configuration File
```bash
sudo nano /etc/prometheus/prometheus.yml
```

Add basic configuration:
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
```

```bash
sudo chown prometheus:prometheus /etc/prometheus/prometheus.yml
```

#### Step 8: Create Systemd Service
```bash
sudo nano /etc/systemd/system/prometheus.service
```

Add the following:
```ini
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
  --config.file /etc/prometheus/prometheus.yml \
  --storage.tsdb.path /var/lib/prometheus/ \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries

[Install]
WantedBy=multi-user.target
```

#### Step 9: Start Prometheus
```bash
sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus
sudo systemctl status prometheus
```

#### Step 10: Configure Firewall
```bash
sudo ufw allow 9090/tcp
```

#### Step 11: Access Web UI
Open browser and navigate to: `http://your-server-ip:9090`

### Installation on Windows

1. Download the Windows binary from [Prometheus Downloads](https://prometheus.io/download/)
2. Extract the ZIP file to `C:\prometheus`
3. Open Command Prompt and navigate to the directory
4. Run: `prometheus.exe --config.file=prometheus.yml`
5. Access UI at `http://localhost:9090`

### Installation on macOS

```bash
# Using Homebrew
brew install prometheus

# Start Prometheus
brew services start prometheus

# Or run manually
prometheus --config.file=/usr/local/etc/prometheus.yml
```

### Docker Installation

```bash
# Pull Prometheus image
docker pull prom/prometheus

# Run Prometheus
docker run -d \
  --name prometheus \
  -p 9090:9090 \
  -v /path/to/prometheus.yml:/etc/prometheus/prometheus.yml \
  prom/prometheus
```

### Kubernetes Installation (Helm)

```bash
# Add Prometheus Helm repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install Prometheus
helm install prometheus prometheus-community/prometheus

# Access Prometheus
kubectl port-forward svc/prometheus-server 9090:80
```

---

## Configuration

### Prometheus Configuration File Structure

The `prometheus.yml` file is the main configuration file:

```yaml
# Global configuration
global:
  scrape_interval: 15s          # How often to scrape targets
  evaluation_interval: 15s       # How often to evaluate rules
  scrape_timeout: 10s           # Timeout for scraping
  external_labels:               # Labels added to any time series
    cluster: 'production'
    region: 'us-east-1'

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - 'alertmanager:9093'

# Rule files
rule_files:
  - 'alerts/*.yml'
  - 'rules/*.yml'

# Scrape configurations
scrape_configs:
  # Job for Prometheus itself
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # Job for Node Exporter
  - job_name: 'node-exporter'
    static_configs:
      - targets: 
          - 'server1:9100'
          - 'server2:9100'
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
```

### Configuration Parameters Explained

#### Global Section
- **scrape_interval**: Default interval between metric scrapes
- **evaluation_interval**: How often to evaluate recording/alerting rules
- **scrape_timeout**: Request timeout for scraping
- **external_labels**: Labels attached to all time series

#### Scrape Configs
```yaml
scrape_configs:
  - job_name: 'my-application'
    scrape_interval: 30s         # Override global interval
    scrape_timeout: 10s
    metrics_path: '/metrics'      # Default: /metrics
    scheme: 'https'              # http or https
    
    static_configs:
      - targets: ['app1:8080', 'app2:8080']
        labels:
          environment: 'production'
          team: 'backend'
    
    # Basic authentication
    basic_auth:
      username: 'prometheus'
      password: 'secret'
    
    # TLS configuration
    tls_config:
      ca_file: '/path/to/ca.pem'
      cert_file: '/path/to/cert.pem'
      key_file: '/path/to/key.pem'
      insecure_skip_verify: false
    
    # Relabeling
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
```

### Complete Example Configuration

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  scrape_timeout: 10s

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['localhost:9093']

rule_files:
  - "rules/*.yml"

scrape_configs:
  # Prometheus itself
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # Node Exporter
  - job_name: 'node-exporter'
    static_configs:
      - targets: 
          - 'node1:9100'
          - 'node2:9100'
          - 'node3:9100'

  # MySQL Exporter
  - job_name: 'mysql'
    static_configs:
      - targets: ['db-server:9104']

  # Redis Exporter
  - job_name: 'redis'
    static_configs:
      - targets: ['redis-server:9121']

  # Application metrics
  - job_name: 'web-app'
    static_configs:
      - targets: ['app1:8080', 'app2:8080']
        labels:
          env: 'production'
```

---

## Prometheus Exporters

### What are Exporters?

Exporters are programs that expose metrics from third-party systems in a format Prometheus can scrape. They act as translators between systems and Prometheus.

### Popular Official Exporters

#### 1. Node Exporter (Linux/Unix Systems)

**Installation:**
```bash
# Download
wget https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz

# Extract
tar -xvf node_exporter-1.7.0.linux-amd64.tar.gz

# Move binary
sudo cp node_exporter-1.7.0.linux-amd64/node_exporter /usr/local/bin/

# Create user
sudo useradd -rs /bin/false node_exporter
```

**Create Service:**
```bash
sudo nano /etc/systemd/system/node_exporter.service
```

```ini
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```

**Start Service:**
```bash
sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
```

**Metrics Exposed:**
- CPU usage and statistics
- Memory usage
- Disk I/O statistics
- Network statistics
- Filesystem usage
- System load

**Configuration in Prometheus:**
```yaml
scrape_configs:
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['localhost:9100']
```

#### 2. MySQL Exporter

**Installation:**
```bash
# Download
wget https://github.com/prometheus/mysqld_exporter/releases/download/v0.15.1/mysqld_exporter-0.15.1.linux-amd64.tar.gz

# Extract and install
tar -xvf mysqld_exporter-0.15.1.linux-amd64.tar.gz
sudo cp mysqld_exporter-0.15.1.linux-amd64/mysqld_exporter /usr/local/bin/
sudo chown root:root /usr/local/bin/mysqld_exporter
```

**Create MySQL User:**
```sql
CREATE USER 'prometheus'@'localhost' IDENTIFIED BY 'password';
GRANT PROCESS, REPLICATION CLIENT ON *.* TO 'prometheus'@'localhost';
GRANT SELECT ON performance_schema.* TO 'prometheus'@'localhost';
FLUSH PRIVILEGES;
```

**Create Configuration File:**
```bash
sudo nano /etc/.mysqld_exporter.cnf
```

```ini
[client]
user=prometheus
password=your_password
host=localhost
```

```bash
sudo chown root:prometheus /etc/.mysqld_exporter.cnf
sudo chmod 640 /etc/.mysqld_exporter.cnf
```

**Create Service:**
```ini
[Unit]
Description=MySQL Exporter
After=network.target

[Service]
Type=simple
User=prometheus
Group=prometheus
ExecStart=/usr/local/bin/mysqld_exporter \
  --config.my-cnf=/etc/.mysqld_exporter.cnf \
  --collect.global_status \
  --collect.info_schema.innodb_metrics \
  --collect.auto_increment.columns \
  --collect.info_schema.processlist \
  --collect.binlog_size \
  --collect.info_schema.tablestats \
  --collect.global_variables \
  --collect.info_schema.query_response_time \
  --collect.info_schema.userstats \
  --collect.info_schema.tables \
  --collect.perf_schema.tablelocks \
  --collect.perf_schema.file_events \
  --collect.perf_schema.eventswaits \
  --collect.perf_schema.indexiowaits \
  --collect.perf_schema.tableiowaits \
  --collect.slave_status \
  --web.listen-address=0.0.0.0:9104

[Install]
WantedBy=multi-user.target
```

**Prometheus Configuration:**
```yaml
scrape_configs:
  - job_name: 'mysql'
    static_configs:
      - targets: ['localhost:9104']
```

**Key Metrics:**
- `mysql_up`: MySQL server status
- `mysql_global_status_connections`: Total connections
- `mysql_global_status_queries`: Total queries
- `mysql_global_status_slow_queries`: Slow queries
- `mysql_global_variables_max_connections`: Max connections

#### 3. Redis Exporter

**Installation:**
```bash
# Download
wget https://github.com/oliver006/redis_exporter/releases/download/v1.55.0/redis_exporter-v1.55.0.linux-amd64.tar.gz

# Extract
tar -xvf redis_exporter-v1.55.0.linux-amd64.tar.gz

# Install
sudo cp redis_exporter-v1.55.0.linux-amd64/redis_exporter /usr/local/bin/
```

**Create Service:**
```ini
[Unit]
Description=Redis Exporter
After=network.target

[Service]
Type=simple
User=redis_exporter
Group=redis_exporter
ExecStart=/usr/local/bin/redis_exporter \
  -redis.addr=redis://localhost:6379 \
  -web.listen-address=:9121

[Install]
WantedBy=multi-user.target
```

**Prometheus Configuration:**
```yaml
scrape_configs:
  - job_name: 'redis'
    static_configs:
      - targets: ['localhost:9121']
```

**Key Metrics:**
- `redis_up`: Redis availability
- `redis_connected_clients`: Connected clients
- `redis_memory_used_bytes`: Memory usage
- `redis_commands_processed_total`: Total commands
- `redis_keyspace_hits_total`: Cache hits

#### 4. PostgreSQL Exporter

**Installation:**
```bash
wget https://github.com/prometheus-community/postgres_exporter/releases/download/v0.15.0/postgres_exporter-0.15.0.linux-amd64.tar.gz
tar -xvf postgres_exporter-0.15.0.linux-amd64.tar.gz
sudo cp postgres_exporter-0.15.0.linux-amd64/postgres_exporter /usr/local/bin/
```

**Environment Configuration:**
```bash
export DATA_SOURCE_NAME="postgresql://prometheus:password@localhost:5432/postgres?sslmode=disable"
./postgres_exporter
```

**Prometheus Configuration:**
```yaml
scrape_configs:
  - job_name: 'postgres'
    static_configs:
      - targets: ['localhost:9187']
```

#### 5. MongoDB Exporter

**Installation:**
```bash
wget https://github.com/percona/mongodb_exporter/releases/download/v0.40.0/mongodb_exporter-0.40.0.linux-amd64.tar.gz
tar -xvf mongodb_exporter-0.40.0.linux-amd64.tar.gz
sudo cp mongodb_exporter /usr/local/bin/
```

**Run:**
```bash
./mongodb_exporter --mongodb.uri=mongodb://localhost:27017
```

**Prometheus Configuration:**
```yaml
scrape_configs:
  - job_name: 'mongodb'
    static_configs:
      - targets: ['localhost:9216']
```

#### 6. Blackbox Exporter (HTTP/HTTPS/DNS/TCP/ICMP Probing)

**Installation:**
```bash
wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.24.0/blackbox_exporter-0.24.0.linux-amd64.tar.gz
tar -xvf blackbox_exporter-0.24.0.linux-amd64.tar.gz
sudo cp blackbox_exporter-0.24.0.linux-amd64/blackbox_exporter /usr/local/bin/
```

**Configuration File (blackbox.yml):**
```yaml
modules:
  http_2xx:
    prober: http
    http:
      preferred_ip_protocol: "ip4"
      valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
      valid_status_codes: []
      method: GET
      fail_if_ssl: false
      fail_if_not_ssl: false
  
  tcp_connect:
    prober: tcp
    tcp:
      preferred_ip_protocol: "ip4"
  
  icmp:
    prober: icmp
    icmp:
      preferred_ip_protocol: "ip4"
```

**Prometheus Configuration:**
```yaml
scrape_configs:
  - job_name: 'blackbox'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - https://example.com
          - https://google.com
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: localhost:9115
```

#### 7. JMX Exporter (Java Applications)

**Download JAR:**
```bash
wget https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.20.0/jmx_prometheus_javaagent-0.20.0.jar
```

**Configuration (config.yaml):**
```yaml
rules:
  - pattern: ".*"
```

**Run Java Application:**
```bash
java -javaagent:./jmx_prometheus_javaagent-0.20.0.jar=8080:config.yaml -jar your-application.jar
```

**Prometheus Configuration:**
```yaml
scrape_configs:
  - job_name: 'java-app'
    static_configs:
      - targets: ['localhost:8080']
```

#### 8. Windows Exporter

**Installation:**
1. Download MSI installer from [GitHub Releases](https://github.com/prometheus-community/windows_exporter/releases)
2. Run installer
3. Service runs on port 9182

**Prometheus Configuration:**
```yaml
scrape_configs:
  - job_name: 'windows'
    static_configs:
      - targets: ['windows-server:9182']
```

### Complete Exporter List

| Exporter | Port | Description |
|----------|------|-------------|
| Node Exporter | 9100 | Linux/Unix system metrics |
| Windows Exporter | 9182 | Windows system metrics |
| MySQL Exporter | 9104 | MySQL database metrics |
| PostgreSQL Exporter | 9187 | PostgreSQL metrics |
| MongoDB Exporter | 9216 | MongoDB metrics |
| Redis Exporter | 9121 | Redis metrics |
| Elasticsearch Exporter | 9114 | Elasticsearch metrics |
| RabbitMQ Exporter | 9419 | RabbitMQ metrics |
| Apache Exporter | 9117 | Apache web server |
| Nginx Exporter | 9113 | Nginx web server |
| HAProxy Exporter | 9101 | HAProxy load balancer |
| Kafka Exporter | 9308 | Kafka metrics |
| Blackbox Exporter | 9115 | Probing endpoints |
| cAdvisor | 8080 | Container metrics |
| JMX Exporter | Custom | Java JMX metrics |

### Creating Custom Exporters

#### Example: Simple Python Exporter

```python
from prometheus_client import start_http_server, Gauge, Counter
import time
import random

# Create metrics
cpu_usage = Gauge('custom_cpu_usage', 'CPU usage percentage')
request_count = Counter('custom_requests_total', 'Total requests', ['method', 'endpoint'])

def collect_metrics():
    """Simulate metric collection"""
    while True:
        # Update gauge
        cpu_usage.set(random.uniform(0, 100))
        
        # Increment counter
        request_count.labels(method='GET', endpoint='/api').inc()
        
        time.sleep(5)

if __name__ == '__main__':
    # Start server on port 8000
    start_http_server(8000)
    print("Custom exporter started on port 8000")
    collect_metrics()
```

**Prometheus Configuration:**
```yaml
scrape_configs:
  - job_name: 'custom-python-exporter'
    static_configs:
      - targets: ['localhost:8000']
```

#### Example: Go Exporter

```go
package main

import (
    "net/http"
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    customMetric = prometheus.NewGauge(prometheus.GaugeOpts{
        Name: "custom_metric_value",
        Help: "A custom metric",
    })
)

func init() {
    prometheus.MustRegister(customMetric)
}

func main() {
    // Update metric
    go func() {
        for {
            customMetric.Set(42.5)
            time.Sleep(10 * time.Second)
        }
    }()
    
    http.Handle("/metrics", promhttp.Handler())
    http.ListenAndServe(":9200", nil)
}
```

---

## PromQL - Prometheus Query Language

PromQL is Prometheus's powerful query language for selecting and aggregating time series data.

### Basic Concepts

#### Data Types

1. **Scalar**: A single numeric value (e.g., `10`, `3.14`)
2. **Instant Vector**: Set of time series with single value per timestamp
3. **Range Vector**: Set of time series with range of values
4. **String**: String value (rarely used)

### Basic Queries

#### Instant Vector Selectors

```promql
# Select all time series for a metric
http_requests_total

# Filter by label
http_requests_total{method="GET"}

# Multiple label filters
http_requests_total{method="GET", status="200"}

# Regex matching
http_requests_total{method=~"GET|POST"}

# Negative regex
http_requests_total{status!~"5.."}
```

#### Range Vector Selectors

```promql
# Last 5 minutes of data
http_requests_total[5m]

# Last 1 hour
http_requests_total[1h]

# Time units: s (seconds), m (minutes), h (hours), d (days), w (weeks), y (years)
```

### Label Matchers

```promql
# Equal
http_requests_total{method="GET"}

# Not equal
http_requests_total{method!="GET"}

# Regex match
http_requests_total{path=~"/api/.*"}

# Negative regex match
http_requests_total{status!~"5.."}
```

### Operators

#### Arithmetic Operators

```promql
# Addition
metric1 + metric2

# Subtraction
metric1 - metric2

# Multiplication
metric1 * metric2

# Division
metric1 / metric2

# Modulo
metric1 % metric2

# Power
metric1 ^ 2
```

#### Comparison Operators

```promql
# Greater than
node_cpu_seconds_total > 100

# Greater than or equal
node_memory_usage >= 80

# Less than
disk_usage < 50

# Less than or equal
response_time <= 0.5

# Equal
status_code == 200

# Not equal
status_code != 500
```

#### Logical Operators

```promql
# AND
metric1 > 10 and metric2 < 100

# OR
metric1 > 100 or metric2 > 100

# UNLESS
metric1 unless metric2
```

### Functions

#### Rate and Increase

```promql
# Per-second rate over 5 minutes
rate(http_requests_total[5m])

# Instant rate (last two samples)
irate(http_requests_total[1m])

# Absolute increase over time range
increase(http_requests_total[1h])
```

#### Aggregation Functions

```promql
# Sum
sum(http_requests_total)

# Sum by label
sum by (method) (http_requests_total)

# Average
avg(node_cpu_usage)

# Average by instance
avg by (instance) (node_cpu_usage)

# Minimum
min(response_time)

# Maximum
max(memory_usage)

# Count
count(up == 1)

# Standard deviation
stddev(response_time)

# Standard variance
stdvar(response_time)

# Count values
count_values("version", app_version)

# Top K
topk(5, http_requests_total)

# Bottom K
bottomk(3, memory_available)

# Quantile
quantile(0.95, response_time)
```

#### Time Functions

```promql
# Time offset
http_requests_total offset 5m

# Time range with offset
rate(http_requests_total[5m] offset 1h)

# Current timestamp
time()

# Day of month
day_of_month()

# Day of week
day_of_week()

# Hour
hour()

# Minute
minute()
```

#### Mathematical Functions

```promql
# Absolute value
abs(temperature_celsius)

# Ceiling
ceil(value)

# Floor
floor(value)

# Round
round(value, to_nearest)

# Square root
sqrt(value)

# Exponential
exp(value)

# Natural logarithm
ln(value)

# Log2
log2(value)

# Log10
log10(value)
```

#### String Functions

```promql
# Label replace
label_replace(up, "new_label", "$1", "instance", "(.*):.*")

# Label join
label_join(up, "address", ",", "instance", "job")
```

#### Sorting Functions

```promql
# Sort ascending
sort(http_requests_total)

# Sort descending
sort_desc(http_requests_total)
```

#### Histogram Functions

```promql
# Calculate quantile from histogram
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

# Calculate quantile by label
histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))
```

### Practical Examples

#### 1. CPU Usage Percentage

```promql
# CPU usage across all cores
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

#### 2. Memory Usage Percentage

```promql
# Memory usage percentage
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100
```

#### 3. Disk Usage Percentage

```promql
# Disk usage by mountpoint
100 - ((node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100)
```

#### 4. Request Rate

```promql
# Requests per second
sum(rate(http_requests_total[5m])) by (method, status)
```

#### 5. Error Rate

```promql
# Error rate percentage
(sum(rate(http_requests_total{status=~"5.."}[5m])) / 
 sum(rate(http_requests_total[5m]))) * 100
```

#### 6. Average Response Time

```promql
# Average response time
rate(http_request_duration_seconds_sum[5m]) / 
rate(http_request_duration_seconds_count[5m])
```

#### 7. 95th Percentile Latency

```promql
# P95 latency
histogram_quantile(0.95, 
  sum by (le) (rate(http_request_duration_seconds_bucket[5m]))
)
```

#### 8. Predict Disk Full

```promql
# Predict when disk will be full (in hours)
predict_linear(node_filesystem_avail_bytes[1h], 4*3600) < 0
```

#### 9. Network Traffic

```promql
# Incoming network traffic (bytes/sec)
rate(node_network_receive_bytes_total[5m])

# Outgoing network traffic (bytes/sec)
rate(node_network_transmit_bytes_total[5m])
```

#### 10. Container CPU Usage

```promql
# Container CPU usage
sum by (container) (rate(container_cpu_usage_seconds_total[5m])) * 100
```

### Query Best Practices

1. **Use rate() for counters**, not irate() for alerts
2. **Always specify time ranges** for range vectors
3. **Use recording rules** for expensive queries
4. **Limit cardinality** by avoiding high-cardinality labels
5. **Use `by` clause** to reduce dimensionality
6. **Test queries** in Prometheus UI before using in production
7. **Use appropriate time ranges** (longer for trends, shorter for real-time)

---

## Metric Types

Prometheus supports four core metric types:

### 1. Counter

A counter is a cumulative metric that only increases (or resets to zero on restart).

**Use Cases:**
- Number of requests
- Number of errors
- Tasks completed
- Bytes sent/received

**Example:**
```go
http_requests_total{method="GET", endpoint="/api/users"} 1234
```

**Client Library Example (Python):**
```python
from prometheus_client import Counter

request_count = Counter('http_requests_total', 'Total HTTP requests', 
                       ['method', 'endpoint'])

# Increment counter
request_count.labels(method='GET', endpoint='/api/users').inc()
```

**PromQL Queries:**
```promql
# Rate of requests per second
rate(http_requests_total[5m])

# Total increase over 1 hour
increase(http_requests_total[1h])
```

**Important Notes:**
- Never use absolute counter values
- Always use rate() or increase() functions
- Counters can reset (e.g., on process restart)

### 2. Gauge

A gauge is a metric that can go up or down.

**Use Cases:**
- Temperature
- Memory usage
- Queue size
- Number of active connections
- CPU usage percentage

**Example:**
```go
node_memory_usage_bytes 8589934592
```

**Client Library Example (Python):**
```python
from prometheus_client import Gauge

memory_usage = Gauge('memory_usage_bytes', 'Memory usage in bytes')

# Set gauge value
memory_usage.set(8589934592)

# Increment
memory_usage.inc(1024)

# Decrement
memory_usage.dec(512)
```

**PromQL Queries:**
```promql
# Current value
node_memory_usage_bytes

# Average over time
avg_over_time(node_memory_usage_bytes[5m])

# Maximum in last hour
max_over_time(node_memory_usage_bytes[1h])
```

### 3. Histogram

A histogram samples observations and counts them in configurable buckets.

**Use Cases:**
- Request duration
- Response size
- Database query times

**Components:**
- `_bucket{le="x"}`: Cumulative counters for buckets
- `_sum`: Sum of all observed values
- `_count`: Total number of observations

**Example:**
```
http_request_duration_seconds_bucket{le="0.1"} 1000
http_request_duration_seconds_bucket{le="0.5"} 1500
http_request_duration_seconds_bucket{le="1.0"} 1800
http_request_duration_seconds_bucket{le="+Inf"} 2000
http_request_duration_seconds_sum 1500.5
http_request_duration_seconds_count 2000
```

**Client Library Example (Python):**
```python
from prometheus_client import Histogram

request_duration = Histogram('http_request_duration_seconds', 
                             'HTTP request duration',
                             buckets=[0.1, 0.5, 1.0, 2.5, 5.0])

# Observe value
with request_duration.time():
    # Process request
    handle_request()

# Or manually
request_duration.observe(0.245)
```

**PromQL Queries:**
```promql
# Calculate 95th percentile
histogram_quantile(0.95, 
  rate(http_request_duration_seconds_bucket[5m])
)

# Average request duration
rate(http_request_duration_seconds_sum[5m]) / 
rate(http_request_duration_seconds_count[5m])

# Request rate
rate(http_request_duration_seconds_count[5m])
```

**Bucket Configuration:**
```python
# Default buckets
[.005, .01, .025, .05, .075, .1, .25, .5, .75, 1.0, 2.5, 5.0, 7.5, 10.0]

# Custom buckets for API latency (milliseconds to seconds)
buckets=[0.001, 0.01, 0.1, 0.5, 1.0, 2.0, 5.0, 10.0]
```

### 4. Summary

A summary is similar to histogram but calculates quantiles on the client side.

**Components:**
- `_sum`: Sum of all observed values
- `_count`: Total number of observations
- `{quantile="x"}`: Pre-calculated quantiles

**Example:**
```
http_request_duration_seconds{quantile="0.5"} 0.123
http_request_duration_seconds{quantile="0.9"} 0.456
http_request_duration_seconds{quantile="0.99"} 0.789
http_request_duration_seconds_sum 987.654
http_request_duration_seconds_count 5000
```

**Client Library Example (Python):**
```python
from prometheus_client import Summary

request_duration = Summary('http_request_duration_seconds',
                          'HTTP request duration')

# Observe value
with request_duration.time():
    handle_request()
```

**PromQL Queries:**
```promql
# Pre-calculated quantiles
http_request_duration_seconds{quantile="0.95"}

# Average
rate(http_request_duration_seconds_sum[5m]) / 
rate(http_request_duration_seconds_count[5m])
```

### Histogram vs Summary

| Feature | Histogram | Summary |
|---------|-----------|---------|
| Quantile calculation | Server-side (PromQL) | Client-side |
| Aggregation | Can aggregate across instances | Cannot aggregate |
| Accuracy | Approximation | Exact (within time window) |
| Resource usage | Lower client CPU | Higher client CPU |
| Flexibility | Can calculate any quantile | Only pre-defined quantiles |
| Use case | Preferred for most cases | When exact quantiles needed |

**Recommendation:** Use histograms unless you need exact quantiles and cannot aggregate.

---

## Alerting with AlertManager

AlertManager handles alerts sent by Prometheus servers. It manages deduplication, grouping, routing, silencing, and inhibition of alerts.

### AlertManager Architecture

```
Prometheus → Alerts → AlertManager → Notifications
                           ↓
                    ┌──────────────────┐
                    │  Grouping        │
                    │  Silencing       │
                    │  Inhibition      │
                    │  Routing         │
                    └──────────────────┘
                           ↓
              ┌────────────┴───────────┐
              ↓                        ↓
         Email/Slack              PagerDuty
```

### Installing AlertManager

```bash
# Download AlertManager
wget https://github.com/prometheus/alertmanager/releases/download/v0.26.0/alertmanager-0.26.0.linux-amd64.tar.gz

# Extract
tar -xvf alertmanager-0.26.0.linux-amd64.tar.gz

# Move binary
sudo cp alertmanager-0.26.0.linux-amd64/alertmanager /usr/local/bin/
sudo cp alertmanager-0.26.0.linux-amd64/amtool /usr/local/bin/

# Create user
sudo useradd -rs /bin/false alertmanager

# Create directories
sudo mkdir /etc/alertmanager
sudo mkdir /var/lib/alertmanager
sudo chown alertmanager:alertmanager /etc/alertmanager
sudo chown alertmanager:alertmanager /var/lib/alertmanager
```

### AlertManager Configuration

**Create `/etc/alertmanager/alertmanager.yml`:**

```yaml
global:
  resolve_timeout: 5m
  
  # SMTP settings for email
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'alerts@example.com'
  smtp_auth_username: 'alerts@example.com'
  smtp_auth_password: 'your-password'
  
  # Slack webhook
  slack_api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'

# Templates
templates:
  - '/etc/alertmanager/templates/*.tmpl'

# Route tree
route:
  receiver: 'default-receiver'
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  
  routes:
    # Critical alerts to PagerDuty
    - match:
        severity: critical
      receiver: pagerduty
      continue: true
    
    # Warning alerts to Slack
    - match:
        severity: warning
      receiver: slack
    
    # Database alerts
    - match:
        team: database
      receiver: database-team
      group_by: ['alertname', 'instance']

# Inhibition rules
inhibit_rules:
  # Inhibit warning if critical is firing
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'instance']
  
  # Inhibit if instance is down
  - source_match:
      alertname: 'InstanceDown'
    target_match_re:
      alertname: '.*'
    equal: ['instance']

# Receivers
receivers:
  - name: 'default-receiver'
    email_configs:
      - to: 'team@example.com'
        subject: '[{{ .Status }}] {{ .CommonLabels.alertname }}'
        html: '{{ template "email.default.html" . }}'
  
  - name: 'slack'
    slack_configs:
      - channel: '#alerts'
        title: '[{{ .Status | toUpper }}] {{ .CommonAnnotations.summary }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
        send_resolved: true
  
  - name: 'pagerduty'
    pagerduty_configs:
      - service_key: 'YOUR_PAGERDUTY_SERVICE_KEY'
        description: '{{ .CommonAnnotations.summary }}'
  
  - name: 'database-team'
    email_configs:
      - to: 'dba-team@example.com'
    slack_configs:
      - channel: '#database-alerts'
```

### Creating AlertManager Service

```bash
sudo nano /etc/systemd/system/alertmanager.service
```

```ini
[Unit]
Description=AlertManager
Wants=network-online.target
After=network-online.target

[Service]
User=alertmanager
Group=alertmanager
Type=simple
ExecStart=/usr/local/bin/alertmanager \
  --config.file=/etc/alertmanager/alertmanager.yml \
  --storage.path=/var/lib/alertmanager/ \
  --web.listen-address=:9093

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable alertmanager
sudo systemctl start alertmanager
```

### Prometheus Alert Rules

Alert rules are defined in YAML files referenced by Prometheus.

**Create `/etc/prometheus/alerts/alerts.yml`:**

```yaml
groups:
  - name: instance_alerts
    interval: 30s
    rules:
      # Instance down
      - alert: InstanceDown
        expr: up == 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Instance {{ $labels.instance }} down"
          description: "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 5 minutes."
      
      # High CPU usage
      - alert: HighCPUUsage
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage on {{ $labels.instance }}"
          description: "CPU usage is {{ $value }}% on {{ $labels.instance }}."
      
      # High memory usage
      - alert: HighMemoryUsage
        expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 85
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage on {{ $labels.instance }}"
          description: "Memory usage is {{ $value }}% on {{ $labels.instance }}."
      
      # Disk space low
      - alert: DiskSpaceLow
        expr: (node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100 < 10
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Low disk space on {{ $labels.instance }}"
          description: "Disk {{ $labels.mountpoint }} on {{ $labels.instance }} has only {{ $value }}% free."
      
      # High error rate
      - alert: HighErrorRate
        expr: (sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))) * 100 > 5
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High HTTP error rate"
          description: "Error rate is {{ $value }}% (threshold: 5%)."
      
      # High request latency
      - alert: HighRequestLatency
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 2
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High request latency"
          description: "95th percentile latency is {{ $value }}s (threshold: 2s)."

  - name: database_alerts
    rules:
      # MySQL down
      - alert: MySQLDown
        expr: mysql_up == 0
        for: 1m
        labels:
          severity: critical
          team: database
        annotations:
          summary: "MySQL instance {{ $labels.instance }} is down"
          description: "MySQL database on {{ $labels.instance }} is not responding."
      
      # High connection usage
      - alert: MySQLHighConnections
        expr: (mysql_global_status_threads_connected / mysql_global_variables_max_connections) * 100 > 80
        for: 5m
        labels:
          severity: warning
          team: database
        annotations:
          summary: "High MySQL connection usage on {{ $labels.instance }}"
          description: "Connection usage is {{ $value }}% on {{ $labels.instance }}."
      
      # Slow queries
      - alert: MySQLSlowQueries
        expr: rate(mysql_global_status_slow_queries[5m]) > 10
        for: 10m
        labels:
          severity: warning
          team: database
        annotations:
          summary: "High rate of slow queries on {{ $labels.instance }}"
          description: "{{ $value }} slow queries per second on {{ $labels.instance }}."
```

### Configure Prometheus to Use Alerts

**Update `/etc/prometheus/prometheus.yml`:**

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

# AlertManager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - 'localhost:9093'

# Load alert rules
rule_files:
  - 'alerts/*.yml'
  - 'rules/*.yml'

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
```

### Alert Rule Syntax

```yaml
groups:
  - name: <group_name>
    interval: <evaluation_interval>
    rules:
      - alert: <alert_name>
        expr: <promql_expression>
        for: <duration>
        labels:
          <label_key>: <label_value>
        annotations:
          <annotation_key>: <annotation_value>
```

**Components:**
- **alert**: Name of the alert (must be valid label value)
- **expr**: PromQL expression that triggers the alert
- **for**: How long condition must be true before firing
- **labels**: Additional labels attached to alert
- **annotations**: Information about the alert (summary, description)

### Grouping

Grouping consolidates similar alerts into single notifications.

```yaml
route:
  group_by: ['alertname', 'cluster']
  group_wait: 30s        # Wait before sending first notification
  group_interval: 5m     # Wait before sending notification about new alerts
  repeat_interval: 4h    # Wait before resending
```

**Example:**
If 10 instances go down:
- Without grouping: 10 separate notifications
- With grouping by `alertname`: 1 notification with 10 instances

### Inhibition

Inhibition suppresses alerts when other alerts are already firing.

```yaml
inhibit_rules:
  # Don't alert on service down if entire instance is down
  - source_match:
      alertname: 'InstanceDown'
    target_match:
      alertname: 'ServiceDown'
    equal: ['instance']
  
  # Suppress warnings if critical alert is active
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'instance']
```

### Silencing

Silencing temporarily mutes specific alerts.

**Via Web UI:**
1. Navigate to `http://alertmanager:9093`
2. Click "New Silence"
3. Add matchers: `alertname="HighCPUUsage"`
4. Set duration
5. Add comment
6. Create

**Via CLI (amtool):**
```bash
# Create silence
amtool silence add \
  alertname="HighCPUUsage" \
  instance="server1:9100" \
  --duration=2h \
  --comment="Planned maintenance"

# List silences
amtool silence query

# Expire silence
amtool silence expire <silence-id>
```

### Routing Examples

#### Route by Severity

```yaml
route:
  receiver: 'default'
  routes:
    - match:
        severity: 'critical'
      receiver: 'pagerduty'
    - match:
        severity: 'warning'
      receiver: 'slack'
    - match:
        severity: 'info'
      receiver: 'email'
```

#### Route by Team

```yaml
route:
  receiver: 'default'
  routes:
    - match:
        team: 'frontend'
      receiver: 'frontend-slack'
    - match:
        team: 'backend'
      receiver: 'backend-slack'
    - match:
        team: 'database'
      receiver: 'dba-pagerduty'
```

#### Complex Routing

```yaml
route:
  receiver: 'default'
  group_by: ['alertname']
  routes:
    # Critical production alerts
    - match:
        severity: 'critical'
        environment: 'production'
      receiver: 'oncall-pagerduty'
      group_wait: 10s
      repeat_interval: 1h
    
    # Development environment
    - match:
        environment: 'development'
      receiver: 'dev-slack'
      group_wait: 5m
      repeat_interval: 24h
```

### Receiver Examples

#### Email

```yaml
receivers:
  - name: 'email-team'
    email_configs:
      - to: 'team@example.com'
        from: 'alerts@example.com'
        smarthost: 'smtp.gmail.com:587'
        auth_username: 'alerts@example.com'
        auth_password: 'password'
        subject: '[{{ .Status }}] {{ .CommonLabels.alertname }}'
        html: |
          <h2>Alert: {{ .CommonLabels.alertname }}</h2>
          <p><strong>Status:</strong> {{ .Status }}</p>
          {{ range .Alerts }}
          <p><strong>Instance:</strong> {{ .Labels.instance }}</p>
          <p><strong>Description:</strong> {{ .Annotations.description }}</p>
          {{ end }}
```

#### Slack

```yaml
receivers:
  - name: 'slack-alerts'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/YOUR/WEBHOOK/URL'
        channel: '#alerts'
        username: 'Prometheus'
        icon_emoji: ':fire:'
        title: '[{{ .Status | toUpper }}] {{ .CommonLabels.alertname }}'
        text: |
          {{ range .Alerts }}
          *Alert:* {{ .Labels.alertname }}
          *Instance:* {{ .Labels.instance }}
          *Severity:* {{ .Labels.severity }}
          *Description:* {{ .Annotations.description }}
          {{ end }}
        send_resolved: true
```

#### PagerDuty

```yaml
receivers:
  - name: 'pagerduty-critical'
    pagerduty_configs:
      - service_key: 'YOUR_SERVICE_KEY'
        description: '{{ .CommonAnnotations.summary }}'
        details:
          firing: '{{ .Alerts.Firing | len }}'
          resolved: '{{ .Alerts.Resolved | len }}'
```

#### Webhook

```yaml
receivers:
  - name: 'custom-webhook'
    webhook_configs:
      - url: 'http://your-service/webhook'
        send_resolved: true
        http_config:
          basic_auth:
            username: 'user'
            password: 'password'
```

### Testing Alerts

#### 1. Manual Test

```bash
# Send test alert to AlertManager
curl -H "Content-Type: application/json" -d '[
  {
    "labels": {
      "alertname": "TestAlert",
      "severity": "critical",
      "instance": "test-instance"
    },
    "annotations": {
      "summary": "This is a test alert",
      "description": "Testing AlertManager configuration"
    }
  }
]' http://localhost:9093/api/v1/alerts
```

#### 2. Create Test Alert Rule

```yaml
- alert: TestAlert
  expr: vector(1)
  for: 1m
  labels:
    severity: warning
  annotations:
    summary: "Test alert"
    description: "This is a test alert that always fires"
```

### Alert Best Practices

1. **Use meaningful alert names**: Clear and descriptive
2. **Set appropriate thresholds**: Avoid alert fatigue
3. **Use `for` clause**: Prevent flapping alerts
4. **Add context in annotations**: Include runbook links
5. **Group related alerts**: Reduce notification spam
6. **Use inhibition rules**: Suppress redundant alerts
7. **Test alerts regularly**: Ensure proper routing
8. **Document alerts**: Explain why alert exists and how to resolve
9. **Set appropriate severity**: critical/warning/info
10. **Use templating**: Make alerts informative

---

## Service Discovery

Service Discovery allows Prometheus to automatically discover targets without manual configuration.

### Why Service Discovery?

In dynamic environments (Kubernetes, cloud), targets constantly change:
- Containers are created/destroyed
- Auto-scaling adds/removes instances
- Services get new IPs

Manual configuration is:
- Time-consuming
- Error-prone
- Doesn't scale

### Service Discovery Mechanisms

#### 1. Kubernetes Service Discovery

**Configuration:**

```yaml
scrape_configs:
  # Discover Kubernetes nodes
  - job_name: 'kubernetes-nodes'
    kubernetes_sd_configs:
      - role: node
    relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - target_label: __address__
        replacement: kubernetes.default.svc:443
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)
        target_label: __metrics_path__
        replacement: /api/v1/nodes/${1}/proxy/metrics

  # Discover Kubernetes pods
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: kubernetes_pod_name

  # Discover Kubernetes services
  - job_name: 'kubernetes-services'
    kubernetes_sd_configs:
      - role: service
    metrics_path: /probe
    params:
      module: [http_2xx]
    relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_probe]
        action: keep
        regex: true
      - source_labels: [__address__]
        target_label: __param_target
      - target_label: __address__
        replacement: blackbox-exporter:9115
      - source_labels: [__param_target]
        target_label: instance

  # Discover Kubernetes endpoints
  - job_name: 'kubernetes-endpoints'
    kubernetes_sd_configs:
      - role: endpoints
    relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
        action: replace
        target_label: __scheme__
        regex: (https?)
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
```

**Pod Annotations for Discovery:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/metrics"
spec:
  containers:
    - name: app
      image: my-app:latest
      ports:
        - containerPort: 8080
```

**Service Annotations:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
    prometheus.io/path: "/metrics"
spec:
  selector:
    app: my-app
  ports:
    - port: 9090
      targetPort: 8080
```

#### 2. Consul Service Discovery

**Install Consul:**
```bash
wget https://releases.hashicorp.com/consul/1.17.0/consul_1.17.0_linux_amd64.zip
unzip consul_1.17.0_linux_amd64.zip
sudo mv consul /usr/local/bin/
```

**Start Consul:**
```bash
consul agent -dev
```

**Register Service in Consul:**

```bash
curl -X PUT -d '{
  "ID": "node-exporter-1",
  "Name": "node-exporter",
  "Tags": ["prometheus"],
  "Address": "192.168.1.10",
  "Port": 9100,
  "Check": {
    "HTTP": "http://192.168.1.10:9100/metrics",
    "Interval": "10s"
  }
}' http://localhost:8500/v1/agent/service/register
```

**Prometheus Configuration:**

```yaml
scrape_configs:
  - job_name: 'consul'
    consul_sd_configs:
      - server: 'localhost:8500'
        services: []
    relabel_configs:
      - source_labels: [__meta_consul_tags]
        regex: .*,prometheus,.*
        action: keep
      - source_labels: [__meta_consul_service]
        target_label: job
      - source_labels: [__meta_consul_node]
        target_label: instance
```

**Service Definition File (`node-exporter.json`):**

```json
{
  "service": {
    "name": "node-exporter",
    "tags": ["prometheus", "monitoring"],
    "port": 9100,
    "check": {
      "http": "http://localhost:9100/metrics",
      "interval": "10s"
    }
  }
}
```

#### 3. DNS Service Discovery

**Configuration:**

```yaml
scrape_configs:
  - job_name: 'dns-discovery'
    dns_sd_configs:
      - names:
          - '_prometheus._tcp.example.com'
        type: 'SRV'
        refresh_interval: 30s
```

**DNS SRV Records:**

```bash
# Create SRV record
_prometheus._tcp.example.com. 3600 IN SRV 0 0 9100 node1.example.com.
_prometheus._tcp.example.com. 3600 IN SRV 0 0 9100 node2.example.com.
```

#### 4. File-Based Service Discovery

**Configuration:**

```yaml
scrape_configs:
  - job_name: 'file-discovery'
    file_sd_configs:
      - files:
          - '/etc/prometheus/targets/*.json'
          - '/etc/prometheus/targets/*.yml'
        refresh_interval: 30s
```

**Target File (`/etc/prometheus/targets/nodes.json`):**

```json
[
  {
    "targets": ["node1:9100", "node2:9100", "node3:9100"],
    "labels": {
      "env": "production",
      "team": "infrastructure"
    }
  },
  {
    "targets": ["node4:9100", "node5:9100"],
    "labels": {
      "env": "staging",
      "team": "infrastructure"
    }
  }
]
```

**YAML Format (`targets.yml`):**

```yaml
- targets:
    - 'node1:9100'
    - 'node2:9100'
  labels:
    env: 'production'
    datacenter: 'dc1'

- targets:
    - 'node3:9100'
  labels:
    env: 'staging'
    datacenter: 'dc2'
```

#### 5. AWS EC2 Service Discovery

**Configuration:**

```yaml
scrape_configs:
  - job_name: 'aws-ec2'
    ec2_sd_configs:
      - region: us-east-1
        access_key: YOUR_ACCESS_KEY
        secret_key: YOUR_SECRET_KEY
        port: 9100
        filters:
          - name: tag:Monitoring
            values: [enabled]
          - name: instance-state-name
            values: [running]
    relabel_configs:
      - source_labels: [__meta_ec2_tag_Name]
        target_label: instance_name
      - source_labels: [__meta_ec2_tag_Environment]
        target_label: environment
      - source_labels: [__meta_ec2_private_ip]
        target_label: private_ip
```

#### 6. Azure Service Discovery

```yaml
scrape_configs:
  - job_name: 'azure-vms'
    azure_sd_configs:
      - subscription_id: YOUR_SUBSCRIPTION_ID
        tenant_id: YOUR_TENANT_ID
        client_id: YOUR_CLIENT_ID
        client_secret: YOUR_CLIENT_SECRET
        port: 9100
```

#### 7. GCP Service Discovery

```yaml
scrape_configs:
  - job_name: 'gce-instances'
    gce_sd_configs:
      - project: your-project-id
        zone: us-central1-a
        port: 9100
        filter: 'labels.monitoring:enabled'
```

#### 8. Docker Service Discovery

```yaml
scrape_configs:
  - job_name: 'docker'
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
    relabel_configs:
      - source_labels: [__meta_docker_container_name]
        regex: /(.*)
        target_label: container_name
      - source_labels: [__meta_docker_container_label_com_docker_compose_service]
        target_label: service
```

### Relabeling

Relabeling is powerful for transforming discovered targets.

**Common Relabel Actions:**

```yaml
relabel_configs:
  # Keep only targets with specific label
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
    action: keep
    regex: true

  # Drop targets
  - source_labels: [__meta_consul_tags]
    regex: .*,private,.*
    action: drop

  # Replace label
  - source_labels: [__meta_kubernetes_namespace]
    action: replace
    target_label: namespace

  # Label map (copy all meta labels)
  - action: labelmap
    regex: __meta_kubernetes_pod_label_(.+)

  # Combine multiple labels
  - source_labels: [__meta_consul_dc, __meta_consul_service]
    separator: '/'
    target_label: service_location

  # Hash modulus (for sharding)
  - source_labels: [__address__]
    modulus: 4
    target_label: __tmp_hash
    action: hashmod
  - source_labels: [__tmp_hash]
    regex: ^1$
    action: keep
```

### Best Practices

1. **Use service discovery** instead of static configs in dynamic environments
2. **Filter early** with relabel_configs to reduce scraping load
3. **Add meaningful labels** for better organization
4. **Use annotations** in Kubernetes for per-pod configuration
5. **Implement health checks** in service registries
6. **Monitor service discovery** itself for errors
7. **Use file-based discovery** for local/development environments
8. **Set appropriate refresh intervals** (balance between staleness and load)

---

## Push Gateway

The Push Gateway is an intermediary service for short-lived jobs to push metrics to Prometheus.

### When to Use Push Gateway

**Use Cases:**
- Batch jobs
- Cron jobs
- Serverless functions
- CI/CD pipelines
- Scripts and scheduled tasks
- Jobs that complete before Prometheus can scrape

**When NOT to use:**
- Long-running applications (use direct instrumentation)
- High-frequency updates (can overwhelm Push Gateway)
- When pull model works fine

### Installing Push Gateway

```bash
# Download
wget https://github.com/prometheus/pushgateway/releases/download/v1.6.2/pushgateway-1.6.2.linux-amd64.tar.gz

# Extract
tar -xvf pushgateway-1.6.2.linux-amd64.tar.gz

# Install
sudo cp pushgateway-1.6.2.linux-amd64/pushgateway /usr/local/bin/

# Create user
sudo useradd -rs /bin/false pushgateway
```

**Create Service:**

```ini
[Unit]
Description=Prometheus Pushgateway
Wants=network-online.target
After=network-online.target

[Service]
User=pushgateway
Group=pushgateway
Type=simple
ExecStart=/usr/local/bin/pushgateway \
  --web.listen-address=:9091 \
  --web.telemetry-path=/metrics \
  --persistence.file=/var/lib/pushgateway/metrics.db \
  --persistence.interval=5m

[Install]
WantedBy=multi-user.target
```

```bash
sudo mkdir -p /var/lib/pushgateway
sudo chown pushgateway:pushgateway /var/lib/pushgateway
sudo systemctl daemon-reload
sudo systemctl enable pushgateway
sudo systemctl start pushgateway
```

### Configure Prometheus

```yaml
scrape_configs:
  - job_name: 'pushgateway'
    honor_labels: true
    static_configs:
      - targets: ['localhost:9091']
```

**Important:** `honor_labels: true` preserves job labels from pushed metrics.

### Pushing Metrics

#### Using cURL

**Basic Push:**

```bash
# Push single metric
echo "some_metric 42" | curl --data-binary @- http://localhost:9091/metrics/job/batch_job

# Push multiple metrics
cat <<EOF | curl --data-binary @- http://localhost:9091/metrics/job/batch_job
# TYPE batch_duration_seconds gauge
batch_duration_seconds 123.45
# TYPE batch_records_processed counter
batch_records_processed 1000
EOF
```

**With Instance Label:**

```bash
cat <<EOF | curl --data-binary @- http://localhost:9091/metrics/job/batch_job/instance/server1
batch_duration_seconds 123.45
batch_records_processed{status="success"} 950
batch_records_processed{status="failed"} 50
EOF
```

**With Additional Labels:**

```bash
curl --data-binary @- http://localhost:9091/metrics/job/backup/environment/production/type/database <<EOF
backup_duration_seconds 3600
backup_size_bytes 10737418240
EOF
```

#### Using Python

```python
from prometheus_client import CollectorRegistry, Gauge, push_to_gateway

# Create registry
registry = CollectorRegistry()

# Create metrics
duration = Gauge('job_duration_seconds', 'Job duration in seconds', registry=registry)
records = Gauge('records_processed', 'Number of records processed', registry=registry)

# Set values
duration.set(123.45)
records.set(1000)

# Push to gateway
push_to_gateway('localhost:9091', job='batch_job', registry=registry)

# With grouping key (instance label)
push_to_gateway('localhost:9091', job='batch_job', 
                registry=registry, grouping_key={'instance': 'server1'})
```

**Full Python Example:**

```python
#!/usr/bin/env python3
import time
import random
from prometheus_client import CollectorRegistry, Gauge, Counter, push_to_gateway

def run_batch_job():
    # Create registry
    registry = CollectorRegistry()
    
    # Define metrics
    duration = Gauge('batch_job_duration_seconds', 
                     'Time spent processing batch job',
                     registry=registry)
    
    records_processed = Counter('batch_records_processed_total',
                               'Total records processed',
                               ['status'],
                               registry=registry)
    
    last_success = Gauge('batch_job_last_success_timestamp',
                        'Timestamp of last successful run',
                        registry=registry)
    
    # Simulate batch job
    start_time = time.time()
    
    # Process records
    success_count = random.randint(900, 1000)
    error_count = random.randint(0, 100)
    
    records_processed.labels(status='success').inc(success_count)
    records_processed.labels(status='error').inc(error_count)
    
    # Calculate duration
    duration_seconds = time.time() - start_time
    duration.set(duration_seconds)
    
    # Update last success timestamp
    last_success.set(time.time())
    
    # Push to gateway
    try:
        push_to_gateway('localhost:9091', 
                       job='batch_processing',
                       registry=registry,
                       grouping_key={'instance': 'prod-batch-server'})
        print(f"Metrics pushed successfully. Duration: {duration_seconds:.2f}s")
    except Exception as e:
        print(f"Failed to push metrics: {e}")

if __name__ == '__main__':
    run_batch_job()
```

#### Using Shell Script

```bash
#!/bin/bash

JOB_NAME="backup_job"
INSTANCE="backup-server-01"
PUSHGATEWAY="localhost:9091"

# Start time
START_TIME=$(date +%s)

# Run backup
echo "Starting backup..."
# Your backup commands here
sleep 10  # Simulating backup

# Calculate duration
END_TIME=$(date +%s)
DURATION=$((END_TIME - START_TIME))

# Get backup size (example)
BACKUP_SIZE=$(du -sb /backup/latest.tar.gz | cut -f1)

# Push metrics
cat <<EOF | curl --data-binary @- http://${PUSHGATEWAY}/metrics/job/${JOB_NAME}/instance/${INSTANCE}
# TYPE backup_duration_seconds gauge
backup_duration_seconds ${DURATION}
# TYPE backup_size_bytes gauge
backup_size_bytes ${BACKUP_SIZE}
# TYPE backup_last_success_timestamp gauge
backup_last_success_timestamp ${END_TIME}
# TYPE backup_status gauge
backup_status 1
EOF

echo "Backup completed in ${DURATION} seconds"
```

#### Using Go

```go
package main

import (
    "fmt"
    "time"
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/push"
)

func main() {
    // Create metrics
    duration := prometheus.NewGauge(prometheus.GaugeOpts{
        Name: "job_duration_seconds",
        Help: "Time spent on job",
    })
    
    recordsProcessed := prometheus.NewCounter(prometheus.CounterOpts{
        Name: "records_processed_total",
        Help: "Total records processed",
    })
    
    // Simulate job
    start := time.Now()
    
    // Do work
    time.Sleep(2 * time.Second)
    recordsProcessed.Add(1000)
    
    // Set duration
    duration.Set(time.Since(start).Seconds())
    
    // Push to gateway
    if err := push.New("http://localhost:9091", "batch_job").
        Collector(duration).
        Collector(recordsProcessed).
        Grouping("instance", "server1").
        Push(); err != nil {
        fmt.Println("Could not push to Pushgateway:", err)
    }
}
```

### Deleting Metrics

**Delete all metrics for a job:**

```bash
curl -X DELETE http://localhost:9091/metrics/job/batch_job
```

**Delete metrics for specific instance:**

```bash
curl -X DELETE http://localhost:9091/metrics/job/batch_job/instance/server1
```

### Cron Job Example

```bash
# Add to crontab
# Run backup every night at 2 AM
0 2 * * * /usr/local/bin/backup.sh && /usr/local/bin/push_backup_metrics.sh
```

**push_backup_metrics.sh:**

```bash
#!/bin/bash

PUSHGATEWAY="localhost:9091"
JOB="nightly_backup"

# Check if backup succeeded
if [ -f /var/log/backup_success ]; then
    STATUS=1
    MESSAGE="Backup completed successfully"
else
    STATUS=0
    MESSAGE="Backup failed"
fi

# Push metrics
cat <<EOF | curl --data-binary @- http://${PUSHGATEWAY}/metrics/job/${JOB}
# TYPE backup_status gauge
# HELP backup_status Backup job status (1=success, 0=failed)
backup_status ${STATUS}
# TYPE backup_last_run_timestamp gauge
# HELP backup_last_run_timestamp Timestamp of last backup run
backup_last_run_timestamp $(date +%s)
EOF

echo "${MESSAGE}"
```

### Monitoring Push Gateway

```promql
# Number of stored time series
pushgateway_metrics_count

# HTTP requests to Push Gateway
rate(pushgateway_http_requests_total[5m])

# Push failures
rate(pushgateway_http_request_duration_seconds_count{code!="200"}[5m])
```

### Best Practices

1. **Always set job label** to identify metric source
2. **Use instance label** for multiple sources of same job
3. **Delete metrics after job completion** if they're not cumulative
4. **Don't push too frequently** (Push Gateway is not for high-frequency metrics)
5. **Include timestamp** of last successful run
6. **Monitor Push Gateway** itself for availability
7. **Use persistence** to survive restarts
8. **Don't use for service monitoring** (use direct instrumentation)

### Common Patterns

#### Success/Failure Tracking

```bash
# On success
cat <<EOF | curl --data-binary @- http://localhost:9091/metrics/job/my_job
job_last_success_timestamp $(date +%s)
job_status 1
EOF

# On failure
cat <<EOF | curl --data-binary @- http://localhost:9091/metrics/job/my_job
job_last_failure_timestamp $(date +%s)
job_status 0
EOF
```

#### Alert on Stale Jobs

```yaml
- alert: BatchJobStale
  expr: time() - job_last_success_timestamp > 86400
  for: 1h
  labels:
    severity: warning
  annotations:
    summary: "Batch job hasn't run in 24 hours"
    description: "Job {{ $labels.job }} last succeeded {{ $value }} seconds ago."
```

---

## Recording Rules

Recording rules precompute frequently used or computationally expensive queries and save results as new time series.

### Why Use Recording Rules?

**Benefits:**
- Faster dashboard loading
- Reduced query load on Prometheus
- Simplified complex queries
- Consistent metric naming
- Enable aggregation across time

**Use Cases:**
- Complex aggregations used in multiple dashboards
- High-cardinality queries
- Rate calculations used frequently
- Multi-level aggregations

### Configuration

**Create `/etc/prometheus/rules/recording_rules.yml`:**

```yaml
groups:
  - name: node_recording_rules
    interval: 30s
    rules:
      # CPU usage percentage
      - record: instance:node_cpu_utilization:rate5m
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
      
      # Memory usage percentage
      - record: instance:node_memory_utilization:ratio
        expr: 1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)
      
      # Disk usage percentage by device
      - record: instance:node_disk_utilization:ratio
        expr: 1 - (node_filesystem_avail_bytes / node_filesystem_size_bytes)
      
      # Network receive rate
      - record: instance:node_network_receive:rate5m
        expr: rate(node_network_receive_bytes_total[5m])
      
      # Network transmit rate
      - record: instance:node_network_transmit:rate5m
        expr: rate(node_network_transmit_bytes_total[5m])

  - name: application_recording_rules
    interval: 15s
    rules:
      # Request rate by method and status
      - record: job:http_requests:rate5m
        expr: sum by(job, method, status) (rate(http_requests_total[5m]))
      
      # Overall request rate
      - record: job:http_requests:rate5m:sum
        expr: sum by(job) (rate(http_requests_total[5m]))
      
      # Error rate percentage
      - record: job:http_requests:error_rate5m
        expr: |
          sum by(job) (rate(http_requests_total{status=~"5.."}[5m]))
          /
          sum by(job) (rate(http_requests_total[5m]))
          * 100
      
      # Average request duration
      - record: job:http_request_duration:avg
        expr: |
          rate(http_request_duration_seconds_sum[5m])
          /
          rate(http_request_duration_seconds_count[5m])
      
      # 95th percentile latency
      - record: job:http_request_duration:p95
        expr: histogram_quantile(0.95, sum by(job, le) (rate(http_request_duration_seconds_bucket[5m])))
      
      # 99th percentile latency
      - record: job:http_request_duration:p99
        expr: histogram_quantile(0.99, sum by(job, le) (rate(http_request_duration_seconds_bucket[5m])))

  - name: database_recording_rules
    rules:
      # MySQL connection usage percentage
      - record: instance:mysql_connection_usage:ratio
        expr: mysql_global_status_threads_connected / mysql_global_variables_max_connections
      
      # Query rate
      - record: instance:mysql_queries:rate5m
        expr: rate(mysql_global_status_queries[5m])
      
      # Slow query rate
      - record: instance:mysql_slow_queries:rate5m
        expr: rate(mysql_global_status_slow_queries[5m])

  - name: aggregation_rules
    interval: 1m
    rules:
      # Cluster-wide CPU usage
      - record: cluster:cpu_utilization:avg
        expr: avg(instance:node_cpu_utilization:rate5m)
      
      # Cluster-wide memory usage
      - record: cluster:memory_utilization:avg
        expr: avg(instance:node_memory_utilization:ratio)
      
      # Per-datacenter aggregations
      - record: datacenter:cpu_utilization:avg
        expr: avg by(datacenter) (instance:node_cpu_utilization:rate5m)
```

### Naming Conventions

**Format:** `level:metric:operations`

**Components:**
- **level**: Aggregation level (instance, job, cluster)
- **metric**: Base metric name
- **operations**: Operations applied (rate5m, sum, avg, ratio)

**Examples:**
```
instance:node_cpu_utilization:rate5m
job:http_requests:rate5m:sum
cluster:cpu:avg
datacenter:memory:avg
```

### Multi-Level Aggregation

```yaml
groups:
  - name: hierarchical_rules
    rules:
      # Level 1: Instance metrics
      - record: instance:cpu:usage
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
      
      # Level 2: Job aggregation
      - record: job:cpu:usage:avg
        expr: avg by(job) (instance:cpu:usage)
      
      # Level 3: Cluster aggregation
      - record: cluster:cpu:usage:avg
        expr: avg(job:cpu:usage:avg)
      
      # Per-datacenter
      - record: datacenter:cpu:usage:avg
        expr: avg by(datacenter) (instance:cpu:usage)
```

### Update Prometheus Configuration

```yaml
global:
  evaluation_interval: 15s

rule_files:
  - 'rules/recording_rules.yml'
  - 'rules/alerting_rules.yml'
```

### Testing Recording Rules

```bash
# Check syntax
promtool check rules /etc/prometheus/rules/recording_rules.yml

# Test specific rule
promtool test rules /etc/prometheus/rules/test_recording_rules.yml
```

### Example: Complete Setup

**recording_rules.yml:**

```yaml
groups:
  - name: api_metrics
    interval: 30s
    rules:
      # Request rate per endpoint
      - record: endpoint:http_requests:rate1m
        expr: sum by(endpoint, method) (rate(http_requests_total[1m]))
      
      # Request rate per service
      - record: service:http_requests:rate1m
        expr: sum by(service) (rate(http_requests_total[1m]))
      
      # Error rate per endpoint
      - record: endpoint:http_errors:rate1m
        expr: |
          sum by(endpoint) (rate(http_requests_total{status=~"5.."}[1m]))
          /
          sum by(endpoint) (rate(http_requests_total[1m]))
      
      # Latency percentiles per endpoint
      - record: endpoint:http_duration:p50
        expr: histogram_quantile(0.50, sum by(endpoint, le) (rate(http_request_duration_seconds_bucket[5m])))
      
      - record: endpoint:http_duration:p95
        expr: histogram_quantile(0.95, sum by(endpoint, le) (rate(http_request_duration_seconds_bucket[5m])))
      
      - record: endpoint:http_duration:p99
        expr: histogram_quantile(0.99, sum by(endpoint, le) (rate(http_request_duration_seconds_bucket[5m])))
      
      # Average latency
      - record: endpoint:http_duration:avg
        expr: |
          rate(http_request_duration_seconds_sum[5m])
          /
          rate(http_request_duration_seconds_count[5m])
```

### Using Recording Rules in Queries

**Before (expensive query):**
```promql
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

**After (simple query):**
```promql
instance:node_cpu_utilization:rate5m
```

### Best Practices

1. **Use consistent naming** following `level:metric:operations` format
2. **Document rules** with comments
3. **Start with long intervals** (30s-1m), optimize if needed
4. **Test rules** before deploying to production
5. **Monitor rule evaluation time** with `prometheus_rule_evaluation_duration_seconds`
6. **Don't over-aggregate** - maintain useful labels
7. **Use recording rules** for dashboard queries, not alerts
8. **Create hierarchical rules** for multi-level aggregations
9. **Keep rules in version control**
10. **Review and clean up** unused rules regularly

---

## Federation

Federation allows a Prometheus server to scrape time series from another Prometheus server.

### Use Cases

**1. Hierarchical Federation:**
- Global Prometheus aggregates from multiple regional Prometheus
- Scalable monitoring for large infrastructures
- Multi-datacenter monitoring

**2. Cross-Service Federation:**
- One service's Prometheus pulls metrics from another
- Correlate metrics across different teams/services

### Architecture Patterns

#### Hierarchical Federation

```
        Global Prometheus
              ↑
      ┌───────┴────────┐
      ↑                ↑
  Region 1         Region 2
  Prometheus       Prometheus
      ↑                ↑
  ┌───┴───┐        ┌───┴───┐
  ↑       ↑        ↑       ↑
DC1-A   DC1-B    DC2-A   DC2-B
Prom    Prom     Prom    Prom
```

#### Cross-Service Federation

```
Service A          Service B
Prometheus ←───→ Prometheus
    ↓                 ↓
Service A         Service B
Metrics          Metrics
```

### Configuration

#### Setup 1: Basic Federation

**Datacenter Prometheus (DC1):**

```yaml
# /etc/prometheus/prometheus-dc1.yml
global:
  scrape_interval: 15s
  external_labels:
    datacenter: 'dc1'
    cluster: 'prod'

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node-exporter'
    static_configs:
      - targets:
          - 'node1:9100'
          - 'node2:9100'
          - 'node3:9100'
```

**Datacenter Prometheus (DC2):**

```yaml
# /etc/prometheus/prometheus-dc2.yml
global:
  scrape_interval: 15s
  external_labels:
    datacenter: 'dc2'
    cluster: 'prod'

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node-exporter'
    static_configs:
      - targets:
          - 'node4:9100'
          - 'node5:9100'
```

**Global Prometheus:**

```yaml
# /etc/prometheus/prometheus-global.yml
global:
  scrape_interval: 30s
  external_labels:
    monitor: 'global'

scrape_configs:
  # Federation from datacenter Prometheus servers
  - job_name: 'federation'
    honor_labels: true
    metrics_path: '/federate'
    
    params:
      'match[]':
        # Federate all instance-level metrics
        - '{job="node-exporter"}'
        # Federate recording rules
        - '{__name__=~"instance:.*"}'
        # Exclude Prometheus internal metrics
        - '{job!="prometheus"}'
    
    static_configs:
      - targets:
          - 'prometheus-dc1:9090'
          - 'prometheus-dc2:9090'
        labels:
          region: 'us'
```

#### Setup 2: Selective Federation

**Only aggregate metrics (not raw data):**

```yaml
scrape_configs:
  - job_name: 'federation'
    honor_labels: true
    metrics_path: '/federate'
    
    params:
      'match[]':
        # Only recording rules (aggregated metrics)
        - '{__name__=~"job:.*"}'
        - '{__name__=~"instance:.*"}'
        - '{__name__=~"datacenter:.*"}'
    
    static_configs:
      - targets: ['dc1-prometheus:9090', 'dc2-prometheus:9090']
```

#### Setup 3: Cross-Service Federation

**Service A Prometheus:**

```yaml
scrape_configs:
  # Own metrics
  - job_name: 'service-a'
    static_configs:
      - targets: ['localhost:8080']

  # Federation from Service B
  - job_name: 'service-b-federation'
    honor_labels: true
    metrics_path: '/federate'
    params:
      'match[]':
        - '{job="service-b"}'
        - '{__name__=~"service_b_.*"}'
    static_configs:
      - targets: ['service-b-prometheus:9090']
```

### Complete Example

**Regional Prometheus Servers:**

```yaml
# prometheus-east.yml
global:
  scrape_interval: 15s
  external_labels:
    region: 'east'
    environment: 'production'

rule_files:
  - 'rules/*.yml'

scrape_configs:
  - job_name: 'nodes'
    static_configs:
      - targets: ['node1:9100', 'node2:9100']

  - job_name: 'applications'
    static_configs:
      - targets: ['app1:8080', 'app2:8080']
```

```yaml
# prometheus-west.yml
global:
  scrape_interval: 15s
  external_labels:
    region: 'west'
    environment: 'production'

rule_files:
  - 'rules/*.yml'

scrape_configs:
  - job_name: 'nodes'
    static_configs:
      - targets: ['node3:9100', 'node4:9100']

  - job_name: 'applications'
    static_configs:
      - targets: ['app3:8080', 'app4:8080']
```

**Global Prometheus:**

```yaml
# prometheus-global.yml
global:
  scrape_interval: 60s
  external_labels:
    monitor: 'global-view'

scrape_configs:
  # Federation endpoint
  - job_name: 'federated-metrics'
    honor_labels: true
    metrics_path: '/federate'
    
    params:
      'match[]':
        # Recording rules only
        - '{__name__=~".*:.*:.*"}'
        # Specific metrics
        - '{__name__=~"up|job:.*"}'
    
    static_configs:
      - targets:
          - 'prometheus-east:9090'
        labels:
          federated_from: 'east'
      
      - targets:
          - 'prometheus-west:9090'
        labels:
          federated_from: 'west'
    
    scrape_interval: 60s
    scrape_timeout: 55s
```

### Federation Query Parameters

**Match Patterns:**

```yaml
params:
  'match[]':
    # Match specific job
    - '{job="node-exporter"}'
    
    # Match multiple jobs
    - '{job=~"node-exporter|mysql-exporter"}'
    
    # Match by metric name prefix
    - '{__name__=~"http_.*"}'
    
    # Match recording rules
    - '{__name__=~".*:.*:.*"}'
    
    # Match all except Prometheus internal
    - '{job!~"prometheus"}'
    
    # Combine conditions
    - '{job="app",environment="production"}'
```

### Docker Compose Example

```yaml
version: '3'

services:
  # Regional Prometheus
  prometheus-east:
    image: prom/prometheus
    volumes:
      - ./prometheus-east.yml:/etc/prometheus/prometheus.yml
      - ./rules:/etc/prometheus/rules
    ports:
      - "9091:9090"
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'

  prometheus-west:
    image: prom/prometheus
    volumes:
      - ./prometheus-west.yml:/etc/prometheus/prometheus.yml
      - ./rules:/etc/prometheus/rules
    ports:
      - "9092:9090"
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'

  # Global Prometheus
  prometheus-global:
    image: prom/prometheus
    volumes:
      - ./prometheus-global.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
    depends_on:
      - prometheus-east
      - prometheus-west
```

### Monitoring Federation

**Check federated metrics:**

```promql
# Count federated time series
count(up{job="federation"})

# Scrape duration
prometheus_target_scrape_duration_seconds{job="federation"}

# Federation endpoint request rate
rate(prometheus_http_requests_total{handler="/federate"}[5m])
```

### Best Practices

1. **Use `honor_labels: true`** to preserve original labels
2. **Federate aggregated metrics** (recording rules) not raw data
3. **Use appropriate scrape intervals** (longer for global Prometheus)
4. **Add external labels** to identify source Prometheus
5. **Monitor federation lag** and scrape duration
6. **Use specific match patterns** to limit data volume
7. **Consider Thanos or Cortex** for larger deployments
8. **Don't create circular federation** (A→B→A)
9. **Test federation queries** before production
10. **Document federation topology**

### Troubleshooting

**High cardinality:**
```yaml
# Be selective with matches
params:
  'match[]':
    - '{__name__=~"job:.*"}' # Only aggregated metrics
```

**Timeout issues:**
```yaml
scrape_configs:
  - job_name: 'federation'
    scrape_timeout: 55s
    scrape_interval: 60s
```

**Missing labels:**
```yaml
# Ensure honor_labels is true
honor_labels: true
```

---

## Grafana Integration

Grafana is the most popular visualization tool for Prometheus metrics.

### Installing Grafana

**Ubuntu/Debian:**

```bash
# Add GPG key
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -

# Add repository
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee /etc/apt/sources.list.d/grafana.list

# Install
sudo apt update
sudo apt install grafana

# Start service
sudo systemctl start grafana-server
sudo systemctl enable grafana-server
```

**Docker:**

```bash
docker run -d \
  --name=grafana \
  -p 3000:3000 \
  grafana/grafana-oss
```

**Docker Compose:**

```yaml
version: '3'
services:
  grafana:
    image: grafana/grafana-oss:latest
    ports:
      - "3000:3000"
    volumes:
      - grafana-storage:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false

volumes:
  grafana-storage:
```

### Initial Setup

1. Access Grafana: `http://localhost:3000`
2. Default credentials: admin/admin
3. Change password on first login

### Adding Prometheus Data Source

**Via UI:**

1. Click **Configuration** (gear icon) → **Data Sources**
2. Click **Add data source**
3. Select **Prometheus**
4. Set URL: `http://prometheus:9090` (or `http://localhost:9090`)
5. Click **Save & Test**

**Via Configuration File:**

```yaml
# /etc/grafana/provisioning/datasources/prometheus.yml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    jsonData:
      timeInterval: 15s
      queryTimeout: 60s
    editable: false
```

### Creating Dashboards

#### Manual Dashboard Creation

1. Click **+** → **Dashboard**
2. Click **Add new panel**
3. Select **Prometheus** data source
4. Enter query
5. Configure visualization
6. Save dashboard

#### Example: System Monitoring Dashboard

**CPU Usage Panel:**
```promql
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

**Memory Usage Panel:**
```promql
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100
```

**Disk Usage Panel:**
```promql
100 - ((node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100)
```

**Network Traffic Panel:**
```promql
# Receive
rate(node_network_receive_bytes_total[5m])

# Transmit
rate(node_network_transmit_bytes_total[5m])
```

### Using Pre-Built Dashboards

**Import Dashboard:**

1. Click **+** → **Import**
2. Enter dashboard ID or upload JSON
3. Select Prometheus data source
4. Click **Import**

**Popular Dashboard IDs:**

| Dashboard | ID | Description |
|-----------|-----|-------------|
| Node Exporter Full | 1860 | Complete system metrics |
| Node Exporter | 11074 | System monitoring |
| Prometheus Stats | 2 | Prometheus internals |
| MySQL Overview | 7362 | MySQL monitoring |
| Redis Dashboard | 11835 | Redis metrics |
| Kubernetes Cluster | 7249 | K8s overview |
| Container Monitoring | 893 | Docker containers |

### Dashboard JSON Provisioning

```json
{
  "dashboard": {
    "title": "System Overview",
    "panels": [
      {
        "title": "CPU Usage",
        "targets": [
          {
            "expr": "100 - (avg by(instance) (rate(node_cpu_seconds_total{mode=\"idle\"}[5m])) * 100)",
            "legendFormat": "{{instance}}"
          }
        ],
        "type": "graph"
      }
    ]
  }
}
```

### Complete Example: Application Dashboard

Save as `/etc/grafana/provisioning/dashboards/app-dashboard.json`:

```json
{
  "annotations": {
    "list": []
  },
  "editable": true,
  "gnetId": null,
  "graphTooltip": 0,
  "id": null,
  "links": [],
  "panels": [
    {
      "datasource": "Prometheus",
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "palette-classic"
          },
          "custom": {
            "axisLabel": "",
            "axisPlacement": "auto"
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              }
            ]
          },
          "unit": "reqps"
        }
      },
      "gridPos": {
        "h": 8,
        "w": 12,
        "x": 0,
        "y": 0
      },
      "id": 1,
      "options": {
        "legend": {
          "calcs": [],
          "displayMode": "list",
          "placement": "bottom"
        },
        "tooltip": {
          "mode": "single"
        }
      },
      "targets": [
        {
          "expr": "sum(rate(http_requests_total[5m])) by (method, status)",
          "legendFormat": "{{method}} - {{status}}",
          "refId": "A"
        }
      ],
      "title": "Request Rate",
      "type": "timeseries"
    }
  ],
  "refresh": "10s",
  "schemaVersion": 30,
  "style": "dark",
  "tags": ["application"],
  "templating": {
    "list": []
  },
  "time": {
    "from": "now-1h",
    "to": "now"
  },
  "timezone": "",
  "title": "Application Dashboard",
  "uid": "app-overview",
  "version": 0
}
```

### Variables for Dynamic Dashboards

**Create Variable:**

1. Dashboard settings → Variables → Add variable
2. Name: `instance`
3. Type: Query
4. Data source: Prometheus
5. Query: `label_values(up, instance)`

**Use in Query:**
```promql
node_cpu_seconds_total{instance="$instance"}
```

**Multiple Variables:**

```promql
node_cpu_seconds_total{instance="$instance", job="$job"}
```

### Alerting in Grafana

**Create Alert:**

1. Edit panel
2. Click **Alert** tab
3. Create alert rule
4. Configure conditions
5. Set notification channel

**Example Alert:**

```
Condition: WHEN avg() OF query(A, 5m) IS ABOVE 80
```

### Best Practices

1. **Use variables** for flexible dashboards
2. **Organize panels** into rows
3. **Add descriptions** to panels
4. **Use appropriate visualizations** (graph, gauge, stat)
5. **Set refresh intervals** appropriately
6. **Use recording rules** for complex queries
7. **Create dashboard folders** for organization
8. **Version control** dashboard JSON
9. **Use provisioning** for automated deployment
10. **Document dashboards** with text panels

---

## Advantages of Prometheus

### 1. Powerful Time Series Database

- **Efficient Storage**: Data compression reduces disk usage
- **Fast Queries**: Optimized for time-range queries
- **Scalability**: Handles millions of time series
- **Flexible Retention**: Configurable data retention policies

### 2. Pull-Based Architecture

**Benefits:**
- Simplified discovery of new services
- Reduced network overhead
- Enhanced security (Prometheus initiates connections)
- Easier debugging (can scrape manually)
- No agent configuration needed on targets

### 3. Flexible Query Language (PromQL)

- Multi-dimensional data model
- Powerful aggregation functions
- Mathematical operations
- Regular expression support
- Time manipulation capabilities

### 4. Service Discovery

- **Automatic Discovery**: No manual configuration needed
- **Multiple Mechanisms**: Kubernetes, Consul, DNS, cloud APIs
- **Dynamic Environments**: Adapts to infrastructure changes
- **Reduces Operational Overhead**: Self-updating targets

### 5. Multi-Dimensional Data Model

```
metric_name{label1="value1", label2="value2"}
```

- Labels enable flexible filtering
- Easy to slice data by dimensions
- Supports complex queries
- Enables powerful aggregations

### 6. No External Dependencies

- **Standalone Operation**: No distributed storage needed
- **Simple Deployment**: Single binary
- **Reliable During Outages**: Works when other systems fail
- **Low Operational Complexity**: Easy to maintain

### 7. Built-in Alerting

- Define alerts as code
- PromQL-based alert conditions
- AlertManager for notification handling
- Flexible routing and grouping
- Multiple notification channels

### 8. Extensive Ecosystem

- **200+ Official Exporters**: Support for diverse systems
- **Active Community**: Continuous development
- **Client Libraries**: Go, Python, Java, Ruby, .NET, etc.
- **Integration**: Works with Grafana, Kubernetes, etc.
- **Cloud Provider Support**: AWS, GCP, Azure integrations

### 9. Cloud-Native Design

- **CNCF Project**: Industry standard
- **Kubernetes Integration**: Native service discovery
- **Container-Friendly**: Easy to deploy in containers
- **Microservices-Oriented**: Perfect for dynamic architectures

### 10. Open Source

- Free to use
- No vendor lock-in
- Community-driven development
- Transparent development process
- Active contribution from users

### 11. Reliable and Proven

- **Battle-Tested**: Used by major companies
- **Stable**: Mature codebase
- **Well-Documented**: Extensive documentation
- **Large User Base**: Strong community support

### 12. Efficient Performance

- **Low Resource Usage**: Efficient data compression
- **Fast Scraping**: Can handle thousands of targets
- **Quick Queries**: Sub-second query response times
- **Horizontal Scalability**: Via federation

---

## Disadvantages and Limitations

### 1. Limited Long-Term Storage

**Problem:**
- Designed for real-time monitoring (days to weeks)
- Not optimized for long-term historical data
- Limited retention periods

**Workarounds:**
- Use remote storage solutions (Thanos, Cortex)
- Integrate with time-series databases (InfluxDB, TimescaleDB)
- Implement data federation

### 2. No Built-in High Availability

**Problem:**
- Single Prometheus instance = single point of failure
- No native clustering
- Data duplication with multiple instances

**Workarounds:**
- Run multiple Prometheus replicas
- Use Thanos for HA and long-term storage
- Implement federation for redundancy

### 3. Horizontal Scaling Challenges

**Problem:**
- Only vertical scaling (bigger server)
- Cannot distribute load across multiple instances
- High cardinality can cause performance issues

**Workarounds:**
- Use federation for sharding
- Implement recording rules to reduce cardinality
- Deploy multiple Prometheus servers with different responsibilities

### 4. Pull-Based Model Limitations

**Problem:**
- Short-lived jobs difficult to monitor
- Targets behind firewalls hard to reach
- NAT traversal issues
- Cannot monitor truly ephemeral workloads

**Workarounds:**
- Use Push Gateway for batch jobs
- Set up VPN for firewall traversal
- Use service discovery with proper network setup

### 5. Steep Learning Curve

**Problem:**
- PromQL can be complex
- Requires understanding of time series concepts
- Configuration can be overwhelming
- Many advanced features to learn

**Mitigation:**
- Extensive documentation available
- Active community support
- Training resources (PromLabs)
- Start simple and expand gradually

### 6. Limited Built-in Visualization

**Problem:**
- Basic expression browser
- Simple graphs only
- No advanced dashboards
- Limited customization

**Solution:**
- Integrate with Grafana
- Use other visualization tools
- Build custom dashboards

### 7. Cardinality Issues

**Problem:**
- High cardinality = performance degradation
- Memory consumption increases
- Query performance suffers
- Storage requirements grow

**Prevention:**
- Limit number of unique labels
- Avoid user IDs in labels
- Use recording rules for aggregations
- Monitor cardinality metrics

### 8. No Built-in Authentication

**Problem:**
- No native authentication
- No authorization mechanisms
- Open access by default
- Security concerns in multi-tenant environments

**Workarounds:**
- Use reverse proxy (Nginx, HAProxy)
- Implement OAuth2 proxy
- Network-level security (firewalls, VPN)
- Basic auth via reverse proxy

### 9. No Native TLS Support

**Problem:**
- Scraping endpoints not encrypted
- Web UI not secure by default

**Workarounds:**
- Use reverse proxy with TLS
- Implement mTLS via proxy
- Network-level encryption (VPN)

### 10. Metrics-Only Focus

**Problem:**
- No log collection
- No distributed tracing
- Not a complete observability solution

**Solution:**
- Combine with ELK/Loki for logs
- Use Jaeger/Zipkin for traces
- Implement full observability stack

### 11. Limited Ecosystem for Legacy Systems

**Problem:**
- Fewer exporters for older systems
- May require custom exporter development
- Integration challenges with proprietary systems

**Solution:**
- Develop custom exporters
- Use generic exporters (SNMP, JMX)
- Community contributions

### 12. Resource Consumption

**Problem:**
- Can consume significant memory
- Storage requirements grow with retention
- 1 million metrics ≈ 100GB RAM

**Management:**
- Tune retention policies
- Use remote storage
- Implement proper capacity planning
- Monitor Prometheus resource usage

### 13. Alert Configuration Complexity

**Problem:**
- Alert rules require careful tuning
- PromQL-based conditions can be complex
- AlertManager configuration is verbose
- Risk of alert fatigue or missed alerts

**Best Practices:**
- Start with simple alerts
- Test thoroughly before production
- Document alert thresholds
- Regular review and tuning

### 14. Federation Overhead

**Problem:**
- Can introduce latency
- Increased network traffic
- Complexity in multi-level setups

**Considerations:**
- Aggregate data before federating
- Use appropriate scrape intervals
- Monitor federation health

---

## Best Practices

### Configuration

1. **Use Consistent Naming**: Follow naming conventions for metrics and labels
2. **Version Control**: Keep configurations in Git
3. **Document Everything**: Add comments to configurations
4. **Test Before Deploy**: Use `promtool check` commands
5. **Use External Labels**: Identify Prometheus instances

### Metrics Design

1. **Keep Labels Low Cardinality**: Avoid user IDs, timestamps in labels
2. **Use Appropriate Metric Types**: Counter for cumulative, Gauge for current values
3. **Include Units in Names**: `_bytes`, `_seconds`, `_total`
4. **Use Base Units**: Seconds not milliseconds, bytes not kilobytes

### Querying

1. **Use Recording Rules**: For expensive queries used repeatedly
2. **Aggregate Early**: Use `by` clause to reduce dimensionality
3. **Use rate() for Counters**: Never use absolute counter values
4. **Set Appropriate Time Ranges**: Match your use case

### Alerting

1. **Alert on Symptoms**: Not causes (alert on high latency, not CPU)
2. **Use `for` Clause**: Avoid flapping alerts
3. **Meaningful Annotations**: Include runbooks and context
4. **Appropriate Severity Levels**: critical/warning/info
5. **Group Related Alerts**: Reduce notification spam

### Security

1. **Use Reverse Proxy**: For TLS and authentication
2. **Network Segmentation**: Limit Prometheus access
3. **Monitor Prometheus**: Alert on Prometheus failures
4. **Regular Updates**: Keep Prometheus up to date
5. **Limit Exposure**: Don't expose to public internet

### Performance

1. **Monitor Prometheus**: Track resource usage
2. **Tune Retention**: Based on storage capacity
3. **Use Recording Rules**: Reduce query complexity
4. **Implement Federation**: For scaling
5. **Capacity Planning**: Plan for growth

### Operations

1. **Backup Configuration**: Regular backups of config files
2. **Test Recovery**: Practice disaster recovery
3. **Monitor Targets**: Alert on scrape failures
4. **Regular Reviews**: Review metrics and alerts
5. **Documentation**: Keep runbooks updated

---

## Conclusion

Prometheus is a powerful, flexible, and reliable monitoring system that excels in cloud-native and dynamic environments. While it has limitations like limited long-term storage and horizontal scaling challenges, its advantages—including the pull-based architecture, powerful query language, extensive ecosystem, and strong community support—make it the de facto standard for modern infrastructure monitoring.

For production deployments, consider:
- Combining with Grafana for visualization
- Using Thanos or Cortex for long-term storage and HA
- Integrating with logging (Loki) and tracing (Jaeger) for full observability
- Implementing proper security measures
- Regular monitoring of Prometheus itself

With proper configuration and understanding, Prometheus provides a robust foundation for observability in modern infrastructure.

---

## Additional Resources

### Official Documentation
- [Prometheus Documentation](https://prometheus.io/docs/)
- [PromQL Documentation](https://prometheus.io/docs/prometheus/latest/querying/basics/)
- [Exporters List](https://prometheus.io/docs/instrumenting/exporters/)

### Training
- [PromLabs Training](https://training.promlabs.com/)
- [Prometheus Certified Associate (PCA)](https://training.linuxfoundation.org/certification/prometheus-certified-associate/)

### Community
- [Prometheus GitHub](https://github.com/prometheus/prometheus)
- [CNCF Slack #prometheus](https://slack.cncf.io/)
- [Prometheus Users Mailing List](https://groups.google.com/forum/#!forum/prometheus-users)

### Books
- "Prometheus: Up & Running" by Brian Brazil
- "Infrastructure Monitoring with Prometheus" by James Turnbull

---

**Version:** 1.0  
**Last Updated:** October 2025  
**Author:** DevOps Tutorial Series  

For updates and corrections, please refer to the official Prometheus documentation.