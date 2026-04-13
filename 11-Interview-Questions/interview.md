# 🎯 Docker Interview Questions

> **"Interview mein Docker ke questions aate hain — aur yahan sab answers hain."**

---

## 🟢 Beginner Level

### Q1. What is Docker and why is it used?

**Answer:**
Docker is an open-source containerization platform that packages applications and their dependencies into isolated, portable units called containers. It's used because:
- Eliminates "works on my machine" problems
- Faster than Virtual Machines (milliseconds vs minutes startup)
- Consistent environments across dev, staging, and production
- Resource-efficient — multiple containers share the host OS kernel
- Foundation for microservices architecture

---

### Q2. What is the difference between a Docker Image and a Docker Container?

**Answer:**

| Aspect | Image | Container |
|--------|-------|-----------|
| Definition | Read-only template/blueprint | Running instance of an image |
| State | Static | Dynamic |
| Analogy | Class (in OOP) | Object / Instance |
| Storage | Shared read-only layers | Thin writable layer on top |
| Count | One image | Many containers from one image |

```bash
# Image → Container relationship
docker pull nginx           # get the image
docker run -d nginx         # create a container FROM the image
docker run -d nginx         # another container, same image
```

---

### Q3. What is the difference between Docker and a Virtual Machine?

**Answer:**

| Feature | Virtual Machine | Docker Container |
|---------|----------------|-----------------|
| OS | Full guest OS per VM | Shares host OS kernel |
| Size | GBs | MBs |
| Startup | Minutes | Milliseconds |
| Isolation | Full (hardware-level) | Process-level |
| Performance | ~70% native | ~98% native |
| Use case | Full OS isolation | App packaging |

VMs use a **Hypervisor** to virtualize hardware. Docker uses **Linux namespaces and cgroups** to isolate processes.

---

### Q4. What is Docker Hub?

**Answer:**
Docker Hub is the default public **container registry** — a cloud-based repository for Docker images. Think of it as GitHub but for Docker images. It hosts:
- Official images: `nginx`, `node`, `python`, `mysql`
- Community images: `username/image-name`
- Private repositories (paid plans)

```bash
docker pull nginx          # pulls from Docker Hub
docker push user/my-app    # pushes to Docker Hub
```

---

### Q5. What is a Dockerfile?

**Answer:**
A Dockerfile is a plain text file containing a series of instructions that Docker reads to automatically build a Docker image. Each instruction creates a new layer.

```dockerfile
FROM node:18-alpine     # base image
WORKDIR /app            # set working directory
COPY package*.json ./   # copy files
RUN npm install         # execute command
EXPOSE 3000             # document port
CMD ["node", "app.js"]  # default command
```

---

### Q6. Explain common Dockerfile instructions.

**Answer:**

| Instruction | Purpose |
|-------------|---------|
| `FROM` | Base image to start from |
| `WORKDIR` | Set working directory |
| `COPY` | Copy files from host to image |
| `RUN` | Execute commands during build |
| `CMD` | Default command (overridable at runtime) |
| `ENTRYPOINT` | Container's main executable |
| `EXPOSE` | Document which port app listens on |
| `ENV` | Set environment variables |
| `ARG` | Build-time variables |
| `VOLUME` | Declare mount points |
| `USER` | Set the running user |
| `HEALTHCHECK` | Define health check command |
| `LABEL` | Add metadata |

---

### Q7. What is the difference between CMD and ENTRYPOINT?

**Answer:**
- **CMD** provides default arguments that can be overridden at `docker run`
- **ENTRYPOINT** defines the container's main process — not easily overridden

```dockerfile
ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]   # default args, overridable

# docker run my-nginx              → runs: nginx -g "daemon off;"
# docker run my-nginx -c custom.conf → runs: nginx -c custom.conf
```

Best practice: Use `ENTRYPOINT` for the executable, `CMD` for default arguments.

---

## 🟡 Intermediate Level

### Q8. What is Docker image layering and why does it matter?

**Answer:**
Docker images are built from stacked, read-only layers. Each `RUN`, `COPY`, or `ADD` instruction creates a new layer.

