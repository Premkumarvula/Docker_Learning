# Docker_Learning

# 🐳 Docker From Scratch — Mapped to AWS ECS

> **For Prem** — You already know AWS ECS (Task Definitions, Services, Fargate, ECR). This course teaches Docker from the ground up, constantly mapping every concept back to what you already know in ECS. Think of Docker as the *engine* and ECS as the *managed orchestrator* that runs that engine at scale.

---

## 🧭 Quick Mental Model: Docker ↔ ECS Mapping

| Docker Concept | AWS ECS Equivalent | What It Means |
|---|---|---|
| Dockerfile | Task Definition (container definition block) | Blueprint for building a container |
| Docker Image | Container Image in ECR | The packaged, runnable artifact |
| Docker Container | Running Task (ECS Task) | A running instance of an image |
| `docker run` | `RunTask` API / ECS starting a task | Launch a container |
| `docker stop` | ECS stopping a task | Gracefully stop a container |
| Docker Network | ECS Task Networking (awsvpc/bridge/host) | How containers communicate |
| Docker Volume | EFS Volume / Bind Mount in task def | Persistent storage |
| Docker Compose | ECS Service + Task Def + Load Balancer | Multi-container app definition |
| Docker Swarm / Kubernetes | ECS Service + Service Discovery | Container orchestration |
| `docker pull` | ECR image pull (happens automatically) | Download an image |
| `docker push` | `docker push` to ECR | Upload an image to a registry |
| Docker Hub | Amazon ECR | Image registry |
| Environment Variables | Container environment vars in task def | Config passed to container |
| Port Mapping (`-p`) | Host port / Container port in task def | Expose container ports |
| Resource Limits (`--cpus`, `--memory`) | Task CPU/Memory limits | Constrain resources |
| Health Check (`HEALTHCHECK`) | Health check in task def | Container health monitoring |
| `docker logs` | CloudWatch Logs (via awslogs driver) | Container log output |
| `docker exec` | ECS Exec (`execute-command`) | Run command inside running container |
| `docker build` | CI/CD pipeline building & pushing to ECR | Create an image from Dockerfile |
| `.dockerignore` | `.gitignore` (conceptually similar) | Exclude files from build context |
| Multi-stage build | CI/CD build stages | Optimize final image size |
| Docker daemon (`dockerd`) | ECS Agent (on EC2) / Fargate infrastructure | The runtime engine |

---

## 📦 Module 1: What Is Docker & Why It Exists

### 1.1 The Problem Before Docker

```
Traditional Deployment:
┌─────────────────────────────────────┐
│           Physical Server            │
│  ┌─────────┐  ┌─────────┐           │
│  │  App A   │  │  App B   │           │
│  │ Node 14  │  │ Node 18  │  CONFLICT!│
│  │ Ubuntu   │  │ CentOS   │           │
│  └─────────┘  └─────────┘           │
│  Shared OS, shared libs, dependency hell │
└─────────────────────────────────────┘

VM Approach:
┌─────────────────────────────────────┐
│           Physical Server            │
│  ┌──────────────┐  ┌──────────────┐  │
│  │  VM 1 (App A) │  │  VM 2 (App B) │  │
│  │  Guest OS     │  │  Guest OS     │  │
│  │  Node 14      │  │  Node 18      │  │
│  └──────────────┘  └──────────────┘  │
│  Hypervisor                          │
│  Heavy! Each VM = full OS overhead   │
└─────────────────────────────────────┘

Docker Approach:
┌─────────────────────────────────────┐
│           Physical Server            │
│  ┌──────┐  ┌──────┐  ┌──────┐      │
│  │App A │  │App B │  │App C │      │
│  │Bins/ │  │Bins/ │  │Bins/ │      │
│  │Libs  │  │Libs  │  │Libs  │      │
│  └──────┴──┴──────┴──┴──────┘      │
│  Docker Engine (shared Host OS)      │
│  Lightweight! Shares kernel         │
└─────────────────────────────────────┘
```

### 1.2 Containers vs VMs

| Aspect | VMs | Containers |
|--------|-----|------------|
| Isolation | Full OS isolation | Process-level isolation |
| Size | GBs | MBs |
| Boot time | Minutes | Seconds (or milliseconds) |
| Resource overhead | High (full OS) | Minimal (shared kernel) |
| Density | Few per host | Hundreds per host |
| Portability | Limited | Excellent (runs anywhere Docker runs) |

### 1.3 ECS Connection 🧠

> **You already use containers!** Every ECS task is a Docker container. When ECS Fargate launches a task, it's pulling a Docker image from ECR and running it as a container. You've been using Docker all along — ECS just manages it for you.

---

## 📦 Module 2: Docker Installation & Setup

### 2.1 Install Docker

```bash
# macOS (Docker Desktop)
brew install --cask docker

# Verify installation
docker --version
docker info

# Test with hello-world
docker run hello-world
```

### 2.2 Docker Architecture

```
┌──────────────────────────────────────────────┐
│                  Docker Architecture           │
│                                                │
│  ┌──────────┐       ┌──────────────────────┐  │
│  │  Docker   │──────▶│    Docker Daemon      │  │
│  │  Client   │ REST  │    (dockerd)          │  │
│  │  (CLI)    │ API   │                      │  │
│  └──────────┘       │  ┌────┐ ┌────┐       │  │
│                      │  │ C1 │ │ C2 │       │  │
│  ┌──────────┐       │  └────┘ └────┘       │  │
│  │  Docker   │◀─────│  ┌──────────────┐     │  │
│  │  Registry │ pull │  │  Images      │     │  │
│  │  (Hub/ECR)│      │  └──────────────┘     │  │
│  └──────────┘       └──────────────────────┘  │
└──────────────────────────────────────────────┘
```

**Key Components:**
- **Docker Client (CLI)**: The `docker` command you type → like `aws` CLI for ECS
- **Docker Daemon (dockerd)**: The background service that actually runs containers → like the ECS Agent on EC2 or Fargate infrastructure
- **Docker Registry**: Where images are stored → like ECR

### 2.3 ECS Connection 🧠

> In ECS, you never interact with the Docker daemon directly. ECS Agent (on EC2) or Fargate manages the daemon for you. Now you'll control the daemon yourself via the CLI.

---

## 📦 Module 3: Docker Images — The Building Blocks

### 3.1 What Is an Image?

An image is a **read-only template** with instructions for creating a container. It contains:
- Application code
- Runtime (Node.js, Python, Java)
- System tools
- Libraries
- Environment variables
- Default commands

```
Image Layers (Union File System):
┌─────────────────────┐
│  Application Code    │  ← Layer 4 (your app)
├─────────────────────┤
│  npm install output  │  ← Layer 3 (dependencies)
├─────────────────────┤
│  Node.js runtime     │  ← Layer 2 (from node:18 base)
├─────────────────────┤
│  Debian Linux        │  ← Layer 1 (base OS)
└─────────────────────┘
Each layer is cached → fast rebuilds!
```

### 3.2 Working with Images

```bash
# Search for images
docker search nginx
docker search node

# Pull an image (like ECR pull, but from Docker Hub)
docker pull nginx:latest
docker pull node:18-alpine
docker pull python:3.11-slim

# List local images
docker images
# REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
# nginx        latest    abc123def456   2 weeks ago    187MB

# Inspect an image
docker inspect nginx:latest

# Image tags (versions)
docker pull nginx:1.25        # specific version
docker pull nginx:alpine      # alpine variant (smaller)
docker pull nginx:1.25-alpine # specific version + alpine

# Remove an image
docker rmi nginx:latest

# Tag an image (like tagging for ECR)
docker tag nginx:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:latest
```

