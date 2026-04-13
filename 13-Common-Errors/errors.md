# 🐛 Common Docker Errors & Fixes

> **"Error aaya? Ghabrao mat — yahan solution hai."**

---

## Error 1: Permission Denied

```
Got permission denied while trying to connect to the Docker daemon socket
at unix:///var/run/docker.sock
```

**Cause:** Your user is not in the `docker` group.

**Fix:**
```bash
sudo usermod -aG docker $USER
newgrp docker           # apply without logout
# OR logout and login again
```

---

## Error 2: Port Already in Use

```
Error: bind: address already in use
docker: Error response from daemon: Ports are not available:
listen tcp 0.0.0.0:8080: bind: address already in use.
```

**Fix:**
```bash
# Find what's using the port
sudo lsof -i :8080
sudo ss -tlnp | grep 8080

# Kill the process
sudo kill -9 <PID>

# OR use a different port
docker run -p 8081:80 nginx
```

---

## Error 3: No Space Left on Device

```
Error response from daemon: no space left on device
```

**Fix:**
```bash
# Check Docker disk usage
docker system df

# Clean up aggressively
docker system prune -a -f --volumes

# Check actual disk space
df -h

# If /var/lib/docker is on a full partition, move Docker's data root:
# Edit /etc/docker/daemon.json:
# { "data-root": "/mnt/larger-disk/docker" }
# sudo systemctl restart docker
```

---

## Error 4: Container Exits Immediately

```
docker run my-app
# Container starts and stops within seconds
docker ps -a
# STATUS: Exited (1) 2 seconds ago
```

**Causes & Fixes:**
```bash
# 1. Check logs to see the error
docker logs <container-id>

# 2. Common cause: CMD runs and exits
# Wrong: The process is not long-running
CMD ["echo", "Hello"]     # ❌ exits after printing

# Fix: Ensure your main process keeps running
CMD ["node", "server.js"]  # ✅ server stays running

# 3. Run interactively to debug
docker run -it my-app bash

# 4. Check entrypoint/cmd
docker inspect --format='{{.Config.Cmd}}' my-app
```

---

## Error 5: Image Not Found

```
Unable to find image 'myapp:latest' locally
docker: Error response from daemon: pull access denied for myapp,
repository does not exist or may require 'docker login'
```

**Fix:**
```bash
# Check if image exists locally
docker images | grep myapp

# Build it first
docker build -t myapp:latest .

# If it's a private registry image, login first
docker login registry.example.com
docker pull registry.example.com/myapp:latest
```

---

## Error 6: Container Can't Connect to Database

```
Error: connect ECONNREFUSED 127.0.0.1:5432
# or
MongoNetworkError: failed to connect to server [localhost:27017]
```

**Cause:** Using `localhost` to connect to another container — doesn't work!

**Fix:**
```bash
# Use the container/service NAME as the hostname
# Wrong:
-e DATABASE_URL=postgresql://user:pass@localhost:5432/db   # ❌

# Right (using container name or service name):
-e DATABASE_URL=postgresql://user:pass@postgres:5432/db    # ✅

# Both containers must be on the same Docker network
docker network create app-net
docker run -d --name postgres --network app-net postgres
docker run -d --name api --network app-net my-api
```

---

## Error 7: Volume Mount Fails / Permission Denied Inside Container

```
Error: EACCES: permission denied, open '/app/data/file.txt'
```

**Fix:**
```bash
# Option 1: Fix permissions on the host directory
sudo chown -R 1001:1001 ./data/

# Option 2: Run container as root (not recommended for prod)
docker run --user root my-app

# Option 3: Fix in Dockerfile
RUN mkdir -p /app/data && chown -R node:node /app/data
USER node

# Option 4: Pre-create volume with correct permissions
docker run --rm -v myvolume:/data alpine chown -R 1001:1001 /data
```

---

## Error 8: Build Fails — COPY File Not Found