**Why it matters:**
1. **Caching:** Unchanged layers are reused → faster builds
2. **Storage efficiency:** Layers shared between images are stored once
3. **Diffing:** Only changed layers are transferred when pushing/pulling

**Cache optimization tip:**
```dockerfile
# ✅ Copy package files before source code
COPY package*.json ./    # cached unless package.json changes
RUN npm install          # skipped if above unchanged
COPY . .                 # changes every commit, but npm install is cached
```

---

### Q9. What is a Docker Volume and why is it needed?

**Answer:**
Volumes provide **persistent storage** for containers. Without volumes, all data inside a container is lost when it's removed.

Types:
- **Named volumes:** Managed by Docker (`-v my-vol:/data`)
- **Bind mounts:** Host path mapped to container (`-v /host/path:/container/path`)
- **tmpfs:** In-memory, temporary

```bash
# Data persists even after container removal
docker run -d -v mysql-data:/var/lib/mysql mysql
docker rm -f mysql-container    # container gone
docker run -d -v mysql-data:/var/lib/mysql mysql  # data still there!
```

---

### Q10. What are Docker networking types?

**Answer:**

| Network | Description | Use Case |
|---------|-------------|----------|
| `bridge` (default) | Internal virtual network, IP-only comms | Basic dev/test |
| Custom bridge | DNS resolution by name | Multi-container apps |
| `host` | Shares host network stack | Max performance |
| `none` | No network | Security-isolated workloads |
| `overlay` | Multi-host networking | Docker Swarm |

**Key difference: Custom bridge enables DNS resolution:**
```bash
docker network create my-net
docker run -d --name db --network my-net postgres
docker run -d --name api --network my-net my-api
# api can reach db by name: postgresql://db:5432/mydb  ✅
```

---

### Q11. What is Docker Compose? When would you use it?

**Answer:**
Docker Compose is a tool for defining and running multi-container applications using a `docker-compose.yml` file. Use it when your application has multiple services (web server + database + cache) that need to start together.

```yaml
services:
  api:
    build: .
    ports: ["3000:3000"]
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: secret
    healthcheck:
      test: ["CMD-SHELL", "pg_isready"]
      interval: 10s
```

```bash
docker compose up -d    # start everything
docker compose down     # stop and remove
```

---

### Q12. What is the difference between `docker stop` and `docker kill`?

**Answer:**
- `docker stop` sends **SIGTERM** to the main process, waits for graceful shutdown (default 10s), then sends SIGKILL if still running
- `docker kill` sends **SIGKILL** immediately — no grace period

```bash
docker stop container    # graceful (SIGTERM → wait → SIGKILL)
docker stop -t 30 container  # wait 30 seconds
docker kill container    # immediate (SIGKILL)
```

Always prefer `docker stop` for production — gives the app time to close connections and flush data.

---

### Q13. What is `.dockerignore` and why is it important?

**Answer:**
`.dockerignore` is like `.gitignore` for Docker builds. It specifies files/directories to exclude from the build context sent to the Docker daemon.

**Why important:**
- Prevents large `node_modules` from being sent to daemon (faster builds)
- Prevents secrets (`.env` files) from being baked into images
- Reduces image size

```
node_modules
.git
.env
*.test.js
coverage/
```

---

## 🔴 Advanced Level

### Q14. What are multi-stage Dockerfile builds?

**Answer:**
Multi-stage builds use multiple `FROM` statements. Each stage starts fresh, and you copy only needed artifacts to the final stage — dramatically reducing image size.

```dockerfile
# Build stage (full toolchain)
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Production stage (minimal runtime)
FROM node:18-alpine AS production
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY package*.json ./
RUN npm ci --only=production
CMD ["node", "dist/server.js"]

# Result: ~100MB vs ~800MB without multi-stage
```

---

### Q15. How do you handle secrets in Docker? What NOT to do?

**Answer:**

**Never do this:**
```dockerfile
ENV DB_PASSWORD=secret123   # ❌ visible in docker history!
ARG API_KEY=xyz             # ❌ visible in build logs
```