### 3.3 Image Naming Convention

```
registry/repository:tag
│         │        │
│         │        └── Version/variant (latest, 1.25, alpine)
│         └────────── Image name
└──────────────────── Registry (default: Docker Hub)

Examples:
nginx                          → docker.io/library/nginx:latest
node:18                        → docker.io/library/node:18
123456789012.dkr.ecr.us-east-1.amazonaws.com/my-api:v1.2
│                              │                    │    │
└── ECR registry               └── repository       └── tag
```

### 3.4 ECS Connection 🧠

> **Image = Container Image in ECR.** When you specify `image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:latest` in an ECS task definition, ECS pulls that image and runs it. Now you're doing the pull and run yourself!

---

## 📦 Module 4: Dockerfile — Writing the Blueprint

### 4.1 What Is a Dockerfile?

A Dockerfile is a text file with instructions to build a Docker image. **It's the core of what becomes an ECS Task Definition's container definition.**

### 4.2 Dockerfile Instructions — Complete Reference

```dockerfile
# ============================================
# BASIC DOCKERFILE — Node.js App Example
# ============================================

# FROM — Base image (REQUIRED, must be first)
# Like choosing the AMI for an EC2 instance
FROM node:18-alpine

# WORKDIR — Set working directory inside container
# Like cd-ing into a directory before running commands
WORKDIR /app

# COPY — Copy files from host → container
# First path = host (build context), Second path = container
COPY package*.json ./
COPY . .

# RUN — Execute commands during BUILD time (creates image layers)
# These commands run ONCE during build, result is baked into image
RUN npm ci --only=production

# ENV — Set environment variables
# Like environment variables in ECS task definition
ENV NODE_ENV=production
ENV PORT=3000

# EXPOSE — Document which port the container listens on
# This is DOCUMENTATION only — doesn't actually publish the port
# Like containerPort in ECS task definition
EXPOSE 3000

# HEALTHCHECK — Define how to check if container is healthy
# Like healthCheck in ECS task definition
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1

# CMD — Default command to run when container starts
# Like command/entryPoint in ECS task definition
# Only one CMD per Dockerfile (last one wins)
CMD ["node", "server.js"]
```

### 4.3 COPY vs ADD

```dockerfile
# COPY — Simple file copy (PREFER THIS)
COPY app.js /app/app.js
COPY ./src /app/src

# ADD — Copy + extra features (auto-extract tar, fetch URLs)
ADD archive.tar.gz /app/          # Auto-extracts!
ADD https://example.com/file.txt /app/  # Download from URL

# Rule of thumb: Use COPY unless you need ADD's extra features
```

### 4.4 CMD vs ENTRYPOINT

```dockerfile
# CMD — Default command (can be overridden at runtime)
# docker run myapp              → runs "node server.js"
# docker run myapp npm test     → runs "npm test" (overrides CMD)
CMD ["node", "server.js"]

# ENTRYPOINT — Command that ALWAYS runs (not easily overridden)
# docker run myapp               → runs "node server.js"
# docker run myapp --port 4000   → runs "node server.js --port 4000"
ENTRYPOINT ["node", "server.js"]

# Best practice: Combine ENTRYPOINT + CMD
# ENTRYPOINT = the executable, CMD = default arguments
ENTRYPOINT ["node"]
CMD ["server.js"]
# docker run myapp           → node server.js
# docker run myapp worker.js  → node worker.js
```

### 4.5 RUN — Building Layers

```dockerfile
# BAD — Each RUN creates a new layer, leaving cache behind
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y git
# Result: 3 layers, apt cache included in image!

# GOOD — Combine commands, clean up in same layer
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
      curl \
      git && \
    rm -rf /var/lib/apt/lists/*
# Result: 1 layer, no cache bloat!
```

### 4.6 Multi-Stage Builds (CRITICAL for production)

```dockerfile
# ============================================
# MULTI-STAGE BUILD — Why & How
# ============================================

# Stage 1: Build stage (compile, bundle, etc.)
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build          # Produces /app/dist/

# Stage 2: Production stage (only what's needed to run)
FROM node:18-alpine AS production
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY --from=builder /app/dist ./dist   # Copy ONLY build output!

ENV NODE_ENV=production
EXPOSE 3000
CMD ["node", "dist/server.js"]

# Result: Image is 10-50x smaller!
# Build stage: ~1GB → Production stage: ~150MB
```

### 4.7 .dockerignore

```text
# .dockerignore — Exclude files from build context (like .gitignore)
node_modules
npm-debug.log
.git
.gitignore
.env
Dockerfile
docker-compose.yml
.DS_Store
README.md
coverage
.nyc_output
```

### 4.8 Complete Real-World Dockerfile Examples

#### Python Flask App
```dockerfile
FROM python:3.11-slim AS builder

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

FROM python:3.11-slim

WORKDIR /app

COPY --from=builder /root/.local /root/.local
COPY . .

ENV PATH=/root/.local/bin:$PATH
ENV FLASK_APP=app.py

EXPOSE 5000

HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost:5000/health || exit 1

CMD ["flask", "run", "--host=0.0.0.0"]
```

#### Java Spring Boot App
```dockerfile
# Build stage
FROM maven:3.9-eclipse-temurin-17 AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn package -DskipTests

# Run stage
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar

ENV JAVA_OPTS="-Xmx512m"
EXPOSE 8080

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

### 4.9 ECS Connection 🧠

> **Dockerfile = The container definition block inside an ECS Task Definition.** When you specify `image`, `command`, `environment`, `portMappings`, `healthCheck` in a task definition — you're defining what the Dockerfile already specifies. The key difference: Dockerfile bakes config INTO the image; Task Definition provides config AT RUNTIME. In production, you often use both — Dockerfile for the app, Task Definition for environment-specific overrides.

---

## 📦 Module 5: Building Images

### 5.1 The Build Command

```bash
# Basic build
docker build -t my-app:1.0 .

# The "." is the BUILD CONTEXT — directory containing Dockerfile
# Docker sends the entire build context to the daemon

# Build with custom Dockerfile name
docker build -f Dockerfile.prod -t my-app:prod .

# Build with build arguments (like parameterized task definitions)
docker build --build-arg NODE_ENV=staging -t my-app:staging .

# Build no cache (force fresh build)
docker build --no-cache -t my-app:1.0 .

# Multi-stage: target specific stage
docker build --target builder -t my-app:builder .
```

### 5.2 Build Arguments (ARG)

```dockerfile
# ARG — Build-time variables (NOT available at runtime)
ARG NODE_VERSION=18
FROM node:${NODE_VERSION}

ARG BUILD_ENV=production
RUN echo "Building for ${BUILD_ENV}"

# ARG vs ENV:
# ARG   → Available during BUILD only
# ENV   → Available during BUILD + RUNTIME
# In ECS: Environment variables in task def = ENV, not ARG
```

### 5.3 Image Layering & Caching

```
Step 1: FROM node:18              → Cache hit (if previously pulled)
Step 2: WORKDIR /app               → Cache hit
Step 3: COPY package*.json ./      → Cache hit (if package.json unchanged)
Step 4: RUN npm ci                  → Cache hit (if Step 3 was cache hit)
Step 5: COPY . .                    → Cache INVALIDATED (code changes often)
Step 6: RUN npm run build           → Cache INVALIDATED