```
COPY failed: file not found in build context or excluded by .dockerignore: stat file.txt
```

**Fix:**
```bash
# 1. Ensure file exists in build context directory
ls -la file.txt

# 2. Check .dockerignore isn't accidentally excluding it
cat .dockerignore

# 3. Ensure you're running docker build from the right directory
docker build -t myapp .        # . means current directory is context
docker build -t myapp ./app/   # use a different context

# 4. Check file path in Dockerfile is relative to context
COPY ./src /app/src    # ✅
COPY ../other /app/    # ❌ can't go above context directory
```

---

## Error 9: Out of Memory / Container Killed

```
container killed due to OOM (Out Of Memory)
```

**Fix:**
```bash
# Check if container was OOM killed
docker inspect container | grep -i oom

# Increase memory limit
docker run -d --memory="1g" my-app

# Check container memory usage
docker stats container --no-stream

# For compose:
services:
  api:
    deploy:
      resources:
        limits:
          memory: 1G
```

---

## Error 10: Docker Compose — Service Unhealthy

```
ERROR: Service 'api' failed to build: ...
# or container keeps restarting
```

**Fix:**
```bash
# Check logs of the failing service
docker compose logs api

# Check health check status
docker inspect api-container | grep -A 10 Health

# Debug: override CMD to keep it running for inspection
docker compose run --rm --entrypoint sh api

# Check depends_on conditions
# Ensure DB health check is defined if using condition: service_healthy
```

---

## Error 11: `docker compose` Command Not Found

```
bash: docker-compose: command not found
```

**Fix:**
```bash
# Docker Compose v2 is now a plugin, not a separate binary
# Use: docker compose (no hyphen)
docker compose up -d    # ✅

# If you need v1 compatibility
pip install docker-compose   # install separately
docker-compose up -d         # old syntax

# Install Compose plugin for Docker Engine
sudo apt-get install docker-compose-plugin
```

---

## Error 12: Cannot Remove — Container Has Dependent Child

```
Error response from daemon: conflict: unable to delete image
(cannot be forced) - image has dependent child images
```

**Fix:**
```bash
# Find containers using this image
docker ps -a --filter ancestor=image-name

# Remove containers first
docker rm -f $(docker ps -aq --filter ancestor=image-name)

# Then remove image
docker rmi image-name

# Force remove with all tags
docker rmi -f image-name
```

---

## Error 13: Dockerfile RUN Command Fails

```
The command '/bin/sh -c apt-get install curl' returned a non-zero code: 100
```

**Fix:**
```bash
# Always update apt before installing
RUN apt-get update && apt-get install -y curl   # ✅

# Don't separate update and install (cache issue)
RUN apt-get update
RUN apt-get install -y curl   # ❌ update might be stale cache

# For Alpine (apk):
RUN apk add --no-cache curl
```

---

## Error 14: SSL/TLS Certificate Errors

```
curl: (60) SSL certificate problem: certificate has expired
x509: certificate has expired or is not yet valid
```

**Fix:**
```bash
# Update certificates in container
RUN apt-get update && apt-get install -y ca-certificates

# For Alpine
RUN apk add --no-cache ca-certificates

# Temporary workaround (not for production)
curl -k https://...   # skip verification
```

---

## 🔧 Debugging Toolkit

```bash
# 1. Full logs
docker logs -f --tail=200 container-name

# 2. Get a shell in running container
docker exec -it container-name bash

# 3. Get a shell in stopped/crashed container
docker run -it --entrypoint bash image-name

# 4. Check all container details
docker inspect container-name | less

# 5. Check network connectivity from inside container
docker exec container-name ping db-container
docker exec container-name curl http://other-service:3000/health
docker exec container-name nslookup db-container

# 6. Check process is running
docker top container-name

# 7. Monitor events in real time
docker events --filter container=my-app
```

---

**Next:** [🔐 Docker Security →](../14-Docker-Security/security.md)
