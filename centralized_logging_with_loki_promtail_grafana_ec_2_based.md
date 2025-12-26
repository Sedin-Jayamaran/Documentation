# Centralized Logging Setup (EC2 + Docker + Promtail + Loki + Grafana)

This document is a **hands-on reference** for setting up centralized logging using **Loki, Promtail, and Grafana** across multiple EC2 instances.

It reflects a **real-world DevOps approach**, not a toy setup.

---

## 🧠 Architecture Overview

```
EC2-A (Python App) ── Promtail ──┐
                                 ├──▶ Loki (EC2-C) ──▶ Grafana (EC2-C)
EC2-B (Node App)   ── Promtail ──┘
```

### Roles

| EC2 | Responsibility |
|----|---------------|
| EC2-A | Python application + Promtail |
| EC2-B | Node.js application + Promtail |
| EC2-C | Loki + Grafana |

---

## 🔑 Core Principles

- Applications log to **stdout / stderr**
- Docker stores logs in `/var/lib/docker/containers/*/*-json.log`
- Promtail runs **on the same node as the app**
- Loki is **centralized**
- Grafana **queries Loki**, not Promtail
- Labels are **low-cardinality** (`app`, `service`, `env`)

---

## 1️⃣ EC2-A / EC2-B: Base OS Setup (Amazon Linux 2023)

```bash
sudo dnf update -y
sudo dnf install docker -y
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ec2-user
exit
```

Reconnect SSH, then verify:

```bash
docker ps
```

---

## 2️⃣ Python App (EC2-A)

### app.py

```python
import time
import logging

logging.basicConfig(level=logging.INFO)

while True:
    logging.info("Python app is running")
    time.sleep(5)
```

### Dockerfile

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY app.py .
CMD ["python", "app.py"]
```

### Build & Run

```bash
docker build -t python-app .
docker run -d --name python-app python-app
```

Verify:

```bash
docker logs -f python-app
```

---

## 3️⃣ Node App (EC2-B)

### app.js

```js
setInterval(() => {
  console.log("Node app is running");
}, 5000);
```

### Dockerfile

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY app.js .
CMD ["node", "app.js"]
```

### Build & Run

```bash
docker build -t node-app .
docker run -d --name node-app node-app
```

Verify:

```bash
docker logs -f node-app
```

---

## 4️⃣ Promtail (EC2-A / EC2-B)

Promtail **must run on every node that runs containers**.

### promtail-config.yml (Python EC2)

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://<LOKI_PRIVATE_IP>:3100/loki/api/v1/push

scrape_configs:
  - job_name: docker
    static_configs:
      - targets:
          - localhost
        labels:
          job: docker
          app: python-app
          __path__: /var/lib/docker/containers/*/*-json.log
```

### promtail-config.yml (Node EC2)

Only label changes:

```yaml
labels:
  job: docker
  app: node-app
```

### Run Promtail

```bash
docker run -d \
  --name promtail \
  -v /var/lib/docker/containers:/var/lib/docker/containers:ro \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  -v $(pwd)/promtail-config.yml:/etc/promtail/config.yml \
  grafana/promtail:3.0.0 \
  -config.file=/etc/promtail/config.yml
```

Verify:

```bash
docker logs -f promtail
```

---

## 5️⃣ Loki (EC2-C)

### loki-config.yml

```yaml
auth_enabled: false

server:
  http_listen_port: 3100

common:
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2024-01-01
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

limits_config:
  retention_period: 168h
```

### Run Loki

```bash
docker run -d \
  --name loki \
  -p 3100:3100 \
  -v $(pwd)/loki-config.yml:/etc/loki/config.yml \
  grafana/loki:2.9.4 \
  -config.file=/etc/loki/config.yml
```

Verify:

```bash
curl http://localhost:3100/ready
```

---

## 6️⃣ Grafana (EC2-C)

```bash
docker run -d \
  --name grafana \
  -p 3000:3000 \
  grafana/grafana:10.2.0
```

Access:
```
http://<EC2-C-public-IP>:3000
```

Login: `admin / admin`

---

## 7️⃣ Grafana → Loki Datasource

⚠️ Grafana runs in a container, so **do not use localhost**.

Use **EC2-C private IP**:

```
http://<LOKI_PRIVATE_IP>:3100
```

Save & Test → ✅ Connected

---

## 8️⃣ LogQL Examples

### View logs

```logql
{app="python-app"}
{app="node-app"}
```

### Multiple apps

```logql
{app=~"python-app|node-app"}
```

### Filter content

```logql
{app="node-app"} |= "running"
```

### Log-based metrics

```logql
count_over_time({app="python-app"}[1m])
```

---

## 🚨 Label Rules (VERY IMPORTANT)

❌ Do NOT label:
- container_id
- request_id
- user_id

✅ DO label:
- app
- service
- env
- region

---

## ✅ Final Outcome

- Centralized logs across EC2s
- Per-service visibility
- Loki receives logs
- Grafana visualizes logs
- Production-grade observability foundation

---

## 🔮 Next Steps

- JSON structured logging
- Log-based alerts
- ECS + FireLens mapping
- Retention & scaling strategies

---

**This setup directly maps to ECS, EKS, and Kubernetes logging architectures.**