💡 KEY OPTIMIZATION: Order instructions from least to most frequently changing!
    1. Base image (rarely changes)
    2. System packages (rarely changes)
    3. Dependency install (changes when package.json changes)
    4. Application code (changes frequently)
```

### 5.4 ECS Connection 🧠

> **Building images is what your CI/CD pipeline does before pushing to ECR.** In ECS, you typically have: GitHub → GitHub Actions → `docker build` → `docker push` to ECR → ECS deploys new task definition. Now you're doing the build step manually!

---

## 📦 Module 6: Running Containers

### 6.1 The `docker run` Command

```bash
# Basic run
docker run nginx

# Run with name (like ECS task ID)
docker run --name my-nginx nginx

# Run in detached mode (background, like ECS tasks)
docker run -d --name my-nginx nginx

# Run with port mapping
# -p HOST_PORT:CONTAINER_PORT
docker run -d -p 8080:80 --name my-nginx nginx
# Now access at http://localhost:8080

# Run with environment variables
docker run -d -e NODE_ENV=production -e PORT=3000 my-app

# Run with resource limits (like task CPU/memory)
docker run -d --memory=512m --cpus=1.0 my-app

# Run with auto-remove on exit
docker run --rm my-app

# Run interactively (for debugging)
docker run -it node:18 /bin/sh

# Override CMD at runtime
docker run my-app npm test

# Run with env file
docker run --env-file .env my-app

# Full production-like run
docker run -d \
  --name my-api \
  -p 3000:3000 \
  -e NODE_ENV=production \
  -e DB_HOST=db.example.com \
  --memory=512m \
  --cpus=0.5 \
  --health-cmd="curl -f http://localhost:3000/health || exit 1" \
  --health-interval=30s \
  --health-timeout=3s \
  --restart=unless-stopped \
  my-app:1.0
```

### 6.2 Port Mapping Deep Dive

```
Port Mapping: -p HOST_PORT:CONTAINER_PORT

┌──────────────────────────────────────┐
│           Host Machine               │
│                                      │
│  Port 8080 ◄──── External Traffic    │
│      │                               │
│      ▼ (mapped)                      │
│  ┌────────────────────────────┐      │
│  │     Container              │      │
│  │     Port 80 (nginx)        │      │
│  └────────────────────────────┘      │
└──────────────────────────────────────┘

# Multiple ports
docker run -p 8080:80 -p 8443:443 nginx

# Bind to specific interface
docker run -p 127.0.0.1:3000:3000 my-app  # Only localhost

# Random host port
docker run -p 80 nginx  # Docker assigns random host port
```

### 6.3 Container Lifecycle

```
                    docker create
                        │
                        ▼
                  ┌───────────┐
                  │  Created   │
                  └─────┬─────┘
                        │ docker start
                        ▼
                  ┌───────────┐  docker pause   ┌───────────┐
                  │  Running   │───────────────▶│  Paused    │
                  └─────┬─────┘  docker unpause └───────────┘
                        │
            ┌───────────┼───────────┐
            │           │           │
    docker stop   docker kill   Process exits
            │           │           │
            ▼           ▼           ▼
      ┌───────────┐ ┌───────────┐ ┌───────────┐
      │  Stopped   │ │  Dead      │ │  Exited    │
      └─────┬─────┘ └───────────┘ └───────────┘
            │ docker start              │
            └──────────────────────────┘
                  (restart if policy set)
```

```bash
# Lifecycle commands
docker create my-app                    # Create without starting
docker start my-app                      # Start a stopped container
docker stop my-app                       # Graceful stop (SIGTERM → SIGKILL after 10s)
docker kill my-app                       # Force kill (SIGKILL immediately)
docker restart my-app                    # Stop + Start
docker pause my-app                      # Freeze all processes (SIGSTOP)
docker unpause my-app                    # Unfreeze
docker rm my-app                         # Remove stopped container
docker rm -f my-app                      # Force remove (even if running)

# Restart policies (like ECS service auto-restart)
docker run --restart=no          # Never restart (default)
docker run --restart=on-failure  # Restart only on non-zero exit
docker run --restart=unless-stopped  # Always restart (except manual stop)
docker run --restart=always     # Always restart (even after manual stop)
```

### 6.4 ECS Connection 🧠

> **`docker run` = ECS `RunTask` API.** When you run a task in ECS, it's doing `docker run` under the hood. Port mapping (`-p`) = `portMappings` in task definition. Resource limits (`--memory`, `--cpus`) = task `memory`/`cpu`. Restart policy = ECS Service's desired count maintenance. The key difference: ECS manages the lifecycle for you; with raw Docker, you manage it yourself.

---

## 📦 Module 7: Managing Containers

### 7.1 Listing & Inspecting

```bash
# List running containers
docker ps

# List ALL containers (including stopped)
docker ps -a

# List with custom format
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Inspect container details (like describing an ECS task)
docker inspect my-nginx

# Get specific info with format
docker inspect --format '{{.NetworkSettings.IPAddress }}' my-nginx
docker inspect --format '{{.Config.Env}}' my-nginx

# Show container resource usage (like CloudWatch metrics)
docker stats
docker stats my-nginx

# Show container processes
docker top my-nginx
```

### 7.2 Logs

```bash
# View container logs (like CloudWatch Logs)
docker logs my-nginx

# Follow logs (tail -f)
docker logs -f my-nginx

# Last N lines
docker logs --tail 100 my-nginx

# With timestamps
docker logs -t my-nginx

# Since a time
docker logs --since 30m my-nginx
docker logs --since 2024-01-01T00:00:00 my-nginx
```

### 7.3 Executing Commands Inside Containers

```bash
# Run a command inside a running container (like ECS Exec)
docker exec my-nginx ls /etc/nginx

# Open interactive shell (most common!)
docker exec -it my-nginx /bin/sh
docker exec -it my-nginx /bin/bash

# Run as specific user
docker exec -u root my-nginx cat /etc/passwd

# Set environment variables
docker exec -e DEBUG=1 my-nginx node --version
```

### 7.4 Copying Files

```bash
# Copy from container to host
docker cp my-nginx:/etc/nginx/nginx.conf ./nginx.conf

# Copy from host to container
docker cp ./custom.conf my-nginx:/etc/nginx/conf.d/custom.conf
```

### 7.5 ECS Connection 🧠

> **`docker logs` = CloudWatch Logs.** In ECS with `awslogs` driver, container stdout/stderr goes to CloudWatch. `docker exec` = ECS Exec (`enableExecuteCommand`). `docker stats` = CloudWatch Container Insights. You're doing the same things, just locally instead of through AWS APIs.

---

## 📦 Module 8: Docker Networking

### 8.1 Network Drivers

```
┌─────────────────────────────────────────────────┐
│              Docker Network Types                 │
├─────────────┬───────────────────────────────────┤
│ bridge      │ Default. Containers on same host    │
│             │ can communicate. Like ECS bridge   │
│             │ network mode.                      │
├─────────────┼───────────────────────────────────┤
│ host        │ Container shares host's network.   │
│             │ No port mapping needed. Like ECS   │
│             │ host network mode.                 │
├─────────────┼───────────────────────────────────┤
│ overlay     │ Multi-host networking (Swarm).     │
│             │ Like ECS awsvpc mode across AZs.   │
├─────────────┼───────────────────────────────────┤
│ macvlan     │ Container gets its own MAC address.│
│             │ Appears as physical device on net.  │
├─────────────┼───────────────────────────────────┤
│ none        │ No networking. Isolated container.  │
└─────────────┴───────────────────────────────────┘
```

### 8.2 Working with Networks

```bash
# List networks
docker network ls

