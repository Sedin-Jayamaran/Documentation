# Complete Guide to Dockerizing Three-Tier Applications

**A Comprehensive Reference with Tricks, Tips, Pro Commands, Best Practices, and Hidden Details**

---

## Table of Contents

1. [Introduction & Architecture Overview](#introduction--architecture-overview)
2. [Three-Tier Application Architecture](#three-tier-application-architecture)
3. [Dockerfile Best Practices & Optimization](#dockerfile-best-practices--optimization)
4. [Building the Frontend Tier](#building-the-frontend-tier)
5. [Building the Backend Tier](#building-the-backend-tier)
6. [Building the Database Tier](#building-the-database-tier)
7. [Docker Networking for Multi-Tier Apps](#docker-networking-for-multi-tier-apps)
8. [Docker Compose Orchestration](#docker-compose-orchestration)
9. [Security Best Practices](#security-best-practices)
10. [Performance Optimization & Resource Management](#performance-optimization--resource-management)
11. [Advanced Techniques & Pro Tips](#advanced-techniques--pro-tips)
12. [Common Practices & Anti-Patterns](#common-practices--anti-patterns)
13. [Troubleshooting & Debugging](#troubleshooting--debugging)

---

## Introduction & Architecture Overview

Dockerizing a three-tier application means containerizing your presentation layer (frontend), business logic layer (backend), and data persistence layer (database) into separate, manageable containers. This approach provides:

- **Scalability**: Each tier scales independently based on demand
- **Isolation**: Services run in isolated environments reducing conflicts
- **Consistency**: Same container runs identically across dev, test, and production
- **Microservices-Ready**: Easier to transition to microservices architecture

**Three-Tier Application Stack:**
```
┌─────────────────────────────────────┐
│  Presentation Tier                  │
│  (Frontend - React/Vue/Angular)     │
│  Port: 3000 (Development)           │
└────────────┬────────────────────────┘
             │ HTTP/REST API
┌────────────▼────────────────────────┐
│  Application Tier                   │
│  (Backend - Node/Python/Java)       │
│  Port: 5000 (Internal)              │
└────────────┬────────────────────────┘
             │ Database Protocol
┌────────────▼────────────────────────┐
│  Data Tier                          │
│  (Database - MySQL/PostgreSQL)      │
│  Port: 3306 (Internal)              │
└─────────────────────────────────────┘
```

---

## Three-Tier Application Architecture

### What Each Tier Does

**1. Frontend (Presentation Tier)**
- User interface and user experience
- Handles client-side logic and rendering
- Makes API calls to backend
- Technologies: React, Vue.js, Angular, Next.js
- Containerized with: Nginx or Node.js server

**2. Backend (Application Tier)**
- Business logic processing
- API endpoints and routing
- Data validation and transformation
- Database operations
- Technologies: Node.js, Python, Java, Go
- Exposed only to frontend and database

**3. Database (Data Tier)**
- Persistent data storage
- Transactions and data integrity
- Technologies: MySQL, PostgreSQL, MongoDB
- Not directly exposed to frontend (only backend communicates)

### Typical Request Flow

```
Client Request → Frontend Container → Backend Container → Database Container
                                    ↓
                              Response (JSON/Data)
```

---

## Dockerfile Best Practices & Optimization

### Pro Tip #1: Understanding Docker Image Layers

Every instruction in a Dockerfile creates a layer. Docker caches layers, so understanding this is crucial:

```bash
# Check layers in an image
docker history my-app:latest

# See layer sizes
docker history --no-trunc my-app:latest
```

**Key Insight**: If you install a package in one layer and delete it in another, the final image STILL contains it because Docker preserves layer history.

### Pro Tip #2: Multi-Stage Builds (The Game Changer)

Multi-stage builds dramatically reduce final image size by separating build-time dependencies from runtime dependencies.

**Before Multi-Stage** (Bloated - 800MB+):
```dockerfile
FROM node:18
WORKDIR /app
COPY . .
RUN npm install
RUN npm run build
# Final image contains ALL dependencies including dev tools
EXPOSE 3000
CMD ["npm", "start"]
```

**After Multi-Stage** (Optimized - 150MB):
```dockerfile
# Stage 1: Builder
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

# Stage 2: Runtime
FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package*.json ./
EXPOSE 3000
CMD ["npm", "start"]
```

**Reduction**: 800MB → 150MB (81% reduction!)

### Pro Tip #3: Choose the Right Base Image

| Base Image | Size | Use Case | Security |
|-----------|------|----------|----------|
| `ubuntu:22.04` | 77 MB | General purpose | Good |
| `node:18` | 913 MB | Node apps | Good |
| `node:18-alpine` | 170 MB | Lightweight Node | Excellent |
| `node:18-slim` | 187 MB | Minimal Node | Excellent |
| `distroless/nodejs18` | 106 MB | Ultra-secure | Excellent (no shell!) |
| `scratch` | 0 MB | Compiled binaries only | Maximum |

**Pro Recommendation**: Use Alpine for most cases, distroless for security-critical apps.

```dockerfile
# Ultra-optimized frontend
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=development && npm run build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
# Final size: ~45MB (compared to 800MB+ with standard images)
```

### Pro Tip #4: Leverage Build Cache Effectively

Place commands that change frequently at the END of Dockerfile:

```dockerfile
# ✓ GOOD: Layers are cached until dependencies change
FROM node:18-alpine
WORKDIR /app

# Dependencies rarely change - cache hit
COPY package*.json ./
RUN npm ci --only=production

# Code changes frequently - only this rebuilds
COPY . .
RUN npm run build

# This layer rebuilds every time due to date
RUN echo "Built on $(date)" > /app/.build-info
```

**The Magic**: If dependencies don't change, Docker reuses the cached layer (instant build). If code changes, Docker rebuilds from that point onward.

### Pro Tip #5: Use .dockerignore

Similar to `.gitignore`, prevents unnecessary files from being copied (saves space & improves caching):

```
# .dockerignore
node_modules
npm-debug.log
.git
.gitignore
README.md
.env.local
dist
coverage
.DS_Store
.vscode
.idea
```

### Pro Tip #6: Minimize Layer Count

```dockerfile
# ✗ Bad: 5 layers created
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y git
RUN apt-get clean
RUN rm -rf /var/lib/apt/lists/*

# ✓ Good: 1 layer created
RUN apt-get update && \
    apt-get install -y curl git && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

### Pro Tip #7: Image Size Optimization Commands

```bash
# Check actual image size
docker images my-app

# Detailed layer analysis (install dive first: go install github.com/wagoodman/dive@latest)
dive my-app:latest

# Check what's taking space
docker inspect my-app:latest

# Show image history
docker history my-app:latest

# Prune unused images and layers
docker system prune -a
```

---

## Building the Frontend Tier

### Typical Frontend Structure
```
frontend/
├── Dockerfile
├── .dockerignore
├── package.json
├── package-lock.json
├── public/
├── src/
└── nginx.conf (for production)
```

### Development Dockerfile (React/Vue/Angular)

```dockerfile
FROM node:18-alpine

WORKDIR /app

# Cache npm dependencies
COPY package*.json ./
RUN npm ci

# Copy source code
COPY . .

# Build application
RUN npm run build

# Expose port
EXPOSE 3000

# Development mode with hot reload
CMD ["npm", "start"]
```

### Production Dockerfile (Multi-Stage, Optimized)

```dockerfile
# Stage 1: Build
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=development

COPY . .
RUN npm run build

# Stage 2: Serve with Nginx
FROM nginx:alpine

# Copy nginx config
COPY nginx.conf /etc/nginx/nginx.conf

# Copy built frontend from builder
COPY --from=builder /app/dist /usr/share/nginx/html

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:80/health || exit 1

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

### Nginx Configuration (nginx.conf)

```nginx
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;
    sendfile on;
    tcp_nopush on;
    keepalive_timeout 65;
    gzip on;

    server {
        listen 80;
        server_name _;
        root /usr/share/nginx/html;

        # Serve SPA
        location / {
            try_files $uri $uri/ /index.html;
        }

        # API proxy to backend
        location /api/ {
            proxy_pass http://backend:5000/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        # Health check endpoint
        location /health {
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }
    }
}
```

**Pro Tips for Frontend**:
- Use `npm ci` instead of `npm install` for reproducible builds
- Split dependencies: `--only=development` vs `--only=production`
- Nginx is much lighter than Node for serving static files
- Always implement health checks for orchestration

---

## Building the Backend Tier

### Typical Backend Structure
```
backend/
├── Dockerfile
├── .dockerignore
├── package.json
├── app.js (or main entry)
├── routes/
├── models/
├── config/
└── .env.example
```

### Production Dockerfile (Node.js Example)

```dockerfile
FROM node:18-alpine

WORKDIR /app

# Don't run as root - create app user
RUN addgroup -g 1001 -S nodejs && adduser -S nodejs -u 1001

# Copy package files
COPY --chown=nodejs:nodejs package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy application code
COPY --chown=nodejs:nodejs . .

# Switch to non-root user
USER nodejs

# Expose port
EXPOSE 5000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s --retries=3 \
    CMD node healthcheck.js

# Use exec form to ensure proper signal handling
CMD ["node", "app.js"]
```

### Health Check Script (healthcheck.js)

```javascript
const http = require('http');

const options = {
    hostname: 'localhost',
    port: 5000,
    path: '/health',
    method: 'GET',
    timeout: 2000
};

const request = http.request(options, (res) => {
    if (res.statusCode === 200) {
        process.exit(0);
    } else {
        process.exit(1);
    }
});

request.on('error', (err) => {
    console.error('Health check failed:', err);
    process.exit(1);
});

request.on('timeout', () => {
    request.destroy();
    process.exit(1);
});

request.end();
```

### Backend App with Database Connection (app.js)

```javascript
const express = require('express');
const mysql = require('mysql2/promise');
const app = express();

app.use(express.json());

// Database connection pool
const pool = mysql.createPool({
    host: process.env.DB_HOST || 'db',
    user: process.env.DB_USER || 'app_user',
    password: process.env.DB_PASSWORD,
    database: process.env.DB_NAME || 'myapp_db',
    waitForConnections: true,
    connectionLimit: 10,
    queueLimit: 0
});

// Health endpoint
app.get('/health', (req, res) => {
    res.json({ status: 'OK', timestamp: new Date() });
});

// API endpoints
app.get('/api/users', async (req, res) => {
    try {
        const connection = await pool.getConnection();
        const [rows] = await connection.query('SELECT * FROM users');
        connection.release();
        res.json(rows);
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});

const PORT = process.env.PORT || 5000;
app.listen(PORT, () => {
    console.log(`Backend running on port ${PORT}`);
});
```

**Pro Tips for Backend**:
- Always run as non-root user for security
- Use connection pooling for databases
- Implement proper health checks
- Pass configuration via environment variables
- Never hardcode secrets in Dockerfile
- Use `exec` form for CMD to ensure proper signal handling

---

## Building the Database Tier

### MySQL Dockerfile (Custom Configuration)

```dockerfile
FROM mysql:8.0-alpine

# Environment variables
ENV MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD:-root_password}
ENV MYSQL_DATABASE=${MYSQL_DATABASE:-myapp_db}
ENV MYSQL_USER=${MYSQL_USER:-app_user}
ENV MYSQL_PASSWORD=${MYSQL_PASSWORD:-app_password}

# Copy custom configuration
COPY my.cnf /etc/mysql/conf.d/

# Copy initialization scripts
COPY ./init-scripts/ /docker-entrypoint-initdb.d/

# Health check
HEALTHCHECK --interval=10s --timeout=3s --start-period=40s --retries=3 \
    CMD mysqladmin ping -h localhost || exit 1

EXPOSE 3306
```

### MySQL Configuration (my.cnf)

```ini
[mysqld]
# Performance
skip-host-cache
skip-name-resolve
max_connections=100
default-storage-engine=InnoDB

# Binary logging (for backup/replication)
log_bin=mysql-bin
binlog_format=row

# Slow query log
slow_query_log=1
slow_query_log_file=/var/log/mysql/slow-query.log
long_query_time=2

# Error logging
log_error=/var/log/mysql/error.log

# InnoDB
innodb_buffer_pool_size=256M
innodb_log_file_size=100M
innodb_file_per_table=1

# Character set
character_set_server=utf8mb4
collation_server=utf8mb4_unicode_ci
```

### Database Initialization Script (init-scripts/01-init.sql)

```sql
-- Create schema
CREATE TABLE IF NOT EXISTS users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS posts (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    title VARCHAR(255) NOT NULL,
    content LONGTEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

-- Create indexes
CREATE INDEX idx_user_email ON users(email);
CREATE INDEX idx_post_user ON posts(user_id);

-- Insert sample data
INSERT INTO users (name, email) VALUES ('Admin User', 'admin@example.com');
INSERT INTO posts (user_id, title, content) VALUES (1, 'Welcome', 'This is the first post');
```

**Pro Tips for Database**:
- Use Alpine-based MySQL images for smaller size
- Initialize database with scripts (auto-runs on first start)
- Implement health checks for database availability
- Configure slow query logging for performance monitoring
- Use proper indexes for frequently queried columns
- Never expose database port publicly in production

---

## Docker Networking for Multi-Tier Apps

### Network Types in Docker

```
┌─────────────────────────────────────────────────────────┐
│ Network Types                                           │
├──────────────────┬──────────────────┬──────────────────┤
│ Bridge (Default) │ Host             │ Overlay (Swarm)  │
├──────────────────┼──────────────────┼──────────────────┤
│ Container to     │ Container shares │ Multi-host       │
│ container on     │ host networking  │ communication    │
│ same host        │                  │                  │
│ Isolated from    │ Better           │ For Docker Swarm │
│ other networks   │ performance      │ or Kubernetes    │
└──────────────────┴──────────────────┴──────────────────┘
```

### Create Custom Bridge Network

```bash
# Create custom bridge network
docker network create -d bridge three-tier-app-network

# Inspect network
docker network inspect three-tier-app-network

# List all networks
docker network ls

# Remove network
docker network rm three-tier-app-network

# Connect existing container to network
docker network connect three-tier-app-network frontend

# Disconnect container
docker network disconnect three-tier-app-network frontend
```

### Service Discovery (DNS in Docker)

Docker Compose automatically creates DNS records:

```
Frontend container can reach Backend using: http://backend:5000
Backend container can reach Database using: mysql://db:3306
```

**How It Works**:
1. Docker creates internal DNS server in the network
2. Each container registers with its service name
3. Other containers can resolve by name automatically

**Example in Node.js**:
```javascript
// No need for IP addresses!
const connection = mysql.createConnection({
    host: 'db',        // Docker DNS resolves this
    user: 'app_user',
    password: process.env.DB_PASSWORD,
    database: 'myapp_db'
});
```

---

## Docker Compose Orchestration

### Production-Ready docker-compose.yml

```yaml
version: '3.9'

services:
  # Frontend Service
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    container_name: three-tier-frontend
    ports:
      - "80:80"
    environment:
      - REACT_APP_API_URL=http://localhost/api
    depends_on:
      backend:
        condition: service_healthy
    networks:
      - app-network
    restart: unless-stopped
    labels:
      - "com.three-tier.description=Frontend tier"

  # Backend Service
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: three-tier-backend
    ports:
      - "5000:5000"
    environment:
      - NODE_ENV=production
      - DB_HOST=db
      - DB_USER=${DB_USER:-app_user}
      - DB_PASSWORD=${DB_PASSWORD}
      - DB_NAME=${DB_NAME:-myapp_db}
      - PORT=5000
    depends_on:
      db:
        condition: service_healthy
    networks:
      - app-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "node", "healthcheck.js"]
      interval: 30s
      timeout: 3s
      retries: 3
      start_period: 40s

  # Database Service
  db:
    image: mysql:8.0-alpine
    container_name: three-tier-database
    environment:
      - MYSQL_ROOT_PASSWORD=${DB_ROOT_PASSWORD}
      - MYSQL_DATABASE=${DB_NAME:-myapp_db}
      - MYSQL_USER=${DB_USER:-app_user}
      - MYSQL_PASSWORD=${DB_PASSWORD}
    volumes:
      - db-data:/var/lib/mysql
      - ./database/init-scripts:/docker-entrypoint-initdb.d
      - ./database/my.cnf:/etc/mysql/conf.d/my.cnf
    ports:
      - "3306:3306"
    networks:
      - app-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 3s
      retries: 3

networks:
  app-network:
    driver: bridge

volumes:
  db-data:
    driver: local
```

### Environment File (.env)

```bash
# .env file (never commit to Git!)
DB_ROOT_PASSWORD=secure_root_password_123
DB_USER=app_user
DB_PASSWORD=secure_app_password_456
DB_NAME=myapp_db

# Frontend
REACT_APP_API_URL=http://localhost/api

# Backend
NODE_ENV=production
```

### Pro Compose Commands

```bash
# Build all services
docker-compose build

# Build specific service
docker-compose build frontend

# Start services in background
docker-compose up -d

# View logs from all services
docker-compose logs

# View logs from specific service
docker-compose logs -f backend

# Restart service
docker-compose restart backend

# Stop all services
docker-compose stop

# Stop and remove containers, networks (data persists)
docker-compose down

# Stop, remove everything including volumes (WARNING: data loss!)
docker-compose down -v

# Execute command in container
docker-compose exec backend npm run migrate

# Scale service (if stateless)
docker-compose up -d --scale backend=3

# Validate compose file
docker-compose config
```

### Pro Tip: Conditional Startup

```yaml
version: '3.9'
services:
  backend:
    build: ./backend
    depends_on:
      db:
        condition: service_healthy  # Wait for DB to be healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
```

---

## Security Best Practices

### Security Layer 1: Don't Run as Root

```dockerfile
FROM node:18-alpine

# Create non-root user
RUN addgroup -g 1001 -S nodejs && adduser -S nodejs -u 1001

COPY --chown=nodejs:nodejs . .

# Switch to non-root user before running app
USER nodejs

CMD ["node", "app.js"]
```

### Security Layer 2: Use Secrets, Not Environment Variables

**❌ Bad (Exposed in docker inspect)**:
```yaml
environment:
  - DB_PASSWORD=secret123
```

**✓ Good (Hidden in /run/secrets)**:
```yaml
services:
  backend:
    environment:
      - DB_PASSWORD_FILE=/run/secrets/db_password
    secrets:
      - db_password

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

**In Application Code**:
```javascript
// Read from file instead of environment
const fs = require('fs');
const dbPassword = fs.readFileSync('/run/secrets/db_password', 'utf8').trim();
```

### Security Layer 3: Container Scanning

```bash
# Scan image for vulnerabilities
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image my-app:latest

# Using Grype
grype my-app:latest

# View image layers for suspicious content
docker history my-app:latest
```

### Security Layer 4: Resource Limits

```yaml
services:
  backend:
    deploy:
      resources:
        limits:
          cpus: '1'           # Max 1 CPU core
          memory: 512M        # Max 512MB RAM
        reservations:
          cpus: '0.5'         # Minimum reserved
          memory: 256M        # Minimum reserved
```

### Security Layer 5: Read-Only Root Filesystem

```yaml
services:
  backend:
    read_only: true
    tmpfs:
      - /tmp
      - /app/logs
```

### Security Checklist

```dockerfile
# ✓ SECURITY BEST PRACTICES DOCKERFILE

FROM node:18-alpine

# 1. Never install unnecessary packages
RUN apk add --no-cache curl

# 2. Create non-root user
RUN addgroup -g 1001 -S nodejs && adduser -S nodejs -u 1001

# 3. Use specific package versions (no latest!)
COPY package-lock.json .

# 4. Don't store secrets
# ✗ NEVER: ENV DB_PASSWORD=secret

# 5. Scan for vulnerabilities during build
# Run: docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image

# 6. Run as non-root
USER nodejs

# 7. Use read-only where possible
RUN mkdir -p /tmp && chmod 777 /tmp

CMD ["node", "app.js"]
```

---

## Performance Optimization & Resource Management

### Pro Tip #1: Memory Limits in Docker Compose

```yaml
services:
  backend:
    deploy:
      resources:
        limits:
          memory: 512M        # Hard limit - container dies if exceeded
        reservations:
          memory: 256M        # Soft limit - warning if exceeded

  db:
    deploy:
      resources:
        limits:
          memory: 1G          # MySQL needs more memory
```

**What Happens When Exceeded**:
- **Hard Limit**: Container is killed with OOM (Out of Memory) error
- **Soft Limit**: Warning issued, container continues if memory available

### Pro Tip #2: CPU Limits

```yaml
services:
  frontend:
    deploy:
      resources:
        limits:
          cpus: '0.5'         # Max 50% of one CPU core
        reservations:
          cpus: '0.25'        # Minimum 25% guarantee

  backend:
    deploy:
      resources:
        limits:
          cpus: '1.0'         # Full CPU core
```

### Pro Tip #3: Volume Performance

```yaml
# ✗ Slow: Bind mounts from host (performance penalty)
volumes:
  - /host/path:/container/path

# ✓ Fast: Named volumes (managed by Docker)
volumes:
  - db-data:/var/lib/mysql

volumes:
  db-data:
    driver: local
```

### Pro Tip #4: Image Pull Performance

```bash
# Pull image in parallel (faster than sequential)
docker-compose pull

# Cache pulled images
docker pull my-registry.com/backend:latest

# Speed up builds with BuildKit
DOCKER_BUILDKIT=1 docker build -t my-app .
```

### Pro Tip #5: Database Query Performance

```sql
-- Check slow queries
SELECT * FROM mysql.slow_log LIMIT 10;

-- Create indexes for frequent queries
CREATE INDEX idx_user_email ON users(email);
CREATE INDEX idx_post_created ON posts(created_at);

-- Verify indexes are used
EXPLAIN SELECT * FROM users WHERE email = 'user@example.com';
```

### Monitor Resource Usage in Real-Time

```bash
# View real-time resource usage
docker stats

# Specific container
docker stats three-tier-backend

# Log stats to file
docker stats --no-stream > resource_usage.log
```

---

## Advanced Techniques & Pro Tips

### Pro Tip #1: Multi-Host Docker with Overlay Networks

```bash
# Initialize Docker Swarm
docker swarm init

# Create overlay network for multi-host communication
docker network create -d overlay three-tier-network --attachable

# Deploy stack
docker stack deploy -c docker-compose.yml three-tier-app
```

### Pro Tip #2: Container Orchestration with Kubernetes

Convert Docker Compose to Kubernetes (Kompose):

```bash
# Install Kompose
curl -L https://github.com/kubernetes/kompose/releases/download/v1.28.0/kompose-linux-amd64 -o kompose
chmod +x kompose

# Convert docker-compose.yml to Kubernetes manifests
./kompose convert -f docker-compose.yml -o kubernetes/

# Deploy to Kubernetes
kubectl apply -f kubernetes/
```

### Pro Tip #3: Persistent Data Backup

```bash
# Backup database volume
docker exec three-tier-database mysqldump -u app_user -p$DB_PASSWORD myapp_db > backup.sql

# Restore database
docker exec -i three-tier-database mysql -u app_user -p$DB_PASSWORD myapp_db < backup.sql

# Backup named volume
docker run --rm -v db-data:/data -v $(pwd):/backup \
  alpine tar czf /backup/db-data-backup.tar.gz -C /data .

# Restore named volume
docker volume create db-data-restore
docker run --rm -v db-data-restore:/data -v $(pwd):/backup \
  alpine tar xzf /backup/db-data-backup.tar.gz -C /data
```

### Pro Tip #4: Logging Best Practices

```yaml
services:
  backend:
    logging:
      driver: "json-file"
      options:
        max-size: "10m"      # Rotate log after 10MB
        max-file: "3"        # Keep only 3 log files
        labels: "service=backend"

  db:
    logging:
      driver: "splunk"       # Send to Splunk
      options:
        splunk-token: "${SPLUNK_TOKEN}"
        splunk-url: "https://splunk.example.com"
```

### Pro Tip #5: Container Restart Policies

```yaml
services:
  backend:
    restart: unless-stopped   # Restart unless manually stopped
    
  db:
    restart: always           # Always restart (even after reboot)
    
  frontend:
    restart: on-failure       # Restart only on failure
    restart: on-failure:5     # Restart max 5 times

  worker:
    restart: no               # Never restart automatically
```

### Pro Tip #6: Build Optimization with BuildKit

```bash
# Enable BuildKit
export DOCKER_BUILDKIT=1

# Build with cache mount for dependencies
docker build --build-arg BUILDKIT_INLINE_CACHE=1 -t my-app .

# Parallel builds in BuildKit
cat > .dockerignore << 'EOF'
node_modules
.git
.npm
EOF
```

### Pro Tip #7: Development vs Production

**Development (docker-compose.yml)**:
```yaml
services:
  backend:
    build: ./backend
    volumes:
      - ./backend:/app      # Hot reload
    environment:
      - NODE_ENV=development
    ports:
      - "5000:5000"
    command: npm run dev    # Watch mode
```

**Production (docker-compose.prod.yml)**:
```yaml
services:
  backend:
    image: my-registry/backend:1.0.0  # Pre-built image
    environment:
      - NODE_ENV=production
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: '1'
```

**Run with different config**:
```bash
# Development
docker-compose up

# Production
docker-compose -f docker-compose.prod.yml up -d
```

---

## Common Practices & Anti-Patterns

### ✓ GOOD PRACTICES

**1. One Process Per Container**
```dockerfile
# ✓ Good: Each service has single responsibility
# frontend container: serves frontend
# backend container: runs Node.js API
# db container: runs MySQL only
```

**2. Use Health Checks**
```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost/health"]
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 40s
```

**3. Proper Signal Handling**
```dockerfile
# ✓ Use exec form (PID 1)
CMD ["node", "app.js"]

# ✗ Don't use shell form (PID > 1)
# CMD node app.js
```

**4. Version Everything**
```dockerfile
# ✓ Good: Specific versions
FROM node:18.17.0-alpine
RUN npm install express@4.18.2

# ✗ Bad: Latest versions (unpredictable)
FROM node:latest
RUN npm install express
```

**5. Immutable Infrastructure**
```bash
# Build once, use everywhere
docker build -t my-app:v1.0.0 .
docker tag my-app:v1.0.0 registry.example.com/my-app:v1.0.0
docker push registry.example.com/my-app:v1.0.0
```

### ❌ ANTI-PATTERNS TO AVOID

**1. Running as Root**
```dockerfile
# ✗ Bad
USER root
CMD ["node", "app.js"]

# ✓ Good
USER nodejs
CMD ["node", "app.js"]
```

**2. Hardcoding Secrets**
```dockerfile
# ✗ NEVER do this
ENV DATABASE_PASSWORD=supersecret123

# ✓ Use secrets or environment files
ENV DATABASE_PASSWORD_FILE=/run/secrets/db_password
```

**3. Large Image Sizes**
```dockerfile
# ✗ 800MB+ images (bloated)
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y nodejs npm build-essential

# ✓ Optimized (150MB)
FROM node:18-alpine
RUN npm ci --only=production
```

**4. Storing Data in Container**
```dockerfile
# ✗ Bad: Data lost when container deleted
RUN mkdir /data && echo "important data" > /data/file.txt

# ✓ Good: Use volumes
volumes:
  - app-data:/data
```

**5. No Health Checks**
```yaml
# ✗ Bad: Orchestrator doesn't know service status
services:
  backend:
    build: ./backend

# ✓ Good: Enable health monitoring
services:
  backend:
    build: ./backend
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5000/health"]
```

**6. Exposing Unnecessary Ports**
```yaml
# ✗ Bad: Database exposed to world
db:
  ports:
    - "3306:3306"

# ✓ Good: Only expose through backend
db:
  ports:
    - "3306:3306"  # Only on localhost
  networks:
    - app-network  # Isolated network
```

---

## Troubleshooting & Debugging

### Pro Command #1: View Container Logs

```bash
# Real-time logs
docker-compose logs -f backend

# View last 100 lines
docker-compose logs --tail=100 backend

# Logs with timestamps
docker-compose logs --timestamps backend

# Follow all services
docker-compose logs -f
```

### Pro Command #2: Execute Commands in Container

```bash
# Interactive bash shell
docker-compose exec backend /bin/sh

# Run specific command
docker-compose exec db mysql -u app_user -p$DB_PASSWORD -e "SELECT * FROM users;"

# Run with no TTY (for scripts)
docker-compose exec -T backend npm run migration
```

### Pro Command #3: Network Debugging

```bash
# Test DNS resolution
docker-compose exec backend nslookup db

# Test connectivity
docker-compose exec backend curl http://db:3306

# View network configuration
docker network inspect three-tier-app-network

# Ping between containers
docker-compose exec backend ping db
```

### Pro Command #4: Inspect Images

```bash
# View image layers
docker history my-app:latest

# Detailed image info
docker inspect my-app:latest

# Check image size
docker images my-app

# Get image digest
docker inspect --format='{{.RepoDigests}}' my-app:latest
```

### Pro Command #5: Check Container Resources

```bash
# Real-time resource usage
docker stats

# Memory usage
docker stats --format "table {{.Container}}\t{{.MemUsage}}"

# CPU usage
docker stats --format "table {{.Container}}\t{{.CPUPerc}}"
```

### Common Issues & Solutions

**Issue: "Connection refused" from frontend to backend**
```bash
# Check if backend is running
docker-compose ps

# Check backend logs
docker-compose logs backend

# Test connectivity
docker-compose exec frontend curl http://backend:5000/health
```

**Issue: "MySQL Connection Error"**
```bash
# Check if database is running
docker-compose ps db

# Check database logs
docker-compose logs db

# Wait for health check
docker-compose exec db mysqladmin ping

# Verify credentials in backend
docker-compose exec backend env | grep DB_
```

**Issue: "Out of Disk Space"**
```bash
# Check disk usage
docker system df

# Remove unused images
docker image prune -a

# Remove unused volumes
docker volume prune

# Remove everything unused
docker system prune -a --volumes
```

**Issue: "Slow Container Startup"**
```bash
# View layer history
docker history my-app

# Check for large layers
docker history --no-trunc my-app

# Use Dive to analyze
dive my-app:latest
```

### Pro Debugging Dockerfile

```dockerfile
# Add debug layer during development
FROM node:18-alpine AS debug
# Install debugging tools
RUN apk add --no-cache curl wget vim

# In production, use clean base
FROM node:18-alpine
COPY --from=debug /app /app
```

---

## Complete Working Example

### Project Structure
```
three-tier-app/
├── frontend/
│   ├── Dockerfile
│   ├── .dockerignore
│   ├── package.json
│   ├── src/
│   └── nginx.conf
├── backend/
│   ├── Dockerfile
│   ├── .dockerignore
│   ├── package.json
│   ├── app.js
│   └── healthcheck.js
├── database/
│   ├── my.cnf
│   └── init-scripts/
│       └── 01-init.sql
├── docker-compose.yml
└── .env
```

### Quick Start Commands

```bash
# Clone and setup
git clone <repo>
cd three-tier-app

# Create environment
cp .env.example .env
# Edit .env with your settings

# Build and start
docker-compose build
docker-compose up -d

# View logs
docker-compose logs -f

# Access application
# Frontend: http://localhost
# API: http://localhost/api

# Stop everything
docker-compose down
```

---

## Summary

**Key Takeaways**:

1. **Multi-stage builds** reduce image size by 80%+
2. **Alpine images** are much smaller than standard images
3. **Health checks** enable proper orchestration
4. **Non-root users** significantly improve security
5. **Docker networks** provide automatic DNS resolution
6. **Docker Compose** simplifies multi-container management
7. **Resource limits** prevent resource exhaustion
8. **Secrets management** keeps sensitive data secure
9. **Volume persistence** ensures data survives container restarts
10. **Proper logging** enables troubleshooting and monitoring

Following these practices will result in a production-ready, secure, scalable three-tier application!

---