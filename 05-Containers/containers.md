# 📦 Docker Containers

> **"Container = running image. Bas itna samajh lo, aur sab samajh aayega."**

---

## 🤔 What is a Container?

A container is a **running instance of a Docker image**. It's an isolated process on your host machine that:
- Has its own filesystem (from the image)
- Has its own network interface
- Has its own process space
- Shares the host OS kernel (unlike VMs)

> 🏠 **Real-life analogy:** Image = Apartment blueprint. Container = An actual family living in an apartment built from that blueprint. Multiple families (containers) can live in identical apartments (containers from same image) at the same time.

---

## 🔄 Container Lifecycle

```
         docker create
Image ──────────────────► Created
                              │
                   docker start│
                              │
                              ▼
                           Running ◄──── docker restart
                              │
              docker stop / docker kill
                              │
                              ▼
                           Stopped
                              │
                    docker start (again)
                              │
                              ▼
                           Running
                              │
                         docker rm
                              │
                              ▼
                           Deleted
```

### States Explained:

| State | Description | Command to reach it |
|-------|-------------|---------------------|
| **Created** | Container exists but not started | `docker create` |
| **Running** | Container process is active | `docker start` or `docker run` |
| **Paused** | Process is frozen (SIGSTOP) | `docker pause` |
| **Stopped** (Exited) | Process has ended | `docker stop` |
| **Deleted** | Container removed from system | `docker rm` |

---

## 🆚 Image vs Container — The Key Difference

| Aspect | Image | Container |
|--------|-------|-----------|
| State | Static, read-only | Dynamic, has writable layer |
| Storage | Shared layers on disk | Thin writable layer on top |
| Count | One image | Many containers from one image |
| Lifecycle | Permanent until deleted | Temporary by nature |
| Analogy | Cake recipe | Actual baked cake |

```bash
# One image → Multiple containers
docker run -d -p 8081:80 --name web1 nginx
docker run -d -p 8082:80 --name web2 nginx
docker run -d -p 8083:80 --name web3 nginx

# All three run independently from the same nginx image
docker ps
# NAMES   IMAGE   PORTS
# web1    nginx   0.0.0.0:8081->80/tcp
# web2    nginx   0.0.0.0:8082->80/tcp
# web3    nginx   0.0.0.0:8083->80/tcp
```

---

## 🔒 Container Isolation

Each container is isolated using Linux kernel features:

### 1. Namespaces (Process Isolation)
- **PID namespace:** Container sees only its own processes
- **Network namespace:** Container has its own network stack
- **Mount namespace:** Container has its own filesystem view
- **UTS namespace:** Container has its own hostname

### 2. Control Groups / cgroups (Resource Limits)
```bash
# Limit container to 512MB RAM and 1 CPU
docker run -d \
  --memory="512m" \
  --memory-swap="1g" \
  --cpus="1.0" \
  --name limited-container \
  nginx

# Verify limits
docker stats limited-container
```

### 3. Union Filesystem (OverlayFS)
```
Container writable layer (your changes)
─────────────────────────────────────
Image Layer 4 (COPY . .)         read-only
─────────────────────────────────────
Image Layer 3 (RUN npm install)  read-only
─────────────────────────────────────
Image Layer 2 (WORKDIR /app)     read-only
─────────────────────────────────────
Image Layer 1 (FROM node:alpine) read-only
```

When you write a file in a container, it's written to the **writable layer** using Copy-on-Write (CoW). The original image layers are never modified.

---

## 🛠️ Container Operations

### Creating and Running

```bash
# Create without starting (useful for inspecting before run)
docker create --name test-container nginx

# Start a created container
docker start test-container

# Run (create + start combined — most common)
docker run -d --name my-app -p 3000:3000 my-image

# Interactive containers (get a shell)
docker run -it ubuntu bash
docker run -it --rm alpine sh   # --rm removes container on exit
```

### Monitoring Containers

```bash
# Real-time resource usage
docker stats

# Stats for specific container
docker stats my-app --no-stream   # single snapshot

# See processes inside container
docker top my-app

# Inspect full details (JSON)
docker inspect my-app

# Get IP address
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' my-app

# Follow logs
docker logs -f --tail=100 my-app
```

### Interacting with Running Containers

```bash
# Execute a command inside container
docker exec my-app ls /app

# Get interactive shell
docker exec -it my-app bash
docker exec -it my-app sh        # for alpine

# Run as root (even if container uses non-root user)
docker exec -it --user root my-app bash

# Copy files to/from container
docker cp ./config.json my-app:/app/config.json
docker cp my-app:/app/logs/app.log ./
```

### Stopping and Removing

```bash
# Graceful stop (SIGTERM → wait 10s → SIGKILL)
docker stop my-app

# Immediate stop (SIGKILL)
docker kill my-app

# Remove stopped container
docker rm my-app

# Force remove running container
docker rm -f my-app

# Remove all stopped containers
docker container prune

# Stop and remove in one line
docker rm -f my-app
```

---

## ♻️ Restart Policies

```bash
# No restart (default)
docker run --restart=no my-app

# Always restart (even after docker daemon restart)
docker run --restart=always my-app

# Restart unless manually stopped
docker run --restart=unless-stopped my-app

# Restart on failure (with max retries)
docker run --restart=on-failure:5 my-app
```

| Policy | Behavior |
|--------|----------|
| `no` | Never restart (default) |
| `always` | Always restart, including on daemon restart |
| `unless-stopped` | Always restart unless manually stopped |
| `on-failure[:max]` | Only restart on non-zero exit code |

> 💡 **Production tip:** Use `--restart=unless-stopped` for web services. This way containers survive server reboots but you can still manually stop them for maintenance.

---

## 🌡️ Container Health Checks

```bash
# Define health check in docker run
docker run -d \
  --name web \
  --health-cmd="curl -f http://localhost/ || exit 1" \
  --health-interval=30s \
  --health-timeout=10s \
  --health-retries=3 \
  nginx

# Check health status
docker inspect --format='{{.State.Health.Status}}' web
# → healthy / unhealthy / starting

# See health check history
docker inspect --format='{{json .State.Health}}' web | python3 -m json.tool
```

---

## 📊 Container Data Persistence

By default, all data written inside a container is **lost when the container is removed**.

```bash
# ❌ Data lost when container removed
docker run -d mysql   # database data is inside container
docker rm -f mysql    # EVERYTHING IS GONE

# ✅ Data persists with a volume
docker run -d -v mysql-data:/var/lib/mysql mysql
docker rm -f mysql    # container gone, but data in 'mysql-data' volume survives
docker run -d -v mysql-data:/var/lib/mysql mysql   # data is back!
```

See [Volumes section](../07-Docker-Volumes/volumes.md) for complete details.

---

## 🎨 Naming Conventions (Best Practice)

```bash
# Use descriptive, environment-prefixed names
docker run -d --name prod-nginx nginx
docker run -d --name staging-api my-api
docker run -d --name dev-postgres postgres

# For microservices
docker run -d --name auth-service auth:v2
docker run -d --name payment-service payment:v1.3
docker run -d --name user-service user:latest
```

---

## ✅ Things to Remember

- Containers are **ephemeral** by design — design apps to expect restarts
- A stopped container still exists (`docker ps -a`) until you `docker rm` it
- Container filesystem changes are lost when container is removed (unless using volumes)
- One process per container is best practice
- Use `--restart=unless-stopped` for production services
- `docker exec -it container bash` is your best debugging tool

---

**Next:** [📝 Dockerfile →](../06-Dockerfile/dockerfile.md)