# Create a custom bridge network
docker network create my-network

# Run containers on the network
docker run -d --name api --network my-network my-api
docker run -d --name db --network my-network postgres:15

# Containers can resolve each other by NAME!
# Inside api container: ping db → works!
# This is like ECS Service Connect / Service Discovery

# Inspect network
docker network inspect my-network

# Connect running container to another network
docker network connect another-network api

# Disconnect
docker network disconnect another-network api

# Remove network
docker network rm my-network
```

### 8.3 DNS & Service Discovery

```
In a custom bridge network:
┌─────────────────────────────────────┐
│         my-network (bridge)          │
│                                      │
│  ┌──────┐    ┌──────┐    ┌──────┐   │
│  │ api  │───▶│  db  │    │redis │   │
│  │      │    │:5432 │    │:6379 │   │
│  └──────┘    └──────┘    └──────┘   │
│                                      │
│  api can reach: db:5432, redis:6379  │
│  (DNS resolution by container name)   │
└─────────────────────────────────────┘

This is EXACTLY like ECS Service Connect / CloudMap!
```

### 8.4 Port Mapping vs Network Modes

```bash
# Bridge mode (default) — must publish ports
docker run -d -p 8080:80 --network bridge nginx

# Host mode — no port mapping needed, uses host network
docker run -d --network host nginx
# nginx accessible on host's port 80 directly

# None — no network
docker run -d --network none my-app
```

### 8.5 ECS Connection 🧠

> **Docker bridge network = ECS bridge network mode.** Docker custom bridge with DNS = ECS Service Connect / CloudMap Service Discovery. Docker host network = ECS host network mode. Docker overlay = ECS awsvpc mode (roughly — both provide isolated IP per container). The key insight: ECS networking is Docker networking, managed by AWS.

---

## 📦 Module 9: Docker Volumes & Persistent Storage

### 9.1 The Problem: Ephemeral Containers

```
Container data is EPHEMERAL:
┌──────────────┐
│  Container    │  docker rm → ALL DATA GONE!
│  /app/data/   │
│  (writable    │
│   layer)      │
└──────────────┘

Solution: Volumes & Bind Mounts
```

### 9.2 Three Types of Mounts

```
┌─────────────────────────────────────────────────────┐
│                  Mount Types                         │
├──────────────┬──────────────┬───────────────────────┤
│ Bind Mount   │ Volume       │ tmpfs                  │
├──────────────┼──────────────┼───────────────────────┤
│ Host dir →   │ Docker-      │ In-memory only         │
│ Container    │ managed dir  │ (ephemeral)            │
│              │ → Container  │                        │
├──────────────┼──────────────┼───────────────────────┤
│ -v /host:/ct │ -v name:/ct │ --tmpfs /ct            │
│              │ --mount      │                        │
├──────────────┼──────────────┼───────────────────────┤
│ Dev workflow │ Production   │ Sensitive data         │
│ (live reload)│ persistence  │ (not on disk)          │
└──────────────┴──────────────┴───────────────────────┘
```

### 9.3 Working with Volumes

```bash
# Create a named volume
docker volume create pg-data

# List volumes
docker volume ls

# Inspect volume
docker volume inspect pg-data

# Use volume with a container
docker run -d \
  --name postgres-db \
  -v pg-data:/var/lib/postgresql/data \
  postgres:15

# Even if container is removed, data persists!
docker rm -f postgres-db
docker run -d \
  --name postgres-db \
  -v pg-data:/var/lib/postgresql/data \
  postgres:15
# Data is still there!

# Remove volume
docker volume rm pg-data

# Remove all unused volumes
docker volume prune
```

### 9.4 Bind Mounts (Development)

```bash
# Bind mount: host directory → container directory
docker run -d \
  -v $(pwd)/src:/app/src \
  my-app
# Changes on host are immediately reflected in container!
# Great for hot-reloading during development

# Read-only bind mount
docker run -d \
  -v $(pwd)/config:/app/config:ro \
  my-app
```

### 9.5 The --mount Syntax (More Explicit)

```bash
# Volume mount (--mount is more verbose but clearer)
docker run -d \
  --mount type=volume,source=pg-data,target=/var/lib/postgresql/data \
  postgres:15

# Bind mount
docker run -d \
  --mount type=bind,source=$(pwd)/src,target=/app/src \
  my-app

# Read-only bind mount
docker run -d \
  --mount type=bind,source=$(pwd)/config,target=/app/config,readonly \
  my-app
```

### 9.6 ECS Connection 🧠

> **Docker Volumes = EFS Volumes in ECS.** When you mount EFS in an ECS task definition, it's the same concept — persistent storage that outlives the container. Bind mounts are more like local EC2 instance storage. In Fargate, you can only use EFS (like Docker volumes). In EC2 launch type, you can use bind mounts to the host.

---

## 📦 Module 10: Docker Compose — Multi-Container Apps

### 10.1 What Is Docker Compose?

Docker Compose defines and runs **multi-container applications** with a single YAML file. **It's like an entire ECS Service + Task Definition + Load Balancer + Service Discovery in one file.**

### 10.2 docker-compose.yml — Full Example

```yaml
version: "3.9"

services:
  # ─── API Service ─────────────────────────
  api:
    build:
      context: ./api
      dockerfile: Dockerfile
      target: production       # Multi-stage build target
    image: my-api:1.0          # Image name
    container_name: my-api
    ports:
      - "3000:3000"            # Like portMappings in task def
    environment:
      - NODE_ENV=production    # Like environment in task def
      - DB_HOST=postgres       # Uses service name as hostname!
      - DB_PORT=5432
      - REDIS_HOST=redis
      - REDIS_PORT=6379
    env_file:
      - .env.production       # Load from file
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 3s
      retries: 3
      start_period: 10s
    volumes:
      - api-logs:/app/logs     # Named volume
    networks:
      - app-network
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
        reservations:
          cpus: "0.5"
          memory: 256M

  # ─── Worker Service ──────────────────────
  worker:
    build:
      context: ./worker
      dockerfile: Dockerfile
    environment:
      - REDIS_HOST=redis
      - DB_HOST=postgres
    depends_on:
      - redis
      - postgres
    networks:
      - app-network
    restart: unless-stopped

  # ─── PostgreSQL Database ────────────────
  postgres:
    image: postgres:15-alpine
    container_name: my-postgres
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: ${DB_PASSWORD}  # From .env file
    ports:
      - "5432:5432"
    volumes:
      - pg-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin -d myapp"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network

  # ─── Redis Cache ────────────────────────
  redis:
    image: redis:7-alpine
    container_name: my-redis
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3
    networks:
      - app-network

  # ─── Nginx Reverse Proxy ────────────────
  nginx:
    image: nginx:alpine
    container_name: my-nginx
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
    depends_on:
      - api
    networks:
      - app-network

