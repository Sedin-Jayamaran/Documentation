# Complete Docker Guide: From Basics to Production

## Table of Contents

1. [Introduction to Docker](#introduction-to-docker)
2. [What is Docker?](#what-is-docker)
3. [How Docker Works - Architecture Deep Dive](#how-docker-works-architecture-deep-dive)
4. [Docker vs Virtual Machines](#docker-vs-virtual-machines)
5. [Why Docker is Preferred](#why-docker-is-preferred)
6. [Core Docker Concepts](#core-docker-concepts)
7. [Installing Docker](#installing-docker)
8. [Docker Basics - Essential Commands](#docker-basics-essential-commands)
9. [Docker Images Explained](#docker-images-explained)
10. [Docker Containers Explained](#docker-containers-explained)
11. [Dockerfile - Complete Guide](#dockerfile-complete-guide)
12. [How to Dockerize Any Application](#how-to-dockerize-any-application)
13. [Docker Volumes - Data Persistence](#docker-volumes-data-persistence)
14. [Docker Networks](#docker-networks)
15. [Docker Compose - Complete Guide](#docker-compose-complete-guide)
16. [Why Docker Compose](#why-docker-compose)
17. [Multi-Container Applications with Docker Compose](#multi-container-applications-with-docker-compose)
18. [Best Practices](#best-practices)
19. [Real-World Examples](#real-world-examples)
20. [Docker Security](#docker-security)
21. [Troubleshooting](#troubleshooting)
22. [Docker in Production](#docker-in-production)

---

## Introduction to Docker

Docker has revolutionized the way we build, ship, and run applications. It's the most popular containerization platform that enables developers to package applications with all their dependencies into standardized units called containers.

**Created:** 2013 by Solomon Hykes at dotCloud (now Docker Inc.)  
**Written in:** Go (Golang)  
**Type:** Containerization Platform  
**License:** Apache License 2.0

Docker solves the age-old problem: **"It works on my machine!"** by ensuring applications run consistently across different environments.

---

## What is Docker?

Docker is an open-source platform that automates the deployment, scaling, and management of applications using containerization technology. It packages applications and their dependencies into containers that can run on any system with Docker installed.

### Key Characteristics

- **Lightweight**: Containers share the host OS kernel, making them much lighter than VMs
- **Portable**: Run anywhere - laptop, cloud, data center
- **Consistent**: Same behavior in development, testing, and production
- **Fast**: Starts in seconds (vs minutes for VMs)
- **Isolated**: Each container runs independently
- **Scalable**: Easy to scale horizontally

### The Container Analogy

Think of Docker containers like shipping containers:
- **Standardized**: All containers have the same interface
- **Portable**: Can be moved between ships, trucks, and trains
- **Isolated**: Contents don't interfere with each other
- **Stackable**: Can be efficiently stacked and managed

---

## How Docker Works - Architecture Deep Dive

Docker follows a **client-server architecture** with three main components working together.

### Docker Architecture Diagram

```
┌────────────────────────────────────────────────────┐
│                Docker Client (CLI)                  │
│            $ docker run nginx                       │
│            $ docker build -t myapp .                │
│            $ docker ps                              │
└────────────────┬───────────────────────────────────┘
                 │ (REST API / Unix Socket)
                 ↓
┌────────────────────────────────────────────────────┐
│              Docker Host (Docker Daemon)            │
│  ┌──────────────────────────────────────────────┐  │
│  │          Docker Daemon (dockerd)              │  │
│  │  - Manages containers, images, networks      │  │
│  │  - Listens to Docker API requests            │  │
│  │  - Communicates with containerd              │  │
│  └──────────────┬───────────────────────────────┘  │
│                 ↓                                   │
│  ┌──────────────────────────────────────────────┐  │
│  │          Containerd (Container Runtime)       │  │
│  │  - Manages container lifecycle               │  │
│  │  - Image management                          │  │
│  │  - Uses runc to run containers               │  │
│  └──────────────┬───────────────────────────────┘  │
│                 ↓                                   │
│  ┌──────────────────────────────────────────────┐  │
│  │  Containers  │  Images  │  Networks │ Volumes│  │
│  └──────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────┘
                 ↑
                 │ Pull/Push Images
                 ↓
┌────────────────────────────────────────────────────┐
│           Docker Registry (Docker Hub)             │
│        Public/Private Image Repository             │
└────────────────────────────────────────────────────┘
```

### 1. Docker Client

**What it is:**
- The primary interface for users to interact with Docker
- Command-line tool (`docker` CLI)
- Sends commands to Docker daemon via REST API

**What it does:**
```bash
# Client commands
docker build     # Build images
docker run       # Run containers
docker pull      # Download images
docker push      # Upload images
docker ps        # List containers
```

**Communication:**
- Uses REST API over Unix socket (`/var/run/docker.sock`) on Linux
- Can communicate with remote Docker daemons
- Uses TLS for secure remote communication

### 2. Docker Daemon (dockerd)

**What it is:**
- Background service running on the host
- Core component that does the heavy lifting
- Manages all Docker objects

**Responsibilities:**
1. **Listen** to API requests from Docker client
2. **Build** container images from Dockerfiles
3. **Manage** images, containers, networks, and volumes
4. **Execute** container operations
5. **Communicate** with other daemons for Docker services

**How it works:**
```
User Command → Docker Client → REST API → Docker Daemon → Containerd → Container
```

### 3. Docker Registry (Docker Hub)

**What it is:**
- Storage and distribution system for Docker images
- Docker Hub is the default public registry
- Can run private registries

**Functions:**
- **Store** Docker images
- **Version** images with tags
- **Share** images publicly or privately
- **Scan** images for vulnerabilities

### 4. Containerd

**What it is:**
- High-level container runtime
- Industry-standard core container runtime
- Manages complete container lifecycle

**Responsibilities:**
- Image transfer and storage
- Container execution and supervision
- Network attachments
- Low-level storage operations

### 5. Docker Objects

**Images:**
- Read-only templates with instructions for creating containers
- Built from Dockerfiles
- Stored in layers for efficiency

**Containers:**
- Runnable instances of images
- Isolated from other containers and host
- Defined by image + configuration

**Networks:**
- Enable communication between containers
- Provide isolation
- Multiple network drivers available

**Volumes:**
- Persist data generated by containers
- Shared between containers
- Independent of container lifecycle

### How a Container is Created

**Step-by-step process when you run `docker run nginx`:**

```
1. Docker Client
   ↓
   Sends 'run' command to Docker Daemon
   
2. Docker Daemon
   ↓
   Checks if 'nginx' image exists locally
   
3. If not found locally
   ↓
   Pulls image from Docker Hub Registry
   
4. Image Downloaded
   ↓
   Stores in local image cache
   
5. Docker Daemon
   ↓
   Creates a new container from the image
   
6. Containerd
   ↓
   Allocates read-write filesystem layer
   
7. Containerd
   ↓
   Creates network interface (bridge network)
   ↓
   Assigns IP address to container
   
8. Containerd
   ↓
   Executes command specified in image (start nginx)
   
9. Container Running
   ↓
   Application runs in isolated environment
```

### Linux Kernel Features Docker Uses

Docker leverages Linux kernel features for containerization:

**1. Namespaces** (Isolation)
- **PID namespace**: Process isolation
- **NET namespace**: Network isolation
- **MNT namespace**: Mount points isolation
- **UTS namespace**: Hostname isolation
- **IPC namespace**: Inter-process communication isolation
- **USER namespace**: User ID isolation

**2. Control Groups (cgroups)** (Resource Limiting)
- CPU usage limits
- Memory limits
- Disk I/O limits
- Network bandwidth limits

**3. Union File Systems** (Layered Images)
- OverlayFS (most common)
- Allows multiple layers to appear as single filesystem
- Efficient storage and fast container startup

**Example of how namespaces work:**
```bash
# Container sees only its processes
Container PID 1 = nginx
Container PID 2 = worker process

# Host sees actual PIDs
Host PID 12345 = nginx (in container)
Host PID 12346 = worker process (in container)
```

---

## Docker vs Virtual Machines

Understanding the difference between Docker containers and virtual machines is crucial.

### Architecture Comparison

```
┌──────────────────────────────────┐  ┌──────────────────────────────────┐
│      VIRTUAL MACHINES            │  │      DOCKER CONTAINERS           │
│                                  │  │                                  │
│  ┌────────┐  ┌────────┐         │  │  ┌────────┐  ┌────────┐         │
│  │  App A │  │  App B │         │  │  │  App A │  │  App B │         │
│  ├────────┤  ├────────┤         │  │  ├────────┤  ├────────┤         │
│  │  Bins/ │  │  Bins/ │         │  │  │  Bins/ │  │  Bins/ │         │
│  │  Libs  │  │  Libs  │         │  │  │  Libs  │  │  Libs  │         │
│  ├────────┤  ├────────┤         │  │  └────┬───┘  └────┬───┘         │
│  │Guest OS│  │Guest OS│         │  │       │           │             │
│  │(Linux) │  │(Win)   │         │  │       └─────┬─────┘             │
│  └────┬───┘  └────┬───┘         │  │             ↓                   │
│       │           │             │  │      Docker Engine              │
│       └─────┬─────┘             │  │                                  │
│             ↓                   │  │                                  │
│        Hypervisor               │  │                                  │
│    (VMware, VirtualBox)         │  │                                  │
│             │                   │  │             │                   │
│             ↓                   │  │             ↓                   │
│       Host OS (Linux/Win)       │  │       Host OS (Linux)           │
│             │                   │  │             │                   │
│             ↓                   │  │             ↓                   │
│    Physical Hardware            │  │    Physical Hardware            │
└──────────────────────────────────┘  └──────────────────────────────────┘
      Heavyweight (GBs)                    Lightweight (MBs)
      Boot in minutes                      Boot in seconds
```

### Detailed Comparison

| Feature | Virtual Machine | Docker Container |
|---------|----------------|------------------|
| **OS** | Full guest OS per VM | Shares host OS kernel |
| **Size** | Gigabytes (1-10 GB+) | Megabytes (10-500 MB) |
| **Startup Time** | Minutes | Seconds |
| **Performance** | Limited performance | Near-native performance |
| **Isolation** | Complete isolation | Process-level isolation |
| **Resource Usage** | High (dedicated resources) | Low (shared resources) |
| **Portability** | Less portable | Highly portable |
| **Number per Host** | Dozens | Hundreds/Thousands |
| **Managed by** | Hypervisor | Docker Engine |
| **Security** | More secure (full isolation) | Less secure (shared kernel) |
| **Use Case** | Different OS requirements | Microservices, apps |

### When to Use Virtual Machines

✅ **Use VMs when:**
- You need complete OS isolation
- Running different operating systems (Windows + Linux)
- Legacy applications requiring specific OS versions
- Maximum security isolation is required
- Graphical desktop environment needed
- Testing OS-level changes

### When to Use Docker Containers

✅ **Use Docker when:**
- Building microservices architectures
- Rapid development and deployment needed
- Scaling applications horizontally
- CI/CD pipelines
- Consistent environments (dev, staging, prod)
- Resource efficiency is important
- Fast startup times required

### Can You Use Both?

**Yes!** Common pattern:
```
Physical Server
  └── Virtual Machine (for isolation)
      └── Docker Engine
          └── Multiple Containers (for efficiency)
```

---

## Why Docker is Preferred

Docker has become the standard for modern application deployment. Here's why:

### 1. Consistency Across Environments

**Problem it solves:**
```
Developer: "It works on my machine!"
Operations: "It doesn't work in production!"
```

**Docker solution:**
```dockerfile
# Same container runs everywhere
FROM node:18
COPY . /app
RUN npm install
CMD ["node", "app.js"]
```

The exact same container runs identically on:
- Developer's laptop (Mac/Windows/Linux)
- QA/Test environments
- Staging environment
- Production servers
- Cloud platforms (AWS, Azure, GCP)

### 2. Fast and Lightweight

**Comparison:**

| Operation | Virtual Machine | Docker Container |
|-----------|----------------|------------------|
| Start | 1-2 minutes | 1-2 seconds |
| Stop | 30-60 seconds | < 1 second |
| Size | 1-10 GB | 10-500 MB |
| Density | 10-20 per host | 100-1000+ per host |

**Real-world impact:**
```bash
# Start 10 VMs: ~10 minutes
# Start 10 containers: ~10 seconds

# Scale application from 3 to 100 instances
# VMs: Not practical
# Containers: Minutes
```

### 3. Version Control for Infrastructure

**Track everything:**
```bash
# Dockerfile is code - version it with Git
git commit -m "Update Node.js version to 18"
git commit -m "Add Redis caching layer"
git commit -m "Optimize multi-stage build"

# Roll back easily
git checkout previous-version
docker build -t myapp:rollback .
```

### 4. Microservices Architecture

**Perfect for microservices:**
```yaml
# docker-compose.yml
services:
  frontend:
    image: myapp/frontend
  backend:
    image: myapp/backend
  database:
    image: postgres:15
  cache:
    image: redis:7
  queue:
    image: rabbitmq:3
```

Each service:
- Runs independently
- Can be scaled separately
- Uses different technology stacks
- Can be updated without affecting others

### 5. Isolation and Security

**Process isolation:**
```bash
# Each container is isolated
Container 1: Node.js app
Container 2: Python app  
Container 3: Java app

# No conflicts between:
- Dependencies
- Library versions
- Port bindings
- File systems
```

### 6. Efficient Resource Utilization

**Better resource usage:**
```
Single Server (32GB RAM, 8 CPUs)

VMs approach:
- 4 VMs (each 8GB RAM, 2 CPUs)
- 50% resource utilization

Docker approach:
- 50+ containers
- 90%+ resource utilization
- Better cost efficiency
```

### 7. Rapid Deployment and Scaling

**Deploy instantly:**
```bash
# Deploy new version
docker pull myapp:v2.0
docker stop myapp-v1
docker run myapp:v2.0

# Scale horizontally
docker service scale myapp=10  # Run 10 instances

# Load balancer distributes traffic
```

### 8. DevOps and CI/CD Integration

**Perfect for automation:**
```yaml
# .gitlab-ci.yml
build:
  script:
    - docker build -t myapp .
    - docker push myapp

test:
  script:
    - docker run myapp npm test

deploy:
  script:
    - docker-compose up -d
```

### 9. Cloud Agnostic

**Run anywhere:**
```
Local Development → Docker
↓
Testing → Docker
↓
Staging → Docker on AWS
↓
Production → Docker on:
  - AWS ECS/EKS
  - Azure ACI/AKS  
  - Google Cloud Run/GKE
  - On-premises Kubernetes
  - Any cloud provider
```

### 10. Simplified Dependency Management

**No more dependency hell:**
```
Without Docker:
❌ Install Node.js
❌ Install Python
❌ Install MySQL
❌ Configure everything
❌ Version conflicts

With Docker:
✅ docker-compose up
✅ Everything runs
```

### 11. Cost Efficiency

**Real-world savings:**
```
Company running 100 applications:

VMs:
- 100 VMs × $50/month = $5,000/month
- Plus management overhead

Docker:
- 10 servers × $100/month = $1,000/month
- 80% cost reduction
```

### 12. Developer Productivity

**Faster onboarding:**
```bash
# Traditional setup: Days
Install tools, configure environment, debug issues...

# Docker setup: Minutes
git clone repo
docker-compose up
# Start coding!
```

---

## Core Docker Concepts

Before diving into practical usage, understand these fundamental concepts.

### 1. Docker Image

**What is it?**
- Read-only template containing application code and dependencies
- Blueprint for creating containers
- Made up of layers (each instruction in Dockerfile creates a layer)
- Immutable (cannot be changed once built)

**Image Structure:**
```
myapp:latest
├── Layer 5: Application code (2 MB)
├── Layer 4: Dependencies (50 MB)
├── Layer 3: Python packages (100 MB)
├── Layer 2: Python runtime (150 MB)
└── Layer 1: Base OS (Ubuntu) (200 MB)
Total: 502 MB (but layers are shared!)
```

**Key points:**
- Images are stored in registries (Docker Hub, private registries)
- Tagged for versioning (e.g., `nginx:1.21`, `nginx:latest`)
- Layered architecture enables efficient storage
- Shared layers save disk space

### 2. Docker Container

**What is it?**
- Running instance of an image
- Isolated process with its own filesystem, networking, and process tree
- Ephemeral by default (data lost when deleted)
- Can be started, stopped, moved, and deleted

**Container Lifecycle:**
```
docker create  → Container Created (not running)
       ↓
docker start   → Container Running
       ↓
docker stop    → Container Stopped (can be restarted)
       ↓
docker rm      → Container Deleted (gone forever)
```

**Key points:**
- Containers are isolated from each other
- Lightweight and fast to start/stop
- Share host OS kernel
- Can be connected via networks
- Can persist data via volumes

### 3. Dockerfile

**What is it?**
- Text file with instructions to build a Docker image
- Defines base image, dependencies, and configuration
- Version-controlled with your code
- Automated and repeatable

**Basic structure:**
```dockerfile
# Choose base image
FROM node:18

# Set working directory
WORKDIR /app

# Copy application files
COPY package*.json ./
COPY . .

# Install dependencies
RUN npm install

# Expose port
EXPOSE 3000

# Start application
CMD ["node", "server.js"]
```

### 4. Docker Registry

**What is it?**
- Storage and distribution system for Docker images
- Similar to Git repositories but for images

**Types:**
- **Docker Hub**: Default public registry
- **Private registries**: Self-hosted or cloud (AWS ECR, Azure ACR, GCP GCR)
- **GitHub Container Registry**
- **GitLab Container Registry**

### 5. Docker Volume

**What is it?**
- Persistent storage for containers
- Survives container deletion
- Shared between containers

**Types:**
```bash
# Named volume (managed by Docker)
docker volume create mydata
docker run -v mydata:/app/data myapp

# Bind mount (host directory)
docker run -v /host/path:/container/path myapp

# Anonymous volume
docker run -v /app/data myapp
```

### 6. Docker Network

**What is it?**
- Virtual network enabling container communication
- Provides isolation and security

**Network types:**
```bash
# Bridge (default): Containers on same host
# Host: Container uses host network
# None: No networking
# Custom bridge: User-defined networks
```

---

## Installing Docker

### Installation on Windows

#### Prerequisites
- Windows 10/11 Pro, Enterprise, or Education (64-bit)
- Hyper-V and Containers Windows features enabled
- BIOS virtualization enabled

#### Method 1: Docker Desktop (Recommended)

**Step 1: Download Docker Desktop**
1. Visit: https://www.docker.com/products/docker-desktop
2. Click "Download for Windows"
3. Download `Docker Desktop Installer.exe`

**Step 2: Install Docker Desktop**
```powershell
# Run installer
.\Docker Desktop Installer.exe

# Or from command line
Start-Process 'Docker Desktop Installer.exe' -Wait install
```

**Step 3: Configure Docker Desktop**
1. Accept Service Agreement
2. Choose configuration:
   - ✅ Use WSL 2 instead of Hyper-V (recommended)
   - ✅ Add shortcut to desktop

**Step 4: Complete Installation**
1. Click "Close and restart"
2. System will restart
3. Docker Desktop starts automatically

**Step 5: Verify Installation**
```powershell
# Check Docker version
docker --version
# Output: Docker version 24.0.6, build ed223bc

# Check Docker Compose version
docker compose version
# Output: Docker Compose version v2.23.0

# Run test container
docker run hello-world
```

#### Method 2: Using WSL 2 (Windows Subsystem for Linux)

**Step 1: Enable WSL 2**
```powershell
# Open PowerShell as Administrator
wsl --install

# Set WSL 2 as default
wsl --set-default-version 2
```

**Step 2: Install Ubuntu from Microsoft Store**
```powershell
# Install Ubuntu
wsl --install -d Ubuntu

# Launch Ubuntu
wsl
```

**Step 3: Install Docker in WSL**
```bash
# Update packages
sudo apt-get update

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add user to docker group
sudo usermod -aG docker $USER

# Start Docker service
sudo service docker start
```

### Installation on Ubuntu/Linux

#### Method 1: Using Official Repository (Recommended)

**Step 1: Set up Docker Repository**
```bash
# Update package index
sudo apt-get update

# Install prerequisites
sudo apt-get install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

# Add Docker's official GPG key
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
    sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Set up repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

**Step 2: Install Docker Engine**
```bash
# Update package index
sudo apt-get update

# Install Docker Engine, containerd, and Docker Compose
sudo apt-get install -y \
    docker-ce \
    docker-ce-cli \
    containerd.io \
    docker-buildx-plugin \
    docker-compose-plugin
```

**Step 3: Verify Installation**
```bash
# Check Docker version
docker --version

# Check Docker service status
sudo systemctl status docker

# Run test container
sudo docker run hello-world
```

**Step 4: Post-Installation Steps**
```bash
# Create docker group (if doesn't exist)
sudo groupadd docker

# Add user to docker group (run without sudo)
sudo usermod -aG docker $USER

# Apply group changes
newgrp docker

# Enable Docker to start on boot
sudo systemctl enable docker.service
sudo systemctl enable containerd.service

# Verify you can run without sudo
docker run hello-world
```

#### Method 2: Convenience Script

```bash
# Download and run Docker installation script
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add user to docker group
sudo usermod -aG docker $USER

# Log out and back in for group changes
```

### Installation on macOS

#### Using Docker Desktop (Recommended)

**Step 1: Download Docker Desktop**
1. Visit: https://www.docker.com/products/docker-desktop
2. Click "Download for Mac"
3. Choose:
   - Apple Silicon (M1/M2/M3) → `Docker.dmg` (ARM64)
   - Intel Chip → `Docker.dmg` (AMD64)

**Step 2: Install Docker Desktop**
```bash
# Open the downloaded .dmg file
# Drag Docker icon to Applications folder
# Open Docker from Applications

# Or install via command line
sudo hdiutil attach Docker.dmg
sudo /Volumes/Docker/Docker.app/Contents/MacOS/install
sudo hdiutil detach /Volumes/Docker
```

**Step 3: Launch Docker Desktop**
1. Open Docker from Applications
2. Accept Service Agreement
3. Docker will start in menu bar
4. Wait for "Docker Desktop is running" message

**Step 4: Verify Installation**
```bash
# Check version
docker --version
docker compose version

# Run test container
docker run hello-world
```

#### Using Homebrew

```bash
# Install Docker
brew install --cask docker

# Launch Docker Desktop
open /Applications/Docker.app

# Verify installation
docker --version
```

### Verify Docker Installation (All Platforms)

```bash
# 1. Check Docker version
docker --version

# 2. Check Docker info
docker info

# 3. Run hello-world container
docker run hello-world

# 4. Check Docker Compose
docker compose version

# 5. List running containers
docker ps

# 6. List all containers
docker ps -a

# 7. List images
docker images
```

**Expected output for hello-world:**
```
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

---

## Docker Basics - Essential Commands

Master these essential Docker commands to work effectively with containers.

### Docker Image Commands

```bash
# Pull image from registry
docker pull nginx
docker pull nginx:1.21
docker pull python:3.11-alpine

# List images
docker images
docker image ls

# Build image from Dockerfile
docker build -t myapp:v1 .
docker build -t myapp:latest -f Dockerfile.prod .

# Tag image
docker tag myapp:v1 myapp:latest
docker tag myapp:v1 myregistry.com/myapp:v1

# Push image to registry
docker push myregistry.com/myapp:v1

# Remove image
docker rmi nginx
docker rmi myapp:v1
docker image rm myapp:v1

# Remove unused images
docker image prune

# Remove all images
docker rmi $(docker images -q)

# Inspect image details
docker inspect nginx
docker history nginx

# Save image to tar file
docker save myapp:v1 > myapp.tar
docker save -o myapp.tar myapp:v1

# Load image from tar file
docker load < myapp.tar
docker load -i myapp.tar
```

### Docker Container Commands

```bash
# Run container
docker run nginx
docker run -d nginx                    # Detached mode
docker run -d --name web nginx         # With name
docker run -d -p 8080:80 nginx         # Port mapping
docker run -d -p 8080:80 --name web nginx

# Run with environment variables
docker run -d -e "DB_HOST=localhost" myapp

# Run with volume
docker run -d -v mydata:/app/data myapp

# Run interactive container
docker run -it ubuntu bash
docker run -it --rm ubuntu bash        # Remove after exit

# List running containers
docker ps
docker container ls

# List all containers
docker ps -a
docker ps -a -q                        # Only IDs

# Start stopped container
docker start web
docker start <container-id>

# Stop container
docker stop web
docker stop <container-id>

# Restart container
docker restart web

# Pause container
docker pause web

# Unpause container
docker unpause web

# Kill container (force stop)
docker kill web

# Remove container
docker rm web
docker rm -f web                       # Force remove

# Remove all stopped containers
docker container prune

# Remove all containers
docker rm -f $(docker ps -aq)

# View container logs
docker logs web
docker logs -f web                     # Follow logs
docker logs --tail 100 web             # Last 100 lines
docker logs --since 10m web            # Last 10 minutes

# Execute command in running container
docker exec web ls /app
docker exec -it web bash               # Interactive shell
docker exec -it web sh                 # For Alpine images

# Inspect container
docker inspect web
docker inspect --format '{{.NetworkSettings.IPAddress}}' web

# View container stats
docker stats
docker stats web

# View container processes
docker top web

# Copy files to/from container
docker cp file.txt web:/app/
docker cp web:/app/log.txt ./

# Export container filesystem
docker export web > web.tar

# View container port mappings
docker port web

# Attach to running container
docker attach web
```

### Docker Network Commands

```bash
# List networks
docker network ls

# Create network
docker network create mynetwork
docker network create --driver bridge mynetwork

# Inspect network
docker network inspect mynetwork

# Connect container to network
docker network connect mynetwork web

# Disconnect container from network
docker network disconnect mynetwork web

# Remove network
docker network rm mynetwork

# Remove unused networks
docker network prune
```

### Docker Volume Commands

```bash
# List volumes
docker volume ls

# Create volume
docker volume create mydata

# Inspect volume
docker volume inspect mydata

# Remove volume
docker volume rm mydata

# Remove unused volumes
docker volume prune

# Remove all volumes
docker volume rm $(docker volume ls -q)
```

### Docker System Commands

```bash
# Show Docker disk usage
docker system df

# Show detailed disk usage
docker system df -v

# Remove unused data (containers, networks, images)
docker system prune

# Remove all unused data including volumes
docker system prune -a --volumes

# Show Docker info
docker info

# Show Docker version
docker version
```

### Docker Compose Commands

```bash
# Start services
docker compose up
docker compose up -d                   # Detached mode
docker compose up --build              # Rebuild images

# Stop services
docker compose down
docker compose down -v                 # Remove volumes

# View logs
docker compose logs
docker compose logs -f                 # Follow logs
docker compose logs web                # Specific service

# List services
docker compose ps

# Execute command in service
docker compose exec web bash

# Build services
docker compose build
docker compose build --no-cache

# Pull images
docker compose pull

# Start specific service
docker compose start web

# Stop specific service
docker compose stop web

# Restart services
docker compose restart

# Scale service
docker compose up -d --scale web=3
```

---

## Docker Images Explained

Docker images are the foundation of containers. Understanding them is crucial.

### What is a Docker Image?

A Docker image is a **read-only template** that contains:
- Application code
- Runtime environment
- System tools and libraries
- Environment variables
- Configuration files

### Image Layers

Images are built using a **layered architecture**:

```
Image: myapp:latest (Total: 500 MB)

┌─────────────────────────────────┐
│ Layer 5: CMD ["node", "app.js"] │ ← 0 bytes (metadata)
├─────────────────────────────────┤
│ Layer 4: COPY . /app            │ ← 10 MB (app code)
├─────────────────────────────────┤
│ Layer 3: RUN npm install        │ ← 150 MB (dependencies)
├─────────────────────────────────┤
│ Layer 2: WORKDIR /app           │ ← 0 bytes (metadata)
├─────────────────────────────────┤
│ Layer 1: FROM node:18           │ ← 340 MB (base OS + Node)
└─────────────────────────────────┘
```

**Key benefits of layers:**
1. **Efficiency**: Layers are cached and reused
2. **Speed**: Only changed layers are rebuilt
3. **Storage**: Shared layers save disk space
4. **Distribution**: Only new layers are downloaded

### Image Naming Convention

```
[registry-host]/[namespace]/[repository]:[tag]

Examples:
docker.io/library/nginx:latest        # Docker Hub official
docker.io/mycompany/myapp:v1.0       # Docker Hub private
ghcr.io/username/myapp:latest         # GitHub Container Registry
gcr.io/project/myapp:v2.0            # Google Container Registry
myregistry.com:5000/myapp:latest      # Private registry
```

**Components:**
- **registry-host**: Where image is stored (optional, defaults to Docker Hub)
- **namespace**: User or organization name
- **repository**: Image name
- **tag**: Version identifier (defaults to `latest`)

### Working with Images

#### Pulling Images

```bash
# Pull latest version
docker pull nginx

# Pull specific version
docker pull nginx:1.21

# Pull from specific registry
docker pull gcr.io/my-project/myapp:v1

# Pull all tags
docker pull --all-tags nginx
```

#### Building Images

```bash
# Build from Dockerfile
docker build -t myapp:v1 .

# Build with specific Dockerfile
docker build -f Dockerfile.prod -t myapp:prod .

# Build with build arguments
docker build --build-arg VERSION=1.0 -t myapp:v1 .

# Build without cache
docker build --no-cache -t myapp:v1 .

# Build with target stage (multi-stage)
docker build --target production -t myapp:prod .
```

#### Inspecting Images

```bash
# View image details
docker inspect nginx

# View image layers
docker history nginx

# View image size
docker images nginx
```

#### Tagging Images

```bash
# Tag image with new name
docker tag myapp:v1 myapp:latest

# Tag for different registry
docker tag myapp:v1 myregistry.com/myapp:v1
```

---

## Docker Containers Explained

Containers are running instances of Docker images.

### Container Lifecycle

```
┌─────────────────────────────────────────────────┐
│           Container Lifecycle                    │
└─────────────────────────────────────────────────┘

        docker create
             ↓
        [ Created ]
             ↓ docker start
        [ Running ] ←──────┐
             ↓              │
             ├─ docker pause → [ Paused ]
             │                      ↓ docker unpause
             │                      ↓
             ├────────────────────────┘
             ↓ docker stop
        [ Stopped ] ←──────┐
             ↓ docker start │
             └───────────────┘
             ↓ docker rm
        [ Deleted ]
```

### Creating and Running Containers

#### Basic Run

```bash
# Run container (create + start)
docker run nginx

# This is equivalent to:
docker create nginx    # Create container
docker start <id>      # Start container
```

#### Common Run Options

```bash
# Detached mode (background)
docker run -d nginx

# With custom name
docker run -d --name webserver nginx

# Port mapping (host:container)
docker run -d -p 8080:80 nginx

# Multiple port mappings
docker run -d -p 8080:80 -p 8443:443 nginx

# Publish all exposed ports
docker run -d -P nginx

# Environment variables
docker run -d -e DB_HOST=localhost -e DB_PORT=5432 myapp

# Volume mount
docker run -d -v /host/data:/container/data myapp

# Network
docker run -d --network mynetwork nginx

# Restart policy
docker run -d --restart always nginx
docker run -d --restart unless-stopped nginx

# Resource limits
docker run -d --memory 512m --cpus 1.5 nginx

# Remove container after exit
docker run --rm -it ubuntu bash

# Interactive terminal
docker run -it ubuntu bash

# Read-only container
docker run -d --read-only nginx
```

### Container States

**Running:**
```bash
# Container is actively running
docker ps
```

**Stopped:**
```bash
# Container exists but not running
docker ps -a
```

**Paused:**
```bash
# Container processes are suspended
docker pause <container>
```

**Exited:**
```bash
# Container finished execution
docker ps -a --filter status=exited
```

### Managing Container Data

#### Temporary Data (Container Layer)

```bash
# Data written to container's writable layer
# Lost when container is removed
docker run -d nginx
docker exec <container> sh -c "echo 'test' > /tmp/file.txt"
docker rm -f <container>  # file.txt is lost
```

#### Persistent Data (Volumes)

```bash
# Data persists after container removal
docker run -d -v mydata:/app/data myapp
docker rm -f <container>  # Data in mydata volume remains
```

---

## Dockerfile - Complete Guide

A Dockerfile is a text file containing instructions to build a Docker image.

### Dockerfile Structure

```dockerfile
# Comments start with #

# 1. Base Image (Required - must be first)
FROM node:18-alpine

# 2. Metadata
LABEL maintainer="[email protected]"
LABEL version="1.0"
LABEL description="My Node.js application"

# 3. Environment Variables
ENV NODE_ENV=production
ENV APP_PORT=3000

# 4. Working Directory
WORKDIR /app

# 5. Copy Files
COPY package*.json ./
COPY . .

# 6. Run Commands
RUN npm install --production
RUN npm cache clean --force

# 7. Expose Ports
EXPOSE 3000

# 8. User (for security)
USER node

# 9. Volumes
VOLUME ["/app/data"]

# 10. Startup Command
CMD ["node", "server.js"]
```

### Dockerfile Instructions

#### FROM - Base Image

Sets the base image for subsequent instructions.

```dockerfile
# Official image
FROM node:18

# Specific version
FROM node:18.16.0

# Alpine variant (smaller)
FROM node:18-alpine

# Multi-platform
FROM --platform=linux/amd64 node:18

# Using ARG for dynamic base
ARG BASE_IMAGE=node:18
FROM ${BASE_IMAGE}
```

#### WORKDIR - Set Working Directory

Sets the working directory for subsequent instructions.

```dockerfile
# Set working directory
WORKDIR /app

# Relative paths now resolve to /app
COPY package.json .  # Copies to /app/package.json
RUN npm install      # Runs in /app
```

#### COPY - Copy Files

Copies files from host to image.

```dockerfile
# Copy single file
COPY package.json /app/

# Copy directory
COPY src/ /app/src/

# Copy with wildcard
COPY *.json /app/

# Copy and rename
COPY config.prod.json /app/config.json

# Copy with ownership
COPY --chown=node:node . /app

# Multi-stage copy
COPY --from=builder /app/dist /app/dist
```

#### ADD - Copy Files (Advanced)

Like COPY but with additional features.

```dockerfile
# Copy file
ADD file.txt /app/

# Auto-extract tar archives
ADD archive.tar.gz /app/

# Download from URL (not recommended)
ADD https://example.com/file.txt /app/

# Best practice: Use COPY unless you need ADD features
```

#### RUN - Execute Commands

Executes commands during build.

```dockerfile
# Shell form (runs in /bin/sh)
RUN npm install

# Exec form (direct execution)
RUN ["npm", "install"]

# Multiple commands (inefficient - creates multiple layers)
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get clean

# Multiple commands (efficient - single layer)
RUN apt-get update && \
    apt-get install -y curl && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# With variables
RUN export VAR=value && \
    echo $VAR > file.txt
```

#### CMD - Default Command

Sets default command when container starts.

```dockerfile
# Exec form (recommended)
CMD ["node", "server.js"]

# Shell form
CMD node server.js

# Providing arguments to ENTRYPOINT
CMD ["--port", "3000"]

# Only one CMD per Dockerfile (last one wins)
```

#### ENTRYPOINT - Configure Container as Executable

Sets the command that always runs.

```dockerfile
# Exec form
ENTRYPOINT ["node"]

# Combined with CMD for arguments
ENTRYPOINT ["node"]
CMD ["server.js"]
# Runs: node server.js

# Override CMD at runtime
# docker run myapp app.js
# Runs: node app.js
```

**CMD vs ENTRYPOINT:**
```dockerfile
# CMD - Can be overridden
CMD ["node", "server.js"]
# docker run myapp
# Runs: node server.js
# docker run myapp python app.py
# Runs: python app.py

# ENTRYPOINT - Always runs
ENTRYPOINT ["node"]
CMD ["server.js"]
# docker run myapp
# Runs: node server.js
# docker run myapp app.js
# Runs: node app.js (ENTRYPOINT stays, CMD overridden)
```

#### EXPOSE - Document Ports

Documents which ports the application uses.

```dockerfile
# Single port
EXPOSE 3000

# Multiple ports
EXPOSE 3000 8080

# UDP port
EXPOSE 5353/udp

# Note: EXPOSE doesn't actually publish ports
# Use -p flag when running container
```

#### ENV - Set Environment Variables

Sets environment variables.

```dockerfile
# Single variable
ENV NODE_ENV=production

# Multiple variables
ENV NODE_ENV=production \
    APP_PORT=3000 \
    DB_HOST=localhost

# Using variables
ENV APP_HOME=/app
WORKDIR ${APP_HOME}
```

#### ARG - Build Arguments

Defines build-time variables.

```dockerfile
# Define argument
ARG NODE_VERSION=18

# Use argument
FROM node:${NODE_VERSION}

# With default value
ARG BUILD_DATE=unknown
LABEL build-date=${BUILD_DATE}

# Pass at build time
# docker build --build-arg NODE_VERSION=20 .
```

#### VOLUME - Create Mount Point

Creates a mount point for volumes.

```dockerfile
# Single volume
VOLUME /app/data

# Multiple volumes
VOLUME ["/app/data", "/app/logs"]

# Note: Actual volume created at runtime
# docker run -v mydata:/app/data myapp
```

#### USER - Set User

Sets the user for subsequent instructions.

```dockerfile
# Create user and switch
RUN addgroup -S appgroup && \
    adduser -S appuser -G appgroup

USER appuser

# All subsequent RUN, CMD, ENTRYPOINT run as appuser

# Switch back to root
USER root

# Use numeric UID/GID
USER 1000:1000
```

#### HEALTHCHECK - Health Checking

Defines health check command.

```dockerfile
# HTTP health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1

# TCP health check
HEALTHCHECK --interval=30s --timeout=3s \
  CMD nc -z localhost 3000 || exit 1

# Custom script
HEALTHCHECK CMD /app/healthcheck.sh

# Disable health check
HEALTHCHECK NONE
```

#### SHELL - Default Shell

Changes the default shell.

```dockerfile
# Use bash instead of sh
SHELL ["/bin/bash", "-c"]

# Use PowerShell on Windows
SHELL ["powershell", "-Command"]
```

#### ONBUILD - Trigger for Child Images

Adds triggers for when image is used as base.

```dockerfile
# In base image
ONBUILD COPY . /app
ONBUILD RUN npm install

# When someone uses: FROM mybase
# These ONBUILD commands execute automatically
```

#### STOPSIGNAL - Signal to Stop

Sets signal to stop container.

```dockerfile
# Default is SIGTERM
STOPSIGNAL SIGTERM

# Use SIGKILL
STOPSIGNAL SIGKILL
```

### Multi-Stage Builds

Build efficient images by using multiple stages.

```dockerfile
# Stage 1: Build stage
FROM node:18 AS builder

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .
RUN npm run build

# Stage 2: Production stage
FROM node:18-alpine

WORKDIR /app

# Copy only necessary files from builder
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY package*.json ./

USER node

EXPOSE 3000

CMD ["node", "dist/server.js"]
```

**Benefits:**
- Smaller final image
- Separates build dependencies from runtime
- More secure (no build tools in production)

### Best Practices

```dockerfile
# ✅ Good Dockerfile Example

# 1. Use specific version tags
FROM node:18.16.0-alpine

# 2. Create non-root user first
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# 3. Set working directory
WORKDIR /app

# 4. Copy dependency files first (better caching)
COPY --chown=appuser:appgroup package*.json ./

# 5. Install dependencies
RUN npm ci --only=production && \
    npm cache clean --force

# 6. Copy application code
COPY --chown=appuser:appgroup . .

# 7. Switch to non-root user
USER appuser

# 8. Expose port
EXPOSE 3000

# 9. Add health check
HEALTHCHECK --interval=30s --timeout=3s \
  CMD node healthcheck.js

# 10. Define startup command
CMD ["node", "server.js"]
```

---

## How to Dockerize Any Application

Step-by-step guide to containerize any application.

### Step 1: Analyze Your Application

**Questions to answer:**
1. What runtime does it need? (Node.js, Python, Java, etc.)
2. What dependencies does it have?
3. What files need to be copied?
4. What ports does it listen on?
5. What environment variables does it need?
6. Does it need persistent storage?

### Step 2: Choose Base Image

```dockerfile
# Choose appropriate base image

# Node.js applications
FROM node:18-alpine

# Python applications
FROM python:3.11-slim

# Java applications
FROM openjdk:17-jdk-alpine

# Go applications
FROM golang:1.21-alpine AS builder
FROM alpine:latest  # For final stage

# PHP applications
FROM php:8.2-apache

# Ruby applications
FROM ruby:3.2-alpine

# .NET applications
FROM mcr.microsoft.com/dotnet/aspnet:8.0
```

### Step 3: Create Dockerfile

Let's dockerize different types of applications.

#### Example 1: Node.js Application

**Application structure:**
```
my-node-app/
├── package.json
├── package-lock.json
├── server.js
├── src/
│   ├── routes/
│   ├── controllers/
│   └── models/
└── Dockerfile
```

**Dockerfile:**
```dockerfile
# Use Node.js LTS Alpine (smallest size)
FROM node:18-alpine

# Set working directory
WORKDIR /app

# Copy dependency files
COPY package*.json ./

# Install production dependencies only
RUN npm ci --only=production && \
    npm cache clean --force

# Copy application code
COPY . .

# Create non-root user
RUN addgroup -S nodejs && \
    adduser -S nodejs -G nodejs && \
    chown -R nodejs:nodejs /app

# Switch to non-root user
USER nodejs

# Expose application port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s \
  CMD node -e "require('http').get('http://localhost:3000/health', (r) => {process.exit(r.statusCode === 200 ? 0 : 1)})"

# Start application
CMD ["node", "server.js"]
```

**Build and run:**
```bash
# Build image
docker build -t my-node-app:v1 .

# Run container
docker run -d \
  --name node-app \
  -p 3000:3000 \
  -e NODE_ENV=production \
  my-node-app:v1

# Check logs
docker logs -f node-app

# Test application
curl http://localhost:3000
```

#### Example 2: Python Flask Application

**Application structure:**
```
my-flask-app/
├── requirements.txt
├── app.py
├── config.py
├── templates/
├── static/
└── Dockerfile
```

**requirements.txt:**
```
Flask==3.0.0
gunicorn==21.2.0
python-dotenv==1.0.0
```

**Dockerfile:**
```dockerfile
# Use Python slim image
FROM python:3.11-slim

# Set working directory
WORKDIR /app

# Install system dependencies
RUN apt-get update && \
    apt-get install -y --no-install-recommends gcc && \
    rm -rf /var/lib/apt/lists/*

# Copy and install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Create non-root user
RUN useradd -m -u 1000 appuser && \
    chown -R appuser:appuser /app

USER appuser

# Expose port
EXPOSE 5000

# Environment variables
ENV FLASK_APP=app.py
ENV PYTHONUNBUFFERED=1

# Health check
HEALTHCHECK --interval=30s --timeout=3s \
  CMD python -c "import requests; requests.get('http://localhost:5000/health')"

# Run with gunicorn
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "4", "app:app"]
```

**Build and run:**
```bash
# Build image
docker build -t my-flask-app:v1 .

# Run container
docker run -d \
  --name flask-app \
  -p 5000:5000 \
  -e FLASK_ENV=production \
  -v $(pwd)/config:/app/config \
  my-flask-app:v1
```

#### Example 3: Java Spring Boot Application

**Application structure:**
```
my-spring-app/
├── pom.xml
├── src/
│   └── main/
│       ├── java/
│       └── resources/
├── target/
│   └── myapp-1.0.0.jar
└── Dockerfile
```

**Multi-stage Dockerfile:**
```dockerfile
# Stage 1: Build stage
FROM maven:3.9-eclipse-temurin-17 AS builder

WORKDIR /app

# Copy pom.xml and download dependencies (cached layer)
COPY pom.xml .
RUN mvn dependency:go-offline

# Copy source and build
COPY src ./src
RUN mvn clean package -DskipTests

# Stage 2: Runtime stage
FROM eclipse-temurin:17-jre-alpine

WORKDIR /app

# Copy JAR from builder stage
COPY --from=builder /app/target/*.jar app.jar

# Create non-root user
RUN addgroup -S spring && \
    adduser -S spring -G spring && \
    chown spring:spring app.jar

USER spring

EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget --no-verbose --tries=1 --spider http://localhost:8080/actuator/health || exit 1

# JVM options
ENV JAVA_OPTS="-Xmx512m -Xms256m"

# Run application
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

**Build and run:**
```bash
# Build image
docker build -t my-spring-app:v1 .

# Run container
docker run -d \
  --name spring-app \
  -p 8080:8080 \
  -e SPRING_PROFILES_ACTIVE=prod \
  -e JAVA_OPTS="-Xmx1g -Xms512m" \
  my-spring-app:v1
```

#### Example 4: React Frontend Application

**Application structure:**
```
my-react-app/
├── package.json
├── public/
├── src/
├── nginx.conf
└── Dockerfile
```

**nginx.conf:**
```nginx
server {
    listen 80;
    server_name localhost;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /api {
        proxy_pass http://backend:3000;
    }
}
```

**Multi-stage Dockerfile:**
```dockerfile
# Stage 1: Build stage
FROM node:18-alpine AS builder

WORKDIR /app

# Copy dependency files
COPY package*.json ./

# Install dependencies
RUN npm ci

# Copy source code
COPY . .

# Build application
RUN npm run build

# Stage 2: Production stage
FROM nginx:alpine

# Copy built files from builder
COPY --from=builder /app/build /usr/share/nginx/html

# Copy nginx configuration
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Expose port
EXPOSE 80

# Health check
HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget --no-verbose --tries=1 --spider http://localhost/health || exit 1

# nginx runs as non-root by default in alpine
CMD ["nginx", "-g", "daemon off;"]
```

**Build and run:**
```bash
# Build image
docker build -t my-react-app:v1 .

# Run container
docker run -d \
  --name react-app \
  -p 3000:80 \
  my-react-app:v1
```

#### Example 5: Full Stack Application (Python + PostgreSQL + Redis)

**Application structure:**
```
fullstack-app/
├── backend/
│   ├── app.py
│   ├── requirements.txt
│   └── Dockerfile
├── frontend/
│   ├── src/
│   ├── package.json
│   └── Dockerfile
└── docker-compose.yml
```

**Backend Dockerfile:**
```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

USER nobody

EXPOSE 5000

CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:app"]
```

**Frontend Dockerfile:**
```dockerfile
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/build /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  backend:
    build: ./backend
    ports:
      - "5000:5000"
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/mydb
      - REDIS_URL=redis://cache:6379
    depends_on:
      - db
      - cache
    networks:
      - app-network

  frontend:
    build: ./frontend
    ports:
      - "3000:80"
    depends_on:
      - backend
    networks:
      - app-network

  db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
      - POSTGRES_DB=mydb
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - app-network

  cache:
    image: redis:7-alpine
    networks:
      - app-network

networks:
  app-network:

volumes:
  postgres-data:
```

**Run full stack:**
```bash
docker-compose up -d
```

### Step 4: Optimize Dockerfile

**Optimization techniques:**

```dockerfile
# ✅ Optimize layer caching (copy dependencies first)
COPY package*.json ./
RUN npm install
COPY . .  # App code changes more frequently

# ✅ Use .dockerignore
# Create .dockerignore file
node_modules
npm-debug.log
.git
.env
*.md

# ✅ Minimize layers (combine RUN commands)
RUN apt-get update && \
    apt-get install -y package1 package2 && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# ✅ Use multi-stage builds
FROM node:18 AS builder
# ... build steps
FROM node:18-alpine
COPY --from=builder /app/dist ./dist

# ✅ Choose smaller base images
FROM node:18-alpine  # Instead of node:18 (saves 900MB)

# ✅ Run as non-root user
USER node

# ✅ Use specific version tags
FROM node:18.16.0-alpine  # Not just 'node:latest'
```

### Step 5: Build and Test

```bash
# Build image
docker build -t myapp:v1 .

# Check image size
docker images myapp:v1

# Run container
docker run -d --name test-app -p 8080:8080 myapp:v1

# Test application
curl http://localhost:8080

# Check logs
docker logs test-app

# Execute shell in container
docker exec -it test-app sh

# Check processes
docker top test-app

# Monitor resources
docker stats test-app

# Clean up
docker stop test-app
docker rm test-app
```

---

## Docker Volumes - Data Persistence

Containers are ephemeral by default. Volumes provide persistent storage.

### Why Volumes?

**Problem without volumes:**
```bash
docker run -d --name db postgres
# Data stored in container

docker rm -f db
# All data lost!
```

**Solution with volumes:**
```bash
docker run -d --name db -v dbdata:/var/lib/postgresql/data postgres
# Data stored in volume

docker rm -f db
# Container deleted, data remains in dbdata volume!
```

### Types of Volumes

#### 1. Named Volumes (Recommended)

Managed by Docker, stored in Docker's storage area.

```bash
# Create volume
docker volume create mydata

# Use volume
docker run -d -v mydata:/app/data myapp

# List volumes
docker volume ls

# Inspect volume
docker volume inspect mydata

# Output:
[
    {
        "Name": "mydata",
        "Driver": "local",
        "Mountpoint": "/var/lib/docker/volumes/mydata/_data"
    }
]

# Remove volume
docker volume rm mydata

# Remove all unused volumes
docker volume prune
```

#### 2. Bind Mounts

Mount host directory into container.

```bash
# Mount host directory
docker run -d \
  -v /host/path:/container/path \
  myapp

# Mount current directory
docker run -d \
  -v $(pwd):/app \
  myapp

# Read-only mount
docker run -d \
  -v $(pwd):/app:ro \
  myapp

# Example: Development with live reload
docker run -d \
  -v $(pwd)/src:/app/src \
  -p 3000:3000 \
  my-dev-app
```

#### 3. tmpfs Mounts

Temporary filesystem in memory (Linux only).

```bash
# Create tmpfs mount
docker run -d \
  --tmpfs /app/cache:rw,size=100m \
  myapp

# Data stored in RAM, lost on container stop
```

### Volume Operations

```bash
# Create volume with driver options
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=192.168.1.1,rw \
  --opt device=:/path/to/dir \
  nfsvolume

# Backup volume data
docker run --rm \
  -v mydata:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/mydata-backup.tar.gz /data

# Restore volume data
docker run --rm \
  -v mydata:/data \
  -v $(pwd):/backup \
  alpine tar xzf /backup/mydata-backup.tar.gz -C /

# Copy data between volumes
docker run --rm \
  -v source-volume:/source \
  -v dest-volume:/dest \
  alpine sh -c "cp -r /source/* /dest/"

# Inspect volume size
docker system df -v
```

### Practical Volume Examples

#### Database Persistence

```bash
# PostgreSQL with persistent data
docker run -d \
  --name postgres \
  -v pgdata:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret \
  postgres:15

# MySQL with persistent data
docker run -d \
  --name mysql \
  -v mysqldata:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=secret \
  mysql:8

# MongoDB with persistent data
docker run -d \
  --name mongo \
  -v mongodata:/data/db \
  mongo:7
```

#### Development with Live Reload

```bash
# Node.js development
docker run -d \
  --name dev-server \
  -v $(pwd):/app \
  -v /app/node_modules \
  -p 3000:3000 \
  node:18 \
  npm run dev

# Python development
docker run -it \
  --name python-dev \
  -v $(pwd):/app \
  -w /app \
  python:3.11 \
  python app.py
```

---

## Docker Networks

Docker networks enable container communication.

### Network Types

#### 1. Bridge Network (Default)

Containers on same host can communicate.

```bash
# Create bridge network
docker network create mynetwork

# Run containers on network
docker run -d --name web --network mynetwork nginx
docker run -d --name api --network mynetwork myapi

# Containers can reach each other by name
# web can access: http://api:3000
# api can access: http://web:80
```

#### 2. Host Network

Container uses host's network directly.

```bash
# Run with host network
docker run -d --network host nginx

# Container binds directly to host ports
# No port mapping needed
```

#### 3. None Network

No networking.

```bash
# Run with no network
docker run -d --network none myapp

# Completely isolated
```

#### 4. Custom Bridge Network

User-defined bridge with DNS.

```bash
# Create network
docker network create \
  --driver bridge \
  --subnet 172.18.0.0/16 \
  --gateway 172.18.0.1 \
  mybridge

# Run containers
docker run -d --name web --network mybridge nginx
docker run -d --name db --network mybridge postgres

# Automatic DNS resolution
# web can connect to: postgresql://db:5432
```

### Network Operations

```bash
# List networks
docker network ls

# Inspect network
docker network inspect mynetwork

# Connect container to network
docker network connect mynetwork mycontainer

# Disconnect container from network
docker network disconnect mynetwork mycontainer

# Remove network
docker network rm mynetwork

# Remove unused networks
docker network prune
```

### Practical Network Examples

#### Multi-Container Application

```bash
# Create network
docker network create app-network

# Run database
docker run -d \
  --name postgres \
  --network app-network \
  -e POSTGRES_PASSWORD=secret \
  postgres:15

# Run backend
docker run -d \
  --name backend \
  --network app-network \
  -e DATABASE_URL=postgresql://postgres:secret@postgres:5432/mydb \
  mybackend

# Run frontend
docker run -d \
  --name frontend \
  --network app-network \
  -p 3000:80 \
  myfrontend

# frontend → backend → postgres (all by name)
```

---

## Docker Compose - Complete Guide

Docker Compose simplifies multi-container application management.

### What is Docker Compose?

Docker Compose is a tool for defining and running multi-container Docker applications using a YAML file.

**Key benefits:**
- Define entire application stack in one file
- Single command to start/stop everything
- Easy environment management
- Service dependencies
- Networking and volumes handled automatically

### Installing Docker Compose

**Included with Docker Desktop** (Windows/Mac)

**Linux:**
```bash
# Install Docker Compose V2 (plugin)
sudo apt-get update
sudo apt-get install docker-compose-plugin

# Verify installation
docker compose version
```

### Docker Compose File Structure

**Basic structure (`docker-compose.yml`):**

```yaml
version: '3.8'

services:
  # Service definitions

networks:
  # Network definitions

volumes:
  # Volume definitions
```

### Compose File Example

**Simple web application:**

```yaml
version: '3.8'

services:
  # Web server
  web:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html
    networks:
      - frontend

  # Application server
  app:
    build: ./app
    environment:
      - DATABASE_URL=postgresql://db:5432/mydb
    depends_on:
      - db
    networks:
      - frontend
      - backend

  # Database
  db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_PASSWORD=secret
    volumes:
      - dbdata:/var/lib/postgresql/data
    networks:
      - backend

networks:
  frontend:
  backend:

volumes:
  dbdata:
```

### Service Configuration Options

```yaml
services:
  myapp:
    # Use existing image
    image: nginx:alpine
    
    # Or build from Dockerfile
    build:
      context: ./app
      dockerfile: Dockerfile
      args:
        - VERSION=1.0
    
    # Container name
    container_name: myapp-container
    
    # Port mappings
    ports:
      - "8080:80"
      - "8443:443"
    
    # Expose ports (internal only)
    expose:
      - "3000"
    
    # Environment variables
    environment:
      - NODE_ENV=production
      - API_KEY=secret
    
    # Environment file
    env_file:
      - .env
      - .env.prod
    
    # Volumes
    volumes:
      - ./data:/app/data
      - myvolume:/app/storage
      - /app/node_modules
    
    # Networks
    networks:
      - frontend
      - backend
    
    # Dependencies
    depends_on:
      - db
      - cache
    
    # Restart policy
    restart: unless-stopped
    
    # Command override
    command: npm start
    
    # Entrypoint override
    entrypoint: /app/entrypoint.sh
    
    # Health check
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    
    # Resource limits
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 512M
        reservations:
          cpus: '0.5'
          memory: 256M
    
    # Logging
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

### Docker Compose Commands

```bash
# Start services
docker compose up

# Start in detached mode
docker compose up -d

# Build images before starting
docker compose up --build

# Start specific services
docker compose up web db

# Stop services
docker compose down

# Stop and remove volumes
docker compose down -v

# View logs
docker compose logs

# Follow logs
docker compose logs -f

# Logs for specific service
docker compose logs web

# List running services
docker compose ps

# Execute command in service
docker compose exec web sh

# Run one-off command
docker compose run web npm test

# Build/rebuild services
docker compose build
docker compose build --no-cache

# Pull images
docker compose pull

# Push images
docker compose push

# Pause services
docker compose pause

# Unpause services
docker compose unpause

# Start services
docker compose start

# Stop services
docker compose stop

# Restart services
docker compose restart

# Scale service
docker compose up -d --scale web=3

# Validate compose file
docker compose config

# View running processes
docker compose top
```

---

## Why Docker Compose

Understanding when and why to use Docker Compose.

### Problems Docker Compose Solves

#### Problem 1: Complex Commands

**Without Compose:**
```bash
# Start database
docker run -d --name db \
  --network mynet \
  -e POSTGRES_PASSWORD=secret \
  -v dbdata:/var/lib/postgresql/data \
  postgres:15

# Start cache
docker run -d --name cache \
  --network mynet \
  redis:7

# Start backend
docker run -d --name backend \
  --network mynet \
  -e DATABASE_URL=postgresql://db:5432/mydb \
  -e REDIS_URL=redis://cache:6379 \
  -p 3000:3000 \
  mybackend

# Start frontend
docker run -d --name frontend \
  --network mynet \
  -p 8080:80 \
  myfrontend

# Too many commands!
```

**With Compose:**
```yaml
# docker-compose.yml
version: '3.8'

services:
  db:
    image: postgres:15
    environment:
      - POSTGRES_PASSWORD=secret
    volumes:
      - dbdata:/var/lib/postgresql/data

  cache:
    image: redis:7

  backend:
    image: mybackend
    environment:
      - DATABASE_URL=postgresql://db:5432/mydb
      - REDIS_URL=redis://cache:6379
    ports:
      - "3000:3000"
    depends_on:
      - db
      - cache

  frontend:
    image: myfrontend
    ports:
      - "8080:80"
    depends_on:
      - backend

volumes:
  dbdata:
```

```bash
# Start everything
docker compose up -d

# One command!
```

#### Problem 2: Service Dependencies

**Docker Compose manages dependencies:**
```yaml
services:
  frontend:
    depends_on:
      - backend
      
  backend:
    depends_on:
      - db
      - cache
      
  db:
    image: postgres
    
  cache:
    image: redis
```

Start order: `db` → `cache` → `backend` → `frontend`

#### Problem 3: Multiple Environments

**Same app, different environments:**

```yaml
# docker-compose.yml (base)
services:
  app:
    image: myapp
    ports:
      - "3000:3000"
```

```yaml
# docker-compose.override.yml (development)
services:
  app:
    build: .
    volumes:
      - ./src:/app/src
    environment:
      - NODE_ENV=development
```

```yaml
# docker-compose.prod.yml (production)
services:
  app:
    image: myapp:v1.0
    environment:
      - NODE_ENV=production
    deploy:
      replicas: 3
```

```bash
# Development
docker compose up -d

# Production
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

---

## Multi-Container Applications with Docker Compose

Real-world examples of complete applications.

### Example 1: WordPress with MySQL

```yaml
version: '3.8'

services:
  db:
    image: mysql:8
    command: '--default-authentication-plugin=mysql_native_password'
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: wordpresspass
    volumes:
      - db_data:/var/lib/mysql
    restart: always
    networks:
      - wordpress-network

  wordpress:
    image: wordpress:latest
    depends_on:
      - db
    ports:
      - "8000:80"
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: wordpresspass
      WORDPRESS_DB_NAME: wordpress
    volumes:
      - wordpress_data:/var/www/html
    restart: always
    networks:
      - wordpress-network

networks:
  wordpress-network:

volumes:
  db_data:
  wordpress_data:
```

```bash
# Start WordPress
docker compose up -d

# Visit http://localhost:8000
```

### Example 2: MEAN Stack (MongoDB + Express + Angular + Node.js)

```yaml
version: '3.8'

services:
  mongodb:
    image: mongo:7
    container_name: mongodb
    environment:
      - MONGO_INITDB_ROOT_USERNAME=admin
      - MONGO_INITDB_ROOT_PASSWORD=secret
    volumes:
      - mongo_data:/data/db
    networks:
      - mean-network
    restart: always

  backend:
    build: ./backend
    container_name: backend
    ports:
      - "3000:3000"
    environment:
      - MONGODB_URI=mongodb://admin:secret@mongodb:27017/mydb?authSource=admin
      - JWT_SECRET=mysecret
    depends_on:
      - mongodb
    networks:
      - mean-network
    restart: always
    volumes:
      - ./backend:/app
      - /app/node_modules

  frontend:
    build: ./frontend
    container_name: frontend
    ports:
      - "4200:80"
    depends_on:
      - backend
    networks:
      - mean-network
    restart: always

networks:
  mean-network:

volumes:
  mongo_data:
```

### Example 3: Microservices Architecture

```yaml
version: '3.8'

services:
  # API Gateway
  gateway:
    build: ./gateway
    ports:
      - "80:80"
    depends_on:
      - auth-service
      - user-service
      - product-service
    networks:
      - microservices

  # Authentication Service
  auth-service:
    build: ./services/auth
    environment:
      - JWT_SECRET=secret
      - REDIS_URL=redis://cache:6379
    depends_on:
      - cache
    networks:
      - microservices

  # User Service
  user-service:
    build: ./services/user
    environment:
      - DATABASE_URL=postgresql://postgres:secret@userdb:5432/users
    depends_on:
      - userdb
    networks:
      - microservices

  # Product Service
  product-service:
    build: ./services/product
    environment:
      - DATABASE_URL=postgresql://postgres:secret@productdb:5432/products
    depends_on:
      - productdb
    networks:
      - microservices

  # User Database
  userdb:
    image: postgres:15-alpine
    environment:
      - POSTGRES_PASSWORD=secret
      - POSTGRES_DB=users
    volumes:
      - userdb_data:/var/lib/postgresql/data
    networks:
      - microservices

  # Product Database
  productdb:
    image: postgres:15-alpine
    environment:
      - POSTGRES_PASSWORD=secret
      - POSTGRES_DB=products
    volumes:
      - productdb_data:/var/lib/postgresql/data
    networks:
      - microservices

  # Redis Cache
  cache:
    image: redis:7-alpine
    networks:
      - microservices

  # Message Queue
  rabbitmq:
    image: rabbitmq:3-management-alpine
    ports:
      - "15672:15672"
    networks:
      - microservices

networks:
  microservices:

volumes:
  userdb_data:
  productdb_data:
```

---

## Best Practices

### Dockerfile Best Practices

1. **Use specific base image versions**
```dockerfile
# ✅ Good
FROM node:18.16.0-alpine

# ❌ Bad
FROM node:latest
```

2. **Minimize layers**
```dockerfile
# ✅ Good (single layer)
RUN apt-get update && \
    apt-get install -y curl wget && \
    apt-get clean

# ❌ Bad (multiple layers)
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y wget
```

3. **Use .dockerignore**
```.dockerignore
node_modules
npm-debug.log
.git
.env
*.md
.DS_Store
```

4. **Leverage build cache**
```dockerfile
# ✅ Good (dependencies cached)
COPY package*.json ./
RUN npm install
COPY . .

# ❌ Bad (cache broken often)
COPY . .
RUN npm install
```

5. **Run as non-root user**
```dockerfile
RUN adduser -D appuser
USER appuser
```

6. **Use multi-stage builds**
```dockerfile
FROM node:18 AS builder
# Build steps

FROM node:18-alpine
COPY --from=builder /app/dist ./dist
```

### Docker Compose Best Practices

1. **Version control compose files**
2. **Use environment variables**
3. **Define networks explicitly**
4. **Use named volumes**
5. **Set restart policies**
6. **Define health checks**
7. **Use depends_on wisely**
8. **Separate concerns (services)**

### Security Best Practices

1. **Scan images for vulnerabilities**
```bash
docker scan myapp:latest
```

2. **Don't run as root**
3. **Use secrets for sensitive data**
4. **Keep base images updated**
5. **Limit container capabilities**
6. **Use read-only filesystems when possible**
7. **Implement resource limits**

---

## Real-World Examples

### E-commerce Platform

Complete dockerized e-commerce platform with frontend, backend, database, cache, and message queue.

```yaml
version: '3.8'

services:
  frontend:
    build: ./frontend
    ports:
      - "3000:80"
    depends_on:
      - backend
    networks:
      - ecommerce

  backend:
    build: ./backend
    environment:
      - DATABASE_URL=postgresql://postgres:secret@db:5432/ecommerce
      - REDIS_URL=redis://cache:6379
      - RABBITMQ_URL=amqp://guest:guest@queue:5672
    depends_on:
      - db
      - cache
      - queue
    networks:
      - ecommerce
    restart: unless-stopped

  db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_PASSWORD=secret
      - POSTGRES_DB=ecommerce
    volumes:
      - db_data:/var/lib/postgresql/data
    networks:
      - ecommerce
    restart: unless-stopped

  cache:
    image: redis:7-alpine
    networks:
      - ecommerce
    restart: unless-stopped

  queue:
    image: rabbitmq:3-management-alpine
    ports:
      - "15672:15672"
    networks:
      - ecommerce
    restart: unless-stopped

  worker:
    build: ./worker
    environment:
      - RABBITMQ_URL=amqp://guest:guest@queue:5672
    depends_on:
      - queue
    networks:
      - ecommerce
    restart: unless-stopped

networks:
  ecommerce:

volumes:
  db_data:
```

---

## Docker Security

1. **Use official images**
2. **Scan for vulnerabilities**
3. **Don't store secrets in images**
4. **Use Docker secrets**
5. **Limit container resources**
6. **Enable user namespaces**
7. **Use AppArmor/SELinux**
8. **Regular updates**

---

## Troubleshooting

### Common Issues

1. **Port already in use**
```bash
# Find process using port
lsof -i :8080
# Kill process or use different port
```

2. **Permission denied**
```bash
# Add user to docker group
sudo usermod -aG docker $USER
```

3. **Out of disk space**
```bash
# Clean up
docker system prune -a --volumes
```

---

## Docker in Production

### Container Orchestration

For production, use:
- **Kubernetes** - Industry standard
- **Docker Swarm** - Simpler alternative
- **AWS ECS/EKS**
- **Azure AKS**
- **Google GKE**

### Monitoring

- Prometheus + Grafana
- ELK Stack (Elasticsearch, Logstash, Kibana)
- Datadog
- New Relic

---

## Conclusion

Docker has revolutionized application deployment by providing:
- **Consistency** across environments
- **Efficiency** in resource usage
- **Speed** in deployment
- **Isolation** for security
- **Scalability** for growth

Docker Compose simplifies multi-container applications by:
- **Defining** entire stacks in YAML
- **Managing** dependencies automatically
- **Enabling** environment-specific configurations
- **Simplifying** development workflows

**Master Docker and you'll be equipped for modern DevOps practices!**

---

**Document Version:** 1.0  
**Last Updated:** October 2025  
**Created For:** DevOps Engineers & Developers

For the latest Docker updates, visit: https://docs.docker.com/