**Safe approaches:**
```bash
# 1. Runtime environment variables
docker run -e DB_PASSWORD=secret my-app

# 2. Env file (not committed to git)
docker run --env-file .env my-app

# 3. Docker Secrets (Swarm)
echo "my_secret" | docker secret create db_password -
docker service create --secret db_password my-app

# 4. External secret managers (production)
# AWS Secrets Manager, HashiCorp Vault, Azure Key Vault
```

---

### Q16. What is the difference between COPY and ADD in Dockerfile?

**Answer:**

| Feature | COPY | ADD |
|---------|------|-----|
| Copy local files | ✅ | ✅ |
| Extract tar archives | ❌ | ✅ (automatic) |
| Download from URL | ❌ | ✅ (avoid!) |
| Recommended? | ✅ Always | Only for tar extraction |

**Best practice:** Use `COPY` unless you specifically need `ADD`'s tar extraction feature. URL downloading in `ADD` is bad practice — use `RUN curl` instead for better caching control.

---

### Q17. How do containers communicate with each other?

**Answer:**
Containers on the same **custom network** can communicate by service name (Docker's built-in DNS):

```bash
# Create network
docker network create app-net

# Start services on the same network
docker run -d --name postgres --network app-net postgres
docker run -d --name api --network app-net \
  -e DB_HOST=postgres \    # ← use container name as hostname
  my-api
```

In Docker Compose, this is automatic — all services are on a shared network by default.

---

### Q18. What is Docker's layer caching mechanism? How do you optimize for it?

**Answer:**
Docker caches each layer. If a layer and all layers before it are unchanged, Docker reuses the cache. **Any change invalidates all subsequent layers.**

**Optimization strategy — order instructions from least to most frequently changed:**
```dockerfile
FROM node:18-alpine          # never changes
WORKDIR /app                 # never changes
COPY package*.json ./        # changes only when deps change
RUN npm ci                   # cached unless package.json changed
COPY . .                     # changes with every commit
CMD ["node", "app.js"]       # rarely changes
```

Force rebuild without cache: `docker build --no-cache -t myapp .`

---

### Q19. How do you reduce Docker image size?

**Answer:**
1. **Use Alpine/slim base images:** `node:18-alpine` (127MB) vs `node:18` (1.1GB)
2. **Multi-stage builds:** Only copy final artifacts
3. **Chain RUN commands:** Fewer layers, clean up in same layer
4. **`.dockerignore`:** Exclude unnecessary files
5. **`--no-cache` in package installs:** `npm ci && npm cache clean --force`
6. **Remove build tools after use:**
   ```dockerfile
   RUN apt-get update && apt-get install -y gcc && \
       make build && \
       apt-get remove -y gcc && \
       apt-get autoremove -y && \
       rm -rf /var/lib/apt/lists/*
   ```
7. **Use `COPY --chown` instead of separate `RUN chown`**

---

### Q20. What is the difference between Docker Compose `depends_on` and health checks?

**Answer:**
`depends_on` alone only waits for the **container to start** — not for the service inside to be **ready**.

```yaml
# ❌ Problem: api starts, but postgres might not be accepting connections yet
depends_on:
  - postgres

# ✅ Solution: Wait until postgres is actually healthy
depends_on:
  postgres:
    condition: service_healthy

# And define the health check:
postgres:
  image: postgres:15
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U myuser"]
    interval: 10s
    timeout: 5s
    retries: 5
    start_period: 10s
```

This is one of the most common mistakes in real-world Docker Compose setups.

---

## 🏆 Bonus Questions

**Q: What happens when you run `docker run nginx`?**
1. Docker client sends request to Docker daemon
2. Daemon checks if `nginx` image exists locally
3. If not, pulls from Docker Hub (nginx:latest)
4. Creates a container from the image (adds writable layer)
5. Starts the container process
6. Returns container ID

**Q: Can you run multiple processes in one container?**
Technically yes, but it's an anti-pattern. Each container should run one process. For multiple processes, use supervisor, but prefer separate containers with Docker Compose.

**Q: What is the difference between `docker exec` and `docker attach`?**
- `docker exec`: Runs a NEW process inside the container — safe
- `docker attach`: Connects to the container's MAIN process stdin/stdout — pressing Ctrl+C kills the container!

---

**Next:** [📋 Cheatsheet →](../12-Cheatsheet/cheatsheet.md)