# ─── Named Volumes ──────────────────────────
volumes:
  pg-data:
  redis-data:
  api-logs:

# ─── Networks ───────────────────────────────
networks:
  app-network:
    driver: bridge
```

### 10.3 Docker Compose Commands

```bash
# Start all services (detached)
docker compose up -d

# Start and rebuild images
docker compose up -d --build

# Start specific services only
docker compose up -d api postgres

# View logs
docker compose logs
docker compose logs -f api          # Follow specific service
docker compose logs --tail 50 api  # Last 50 lines

# List running services
docker compose ps

# Execute command in running service
docker compose exec api sh
docker compose exec postgres psql -U admin -d myapp

# Scale a service (like increasing desired count in ECS)
docker compose up -d --scale api=3

# Stop all services
docker compose stop

# Stop and remove containers, networks
docker compose down

# Stop and remove everything (including volumes!)
docker compose down -v

# Restart a single service
docker compose restart api

# Pull latest images
docker compose pull

# Validate compose file
docker compose config
```

### 10.4 Compose for Development vs Production

```yaml
# docker-compose.override.yml (auto-loaded for dev)
# This file is automatically merged with docker-compose.yml
services:
  api:
    build:
      target: development    # Use dev stage
    volumes:
      - ./api/src:/app/src   # Hot reload!
    environment:
      - NODE_ENV=development
      - DEBUG=1
    command: npm run dev      # Override with dev server

# docker-compose.prod.yml (explicit: -f flag)
# docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
services:
  api:
    build:
      target: production
    environment:
      - NODE_ENV=production
```

### 10.5 ECS Connection 🧠

> **Docker Compose = ECS Service + Task Definition + ALB + Service Discovery, all in one file.** Each `service` in Compose ≈ one ECS Service. `depends_on` ≈ service dependency. `healthcheck` = task definition healthCheck. `volumes` = EFS mounts. `networks` = VPC/service mesh. `deploy.resources` = task CPU/memory. The big difference: Compose runs on one machine; ECS distributes across a cluster. **Docker Compose is for local dev; ECS is for production.**

---

## 📦 Module 11: Docker Registry — Pushing & Pulling Images

### 11.1 Docker Hub (Public Registry)

```bash
# Login to Docker Hub
docker login

# Tag image for Docker Hub
docker tag my-api:1.0 myusername/my-api:1.0

# Push to Docker Hub
docker push myusername/my-api:1.0

# Pull from Docker Hub
docker pull myusername/my-api:1.0
```

### 11.2 Amazon ECR (What You Already Know!)

```bash
# Login to ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com

# Create ECR repository
aws ecr create-repository --repository-name my-api

# Tag for ECR
docker tag my-api:1.0 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-api:1.0

# Push to ECR
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-api:1.0

# Pull from ECR
docker pull 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-api:1.0
```

### 11.3 ECS Connection 🧠

> **This is the exact workflow in your CI/CD pipeline!** GitHub Actions → `docker build` → `docker push` to ECR → Update ECS task definition → Deploy. You're just doing manually what the pipeline automates.

---

## 📦 Module 12: Docker Best Practices for Production

### 12.1 Image Best Practices

```
✅ DO                                    ❌ DON'T
───────────────────────────              ───────────────────────────
Use specific tags (node:18.17)          Use :latest tag in production
Use Alpine/slim variants                 Use full Debian images
Multi-stage builds                      Include build tools in final image
Run as non-root user                    Run as root
Use .dockerignore                        Send entire context
Order layers least→most changing         Put COPY . . before npm ci
Combine RUN commands                     Create many layers
Clean package manager cache              Leave cache in image
Use COPY over ADD                        Use ADD for simple copies
```

### 12.2 Security Best Practices

```dockerfile
# Run as non-root user
FROM node:18-alpine
WORKDIR /app
COPY --chown=node:node . .
USER node                    # ← Critical! Don't run as root
CMD ["node", "server.js"]

# Scan images for vulnerabilities
docker scout cves my-api:1.0

# Use read-only filesystem where possible
docker run --read-only --tmpfs /tmp my-api
```

### 12.3 Performance Best Practices

```dockerfile
# Optimize layer caching
# GOOD: Copy dependency files first, install, then copy code
COPY package*.json ./
RUN npm ci
COPY . .

# BAD: Copy everything first (code changes invalidate cache)
COPY . .
RUN npm ci

# Use BuildKit for faster builds
DOCKER_BUILDKIT=1 docker build -t my-app .

# Use .dockerignore to reduce context size
echo "node_modules\n.git\n*.md" > .dockerignore
```

### 12.4 Logging Best Practices

```bash
# Docker uses json-file driver by default (can fill up disk!)
# Configure log rotation in /etc/docker/daemon.json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}

# In ECS: awslogs driver → CloudWatch (you already have this!)
```

### 12.5 ECS Connection 🧠

> **These best practices directly apply to your ECS workloads.** Non-root user = `user` in task definition. Resource limits = task CPU/memory. Log driver = `logConfiguration` in task definition. Image scanning = ECR image scanning. The container is the container — best practices are universal.

---

## 📦 Module 13: Docker System Management & Cleanup

### 13.1 System Commands

```bash
# System-wide information
docker system info
docker info

# Disk usage
docker system df
docker system df -v    # Verbose

# Clean up everything
docker system prune     # Remove unused data
docker system prune -a  # Remove ALL unused (including images!)
docker system prune --volumes  # Include volumes

# Specific cleanup
docker image prune       # Remove dangling images
docker image prune -a    # Remove all unused images
docker container prune   # Remove stopped containers
docker volume prune      # Remove unused volumes
docker network prune     # Remove unused networks
```

### 13.2 Common Maintenance

```bash
# Remove dangling images (none:none)
docker rmi $(docker images -f "dangling=true" -q)

# Remove all stopped containers
docker rm $(docker ps -aq -f status=exited)

# Force remove all containers (DANGER!)
docker rm -f $(docker ps -aq)

# Remove all images (DANGER!)
docker rmi $(docker images -q)
```

---

## 📦 Module 14: Docker Networking Deep Dive

### 14.1 Container Communication Patterns

```
Pattern 1: Same Network (Service Discovery by name)
┌──────────────────────────────┐
│        app-network            │
│  ┌──────┐      ┌──────┐     │
│  │ api  │─────▶│  db  │     │
│  └──────┘      └──────┘     │
│  api → db:5432 (DNS works!) │
└──────────────────────────────┘

Pattern 2: Port Mapping (External Access)
┌──────────────────────────────┐
│  Host :8080 ──▶ api :80     │
│  Host :5432 ──▶ db :5432    │
└──────────────────────────────┘

Pattern 3: Multi-network (Isolation)
┌──────────────┐  ┌──────────────┐
│  frontend-net │  │  backend-net │
│  ┌──────┐    │  │  ┌──────┐    │
│  │ web  │────┼──┼─▶│ api  │    │
│  └──────┘    │  │  └──────┘    │
│              │  │  ┌──────┐    │
│              │  │  │  db  │    │
│              │  │  └──────┘    │
└──────────────┘  └──────────────┘
  web can reach api, but NOT db
```

### 14.2 Advanced Networking

```bash
# Create network with custom subnet
docker network create \
  --driver bridge \
  --subnet 172.20.0.0/16 \
  --gateway 172.20.0.1 \
  my-custom-network

# Connect container to multiple networks
docker network connect frontend-net api
docker network connect backend-net api
# api is now on both networks!

# DNS within Docker
# Default: container name resolves
# Custom: --network-alias
docker run --network app-network --network-alias api my-api
docker run --network app-network --network-alias api-v2 my-api-v2
# Both resolve to "api" → basic load balancing!
```

---

## 📦 Module 15: Docker Environment & Configuration

### 15.1 Ways to Pass Configuration

```bash
# 1. Environment variables (-e)
docker run -e DB_HOST=localhost -e DB_PORT=5432 my-app

# 2. Env file (--env-file)
docker run --env-file .env my-app

# 3. From Dockerfile ENV
# Baked into image, can be overridden at runtime

# 4. From Docker Compose
# environment: or env_file: in compose file
```

### 15.2 .env File

```bash
# .env file
DB_HOST=postgres
DB_PORT=5432
DB_NAME=myapp
DB_USER=admin
DB_PASSWORD=supersecret
NODE_ENV=production
```

### 15.3 Secrets Management

```bash
# Docker Swarm secrets (not available in standalone Docker)
echo "mysecret" | docker secret create db_password -

# For standalone Docker, use:
# 1. Environment variables (not ideal for secrets)
# 2. Mounted files (better)
docker run -v ./secrets/db_password:/run/secrets/db_password:ro my-app

# In ECS: Use AWS Secrets Manager or SSM Parameter Store
# (You already know this!)
```

### 15.4 ECS Connection 🧠

> **Docker env vars = ECS task definition environment variables.** Docker env file = ECS task definition `environment` array. Docker secrets = AWS Secrets Manager / SSM Parameter Store (via `secrets` in task definition). The pattern is identical; AWS just provides managed secret stores.

---

## 📦 Module 16: Docker Debugging & Troubleshooting

### 16.1 Common Issues & Solutions

```bash
# Issue: Container exits immediately
docker logs <container>           # Check what went wrong
docker inspect <container>        # Check exit code, state
docker run -it <image> /bin/sh   # Run interactively to debug

# Issue: Port already in use
lsof -i :3000                     # Find what's using the port
docker stop $(docker ps -q)       # Stop all containers

# Issue: Image not found / pull errors
docker pull <image>               # Try pulling manually
docker images                     # Check if image exists locally

# Issue: Container keeps restarting
docker logs --tail 50 <container> # Check last 50 lines
docker inspect --format '{{.State.Status}}' <container>
docker inspect --format '{{.State.ExitCode}}' <container>
# Exit code 1 = app error, 137 = OOM killed, 0 = graceful exit

# Issue: Permission denied
docker exec -u root <container> chmod +x /app/entrypoint.sh
# Or fix in Dockerfile: USER root before the command

# Issue: Out of disk space
docker system df                  # Check disk usage
docker system prune -a            # Clean up everything
```

### 16.2 Exit Codes Reference

```
Exit Code  Meaning                    Docker Context
─────────  ─────────────────────────  ─────────────────────────
0          Graceful exit               App completed successfully
1          Application error           General error in app
137        OOM Killed (128+9)          Out of memory — increase --memory
139        Segfault (128+11)           App crashed — check code
143        SIGTERM (128+15)            docker stop — graceful shutdown
125        Docker daemon error         Docker itself had an issue
126        Cannot invoke command       Permission issue on entrypoint
127        Command not found           Wrong CMD/ENTRYPOINT path
```

### 16.3 Debugging Workflow

```
1. Container won't start?
   └─▶ docker logs <container>
   └─▶ docker inspect <container> (check State.ExitCode)
   └─▶ docker run -it <image> /bin/sh (interactive debug)

2. Container runs but app misbehaves?
   └─▶ docker exec -it <container> /bin/sh
   └─▶ Check env vars: env
   └─▶ Check filesystem: ls, cat config files
   └─▶ Check connectivity: curl, ping, nc

3. Performance issues?
   └─▶ docker stats (CPU, memory, network, disk I/O)
   └─▶ docker top <container> (running processes)
   └─▶ Check if hitting resource limits

4. Networking issues?
   └─▶ docker network inspect <network>
   └─▶ docker exec -it <container> ping <other-container>
   └─▶ docker exec -it <container> curl http://api:3000/health
   └─▶ docker port <container> (check port mappings)
```

### 16.4 ECS Connection 🧠

> **Debugging Docker locally = debugging ECS tasks.** `docker logs` = CloudWatch Logs. `docker exec` = ECS Exec. `docker inspect` = `DescribeTasks` API. Exit code 137 = same OOM kill you see in ECS stopped tasks. The debugging mindset is identical — only the tools differ.

---

## 📦 Module 17: Docker Compose → ECS Migration Playbook

### 17.1 From docker-compose.yml to ECS Task Definition

This is the **most practical module** — mapping your local Docker Compose setup to production ECS.

```
docker-compose.yml          →    ECS Infrastructure
───────────────────────        ─────────────────────────
services:                       AWS::ECS::Service
  api:                            Task Definition
    image: my-api:1.0        →      Container Definition: image
    ports:                    →      portMappings
    environment:              →      environment (or secrets from SM/SSM)
    healthcheck:              →      healthCheck
    deploy.resources:         →      cpu, memory (task level)
    volumes:                  →      mountPoints + EFS volume
    networks:                 →      awsvpc network mode
    depends_on:               →      Service Connect / ALB routing
    restart:                  →      Service desired count maintenance

volumes:                      →    EFS File System + Access Points
networks:                     →    VPC + Subnets + Security Groups
```

### 17.2 Step-by-Step Migration

```bash
# Step 1: Build & push images to ECR
docker build -t my-api:1.0 ./api
docker tag my-api:1.0 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-api:1.0
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-api:1.0

# Step 2: Create ECS Cluster
aws ecs create-cluster --cluster-name my-app-cluster

# Step 3: Create Task Definition (JSON)
# This is your docker-compose.yml translated to AWS format
```

```json
{
  "family": "my-api-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::123456789012:role/ecsTaskRole",
  "containerDefinitions": [
    {
      "name": "api",
      "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/my-api:1.0",
      "essential": true,
      "portMappings": [
        { "containerPort": 3000, "protocol": "tcp" }
      ],
      "environment": [
        { "name": "NODE_ENV", "value": "production" },
        { "name": "DB_HOST", "value": "my-rds.cluster-xxxxx.us-east-1.rds.amazonaws.com" }
      ],
      "secrets": [
        { "name": "DB_PASSWORD", "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:db-password" }
      ],
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:3000/health || exit 1"],
        "interval": 30,
        "timeout": 3,
        "retries": 3,
        "startPeriod": 10
      },
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/my-api",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

```bash
# Step 4: Register Task Definition
aws ecs register-task-definition --cli-input-json file://task-definition.json

# Step 5: Create ECS Service
aws ecs create-service \
  --cluster my-app-cluster \
  --service-name my-api \
  --task-definition my-api-task:1 \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-xxx],securityGroups=[sg-xxx],assignPublicIp=DISABLED}" \
  --load-balancers targetGroupArn=arn:aws:elasticloadbalancing:...,containerName=api,containerPort=3000
```

### 17.3 Docker Compose to ECS with Copilot (Easy Way!)

```bash
# AWS Copilot CLI — translates Compose to ECS automatically!
brew install aws/tap/copilot

# Initialize from your docker-compose.yml
copilot init

# Deploy!
copilot deploy

# Copilot auto-creates:
# - ECS Cluster
# - Task Definitions
# - Load Balancer
# - CloudWatch Log Groups
# - IAM Roles
# - VPC networking
```

### 17.4 ECS Connection 🧠

> **This is the bridge between your Docker knowledge and your ECS experience.** Every `docker-compose.yml` you write locally can be translated to ECS infrastructure. The concepts are 1:1 — only the syntax and managed services differ. Docker Compose is your local prototype; ECS is your production deployment.

---

## 📦 Module 18: Docker in CI/CD Pipelines

### 18.1 The Complete Build Pipeline

```yaml
# .github/workflows/deploy.yml
name: Build and Deploy to ECS

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Login to ECR
        run: |
          aws ecr get-login-password --region us-east-1 | \
            docker login --username AWS --password-stdin \
            123456789012.dkr.ecr.us-east-1.amazonaws.com

      - name: Build Docker image
        run: |
          docker build -t my-api:${{ github.sha }} .
          docker tag my-api:${{ github.sha }} \
            123456789012.dkr.ecr.us-east-1.amazonaws.com/my-api:${{ github.sha }}

      - name: Push to ECR
        run: |
          docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-api:${{ github.sha }}

      - name: Update ECS Task Definition
        run: |
          aws ecs update-service \
            --cluster my-cluster \
            --service my-api \
            --force-new-deployment
```

### 18.2 Docker Build Cache in CI

```yaml
# Use GitHub Actions cache for Docker layers
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3

- name: Cache Docker layers
  uses: actions/cache@v3
  with:
    path: /tmp/.buildx-cache
    key: ${{ runner.os }}-buildx-${{ github.sha }}
    restore-keys: |
      ${{ runner.os }}-buildx-

- name: Build with cache
  run: |
    docker buildx build \
      --cache-from type=local,src=/tmp/.buildx-cache \
      --cache-to type=local,dest=/tmp/.buildx-cache-new \
      -t my-api:${{ github.sha }} \
      --push \
      .
```

### 18.3 Multi-Architecture Builds

```bash
# Build for multiple platforms (ARM + AMD)
docker buildx create --use
docker buildx build --platform linux/amd64,linux/arm64 \
  -t my-api:1.0 \
  --push \
  .

# This matters for:
# - ECS on Graviton (ARM) instances
# - M1/M2 Mac development (ARM)
# - Cost savings (ARM is cheaper on AWS)
```

### 18.4 ECS Connection 🧠

> **This IS your production workflow.** Every Docker command in this pipeline is what you've learned in this course. `docker build` → `docker tag` → `docker push` to ECR → ECS deploys. Understanding Docker deeply makes you a better DevOps engineer because you can debug the pipeline when it breaks.

---

## 📦 Module 19: Docker vs Container Runtimes

### 19.1 The Container Ecosystem

```
┌─────────────────────────────────────────────────────┐
│              Container Ecosystem                      │
│                                                       │
│  OCI (Open Container Initiative)                      │
│  ├── Image Spec (how images are structured)           │
│  └── Runtime Spec (how containers run)               │
│                                                       │
│  Container Runtimes:                                  │
│  ├── runc (low-level, default in Docker)             │
│  ├── containerd (high-level, manages runc)            │
│  ├── CRI-O (Kubernetes-optimized)                    │
│  └── Firecracker (microVM, AWS Lambda/Fargate)       │
│                                                       │
│  Container Engines:                                   │
│  ├── Docker (uses containerd + runc)                  │
│  ├── Podman (daemonless, Docker-compatible)           │
│  └── Nerdctl (containerd CLI)                        │
│                                                       │
│  Orchestrators:                                       │
│  ├── Kubernetes (K8s)                                 │
│  ├── Amazon ECS (your expertise!)                     │
│  ├── Docker Swarm                                     │
│  └── Nomad                                            │
└─────────────────────────────────────────────────────┘
```

### 19.2 Docker Architecture Internals

```
┌──────────────────────────────────────────────┐
│  docker run nginx                             │
│       │                                       │
│       ▼                                       │
│  Docker Client (CLI)                          │
│       │ REST API                              │
│       ▼                                       │
│  Docker Daemon (dockerd)                      │
│       │                                       │
│       ├──▶ containerd                         │
│  │         │                                  │
│  │         └──▶ runc                           │
│  │              │                              │
│  │              └──▶ Container Process         │
│  │                                            │
│  └──▶ Image management (pull, build, etc.)   │
└──────────────────────────────────────────────┘
```

### 19.3 ECS Under the Hood

```
ECS Fargate:
┌──────────────────────────────────────┐
│  AWS-managed infrastructure           │
│  ┌────────────────────────────────┐  │
│  │  Firecracker microVM            │  │
│  │  ┌──────────────────────────┐  │  │
│  │  │  containerd               │  │  │
│  │  │  ┌────┐  ┌────┐         │  │  │
│  │  │  │ C1 │  │ C2 │         │  │  │
│  │  │  └────┘  └────┘         │  │  │
│  │  └──────────────────────────┘  │  │
│  └────────────────────────────────┘  │
│  You never see this — AWS manages it │
└──────────────────────────────────────┘

ECS on EC2:
┌──────────────────────────────────────┐
│  Your EC2 Instance                    │
│  ┌────────────────────────────────┐  │
│  │  Docker Daemon                 │  │
│  │  ┌──────────────────────────┐  │  │
│  │  │  ECS Agent                │  │  │
│  │  │  (manages Docker for ECS) │  │  │
│  │  └──────────────────────────┘  │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
```

### 19.4 ECS Connection 🧠

> **ECS doesn't use the Docker daemon on Fargate — it uses containerd directly.** On EC2 launch type, the ECS Agent talks to the Docker daemon. This is why ECS is "Docker-compatible" but not "Docker-dependent." Understanding this helps you debug: if a container works locally with Docker but fails on ECS, it might be a runtime difference.

---

## 📦 Module 20: Hands-On Labs — Practice Projects

### Lab 1: Containerize a Node.js App (30 min)

```bash
# 1. Create a simple Node.js app
mkdir my-docker-app && cd my-docker-app
cat > server.js << 'EOF'
const http = require('http');
const host = '0.0.0.0';
const port = process.env.PORT || 3000;
const server = http.createServer((req, res) => {
  res.writeHead(200);
  res.end(`Hello from Docker! Env: ${process.env.NODE_ENV || 'not set'}\n`);
});
server.listen(port, host, () => {
  console.log(`Running on http://${host}:${port}`);
});
EOF

cat > package.json << 'EOF'
{
  "name": "my-docker-app",
  "version": "1.0.0",
  "main": "server.js"
}
EOF

# 2. Write Dockerfile
cat > Dockerfile << 'EOF'
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
COPY . .
ENV NODE_ENV=production
ENV PORT=3000
EXPOSE 3000
USER node
CMD ["node", "server.js"]
EOF

# 3. Build and run
docker build -t my-docker-app:1.0 .
docker run -d -p 3000:3000 --name my-app my-docker-app:1.0
curl http://localhost:3000

# 4. Inspect
docker logs my-app
docker inspect my-app
docker exec -it my-app sh

# 5. Clean up
docker stop my-app && docker rm my-app
```

### Lab 2: Multi-Container App with Compose (45 min)

```bash
# 1. Create project structure
mkdir docker-compose-lab && cd docker-compose-lab
mkdir api db

# 2. Create a simple API
cat > api/server.js << 'EOF'
const http = require('http');
const host = '0.0.0.0';
const port = 3000;
const server = http.createServer((req, res) => {
  res.writeHead(200, {'Content-Type': 'application/json'});
  res.end(JSON.stringify({
    message: 'Hello from API!',
    db_host: process.env.DB_HOST,
    db_port: process.env.DB_PORT
  }));
});
server.listen(port, host, () => console.log(`API on ${host}:${port}`));
EOF

cat > api/Dockerfile << 'EOF'
FROM node:18-alpine
WORKDIR /app
COPY . .
ENV PORT=3000
EXPOSE 3000
CMD ["node", "server.js"]
EOF

# 3. Create docker-compose.yml
cat > docker-compose.yml << 'EOF'
version: "3.9"
services:
  api:
    build: ./api
    ports:
      - "3000:3000"
    environment:
      - DB_HOST=postgres
      - DB_PORT=5432
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - app-net

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: password
    volumes:
      - pg-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin"]
      interval: 5s
      timeout: 3s
      retries: 5
    networks:
      - app-net

volumes:
  pg-data:

networks:
  app-net:
    driver: bridge
EOF

# 4. Run it
docker compose up -d --build
docker compose ps
curl http://localhost:3000

# 5. Explore
docker compose exec api sh -c "ping -c 2 postgres"
docker compose logs api
docker compose exec postgres psql -U admin -d myapp -c "\\dt"

# 6. Scale the API
docker compose up -d --scale api=3

# 7. Clean up
docker compose down -v
```

### Lab 3: Build → Push to ECR → Run (60 min)

```bash
# 1. Build a production-grade image
docker build -t my-api:1.0 .

# 2. Login to ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com

# 3. Create ECR repo
aws ecr create-repository --repository-name my-api

# 4. Tag and push
docker tag my-api:1.0 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-api:1.0
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-api:1.0

# 5. Verify in ECR
aws ecr describe-images --repository-name my-api

# 6. Now deploy to ECS (you know this part!)
aws ecs update-service \
  --cluster my-cluster \
  --service my-api \
  --force-new-deployment
```

### Lab 4: Debugging Challenge (30 min)

```bash
# Challenge: Fix the broken container!

# 1. Run this "broken" container
docker run -d --name broken -p 8080:80 nginx:alpine

# 2. It's running but returning 403 Forbidden
# Debug it:
docker exec -it broken sh
ls /usr/share/nginx/html/     # Empty!
exit

# 3. Fix: Copy content into the container
echo "<h1>Fixed!</h1>" > index.html
docker cp index.html broken:/usr/share/nginx/html/index.html
curl http://localhost:8080    # Works now!

# 4. Better fix: Use a volume mount
docker stop broken && docker rm broken
mkdir -p ./html && echo "<h1>Volume mounted!</h1>" > ./html/index.html
docker run -d --name fixed -p 8080:80 -v $(pwd)/html:/usr/share/nginx/html:ro nginx:alpine
curl http://localhost:8080

# 5. Clean up
docker stop fixed && docker rm fixed
```

---

## 🏆 Docker Command Cheat Sheet (Quick Reference)

### Image Commands
```bash
docker build -t name:tag .              # Build image from Dockerfile
docker images                            # List local images
docker pull image:tag                    # Pull image from registry
docker push image:tag                    # Push image to registry
docker tag src dst                       # Tag an image
docker rmi image:tag                     # Remove image
docker image prune                       # Remove dangling images
docker inspect image:tag                 # Inspect image details
```

### Container Commands
```bash
docker run -d -p 8080:80 --name web nginx   # Run container
docker ps                                    # List running containers
docker ps -a                                 # List all containers
docker stop <name>                           # Stop container
docker start <name>                          # Start stopped container
docker rm <name>                             # Remove container
docker rm -f <name>                          # Force remove
docker logs -f <name>                        # Follow logs
docker exec -it <name> sh                    # Shell into container
docker cp <name>:/path ./local               # Copy from container
docker stats                                 # Resource usage
docker inspect <name>                        # Full details
```

### Compose Commands
```bash
docker compose up -d                         # Start all services
docker compose down                          # Stop and remove
docker compose down -v                       # Include volumes
docker compose logs -f                       # Follow all logs
docker compose ps                            # List services
docker compose exec <svc> sh                 # Shell into service
docker compose build                         # Rebuild images
docker compose pull                          # Pull latest images
docker compose up -d --scale api=3           # Scale service
```

### Network Commands
```bash
docker network ls                            # List networks
docker network create my-net                 # Create network
docker network inspect my-net                # Inspect network
docker network rm my-net                     # Remove network
docker network connect <net> <container>      # Connect container
docker network disconnect <net> <container>   # Disconnect container
```

### Volume Commands
```bash
docker volume ls                             # List volumes
docker volume create my-vol                  # Create volume
docker volume inspect my-vol                 # Inspect volume
docker volume rm my-vol                      # Remove volume
docker volume prune                          # Remove unused volumes
```

### System Commands
```bash
docker system df                             # Disk usage
docker system prune                          # Clean unused
docker system prune -a --volumes             # Nuclear cleanup
docker info                                  # System info
```

---

## 🎯 Interview Questions — Docker + ECS

### Beginner
1. What is the difference between a Docker image and a container?
2. What is the difference between CMD and ENTRYPOINT?
3. How does Docker differ from a virtual machine?
4. What is a Dockerfile and what are the key instructions?
5. How do you pass environment variables to a Docker container?

### Intermediate
6. What are multi-stage builds and why are they important?
7. Explain Docker layer caching and how to optimize it.
8. How does Docker networking work? What is the bridge network?
9. What is the difference between a volume and a bind mount?
10. How do you debug a container that keeps crashing?

### Advanced
11. How does Docker map to ECS task definitions? (Explain the 1:1 mapping)
12. What happens when you run `docker run`? (Explain the full flow: CLI → daemon → containerd → runc)
13. How would you migrate a docker-compose.yml to ECS Fargate?
14. What is the difference between Docker on EC2 launch type vs Fargate?
15. How do you optimize Docker images for production ECS workloads?

### Scenario-Based
16. Your ECS task keeps getting OOM killed. How do you debug this locally with Docker?
17. You need to run a database alongside your API in development. How do you set this up with Docker Compose, and how would this differ in production ECS?
18. Your Docker build is slow. How do you optimize the Dockerfile for faster builds?
19. You need to share a Docker image with your team. Walk through the process of pushing to ECR and pulling on another machine.
20. How would you implement a blue/green deployment using Docker + ECS?

---

## 📚 Recommended Learning Path

| Week | Focus | What to Do |
|------|-------|-----------|
| **1** | Docker Basics | Install Docker, run containers, pull images (Modules 1-3) |
| **2** | Dockerfile & Building | Write Dockerfiles, build images, multi-stage builds (Modules 4-5) |
| **3** | Running & Managing | Run containers, networking, volumes, Compose (Modules 6-10) |
| **4** | Production & ECS | Best practices, CI/CD, ECS migration (Modules 11-18) |
| **5** | Practice | Complete all hands-on labs, interview questions (Module 20) |

---

*Built with ❤️ by Cline SR for Prem — Docker from scratch, mapped to what you already know*
