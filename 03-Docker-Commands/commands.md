# 💻 Docker Commands — Complete Reference

> **"Commands yaad karne se nahi, samajhne se aate hain."**

---

## 🗺️ Command Categories

```
Docker Commands
├── Container Commands   → run, start, stop, rm, exec, logs, ps
├── Image Commands       → pull, push, build, images, rmi, tag
├── Volume Commands      → volume create/ls/rm/inspect
├── Network Commands     → network create/ls/rm/inspect
├── System Commands      → info, df, prune, events
└── Registry Commands    → login, logout, push, pull
```

---

## 📦 Container Commands

### `docker run` — Create and Start a Container

The most important Docker command. Combines `create` + `start`.

```bash
# Basic syntax
docker run [OPTIONS] IMAGE [COMMAND]

# Run nginx (foreground — blocks terminal)
docker run nginx

# Run in detached mode (background) — most common
docker run -d nginx

# Run with a name
docker run -d --name my-nginx nginx

# Map ports: host:container
docker run -d -p 8080:80 --name web nginx

# Run and auto-remove when stopped
docker run --rm ubuntu echo "Hello World"

# Run interactively (get a shell inside container)
docker run -it ubuntu bash
docker run -it ubuntu /bin/sh        # for alpine-based images

# Pass environment variables
docker run -d -e MYSQL_ROOT_PASSWORD=secret mysql

# Mount a volume
docker run -d -v myvolume:/data nginx
docker run -d -v /host/path:/container/path nginx

# Set resource limits
docker run -d --memory="512m" --cpus="1.0" nginx

# Run with restart policy
docker run -d --restart=always nginx
docker run -d --restart=unless-stopped nginx
docker run -d --restart=on-failure:3 nginx

# Full real-world example
docker run -d \
  --name api-server \
  -p 3000:3000 \
  -e NODE_ENV=production \
  -e DB_HOST=mongodb \
  --memory="256m" \
  --restart=unless-stopped \
  my-api:v2
```

### `docker ps` — List Containers

```bash
# Show only running containers
docker ps

# Show ALL containers (including stopped)
docker ps -a
docker ps --all

# Show only container IDs
docker ps -q

# Custom format output
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Show last N containers
docker ps -n 5
```

**Output columns explained:**
```
CONTAINER ID   IMAGE     COMMAND    CREATED      STATUS       PORTS              NAMES
a1b2c3d4e5f6   nginx     "/docker…" 5 mins ago   Up 5 mins    0.0.0.0:80->80/tcp web-server
```

### `docker start` / `docker stop` / `docker restart`

```bash
# Start a stopped container
docker start my-nginx
docker start a1b2c3d4e5f6   # using container ID

# Stop a running container (graceful — sends SIGTERM, waits 10s, then SIGKILL)
docker stop my-nginx

# Stop with custom timeout (seconds)
docker stop --time=30 my-nginx

# Force kill immediately (SIGKILL)
docker kill my-nginx

# Restart a container
docker restart my-nginx
docker restart --time=5 my-nginx   # wait 5s before restart

# Stop ALL running containers
docker stop $(docker ps -q)
```

### `docker exec` — Run Commands Inside a Container

```bash
# Get an interactive bash shell inside running container
docker exec -it my-nginx bash
docker exec -it my-nginx /bin/sh    # alpine containers

# Run a single command
docker exec my-nginx nginx -t      # test nginx config
docker exec my-nginx ls /etc/nginx

# Run as specific user
docker exec -it --user root my-nginx bash

# Set environment variable for the exec session
docker exec -it -e DEBUG=true my-nginx bash

# Real-world use: check a file inside container
docker exec my-api cat /app/config.json

# Real-world use: run a database backup inside container
docker exec mysql-container mysqldump -u root -psecret mydb > backup.sql
```

### `docker logs` — View Container Logs

```bash
# View all logs
docker logs my-nginx

# Follow logs in real-time (like tail -f)
docker logs -f my-nginx
docker logs --follow my-nginx

# Show last N lines
docker logs --tail=100 my-nginx
docker logs -n 50 my-nginx

# Show logs with timestamps
docker logs -t my-nginx

# Combine: follow last 50 lines with timestamps
docker logs -f --tail=50 -t my-nginx

# Show logs since a time
docker logs --since="2024-01-15T10:00:00" my-nginx
docker logs --since=30m my-nginx   # last 30 minutes
docker logs --until=1h my-nginx    # up to 1 hour ago
```

### `docker rm` — Remove Containers

```bash
# Remove a stopped container
docker rm my-nginx

# Force remove a RUNNING container
docker rm -f my-nginx

# Remove multiple containers
docker rm container1 container2 container3

# Remove ALL stopped containers
docker container prune
docker container prune -f   # skip confirmation prompt

# Remove ALL containers (running + stopped) — be careful!
docker rm -f $(docker ps -aq)
```

### `docker inspect` — Get Detailed Info

```bash
# Full JSON details about a container
docker inspect my-nginx

# Get specific fields with Go template
docker inspect --format='{{.NetworkSettings.IPAddress}}' my-nginx
docker inspect --format='{{.State.Status}}' my-nginx
docker inspect --format='{{.HostConfig.RestartPolicy.Name}}' my-nginx

# Inspect an image
docker inspect nginx:latest
```

### Other Useful Container Commands

```bash
# Copy files between host and container
docker cp myfile.txt my-nginx:/etc/nginx/myfile.txt
docker cp my-nginx:/etc/nginx/nginx.conf ./nginx.conf

# Show resource usage (CPU, Memory, Network I/O)
docker stats
docker stats my-nginx
docker stats --no-stream   # snapshot (not live)

# Pause / Unpause a container
docker pause my-nginx
docker unpause my-nginx

# Show processes running inside container
docker top my-nginx

# Show port mappings
docker port my-nginx

# Attach to a running container's stdout
docker attach my-nginx   # Ctrl+C will stop the container!
```

---

## 🖼️ Image Commands

```bash
# List local images
docker images
docker image ls

# Pull an image from Docker Hub
docker pull nginx
docker pull nginx:1.25       # specific version
docker pull ubuntu:22.04
docker pull python:3.11-slim

# Remove an image
docker rmi nginx
docker rmi nginx:latest
docker rmi -f nginx   # force remove (even if container exists)

# Remove ALL unused images
docker image prune
docker image prune -a   # remove ALL images not used by any container

# Tag an image
docker tag my-app:latest yourusername/my-app:v1.0

# Search Docker Hub
docker search nginx
docker search --filter=stars=100 node   # filter by stars

# Show image build history (layers)
docker image history nginx

# Save image to tar file (offline transfer)
docker save -o nginx-backup.tar nginx:latest

# Load image from tar file
docker load -i nginx-backup.tar
```

---

## 🔨 Build Commands

```bash
# Build image from Dockerfile in current directory
docker build -t my-app:v1 .

# Build from specific Dockerfile
docker build -f Dockerfile.prod -t my-app:prod .

# Build with build arguments
docker build --build-arg NODE_ENV=production -t my-app .

# Build with no cache (fresh build)
docker build --no-cache -t my-app .

# Build and push (multi-platform)
docker buildx build --platform linux/amd64,linux/arm64 -t myrepo/app:latest --push .
```

---

## 🌐 Network Commands

```bash
docker network ls                        # list networks
docker network create my-network         # create bridge network
docker network create --driver host h-net
docker network inspect bridge            # inspect network details
docker network connect my-network my-container
docker network disconnect my-network my-container
docker network rm my-network             # remove network
docker network prune                     # remove all unused networks
```

---

## 💾 Volume Commands

```bash
docker volume ls                         # list volumes
docker volume create my-vol              # create volume
docker volume inspect my-vol             # inspect volume
docker volume rm my-vol                  # remove volume
docker volume prune                      # remove all unused volumes
```

---

## 🧹 System Commands

```bash
# System information
docker info
docker version

# Disk usage
docker system df
docker system df -v   # verbose

# Remove ALL unused resources (containers, images, networks, volumes)
docker system prune
docker system prune -a        # include all unused images
docker system prune -a -f     # skip confirmation

# Real-time events
docker events
docker events --filter type=container
```

---

## 🔐 Registry Commands

```bash
# Login to Docker Hub
docker login
docker login -u username -p password   # non-interactive

# Login to private registry
docker login registry.example.com

# Logout
docker logout

# Push image to Docker Hub
docker push yourusername/my-app:v1

# Push to private registry
docker tag my-app registry.example.com/my-app:v1
docker push registry.example.com/my-app:v1
```

---

## 💡 Real-World Command Combos

```bash
# Clean up everything on a server (free disk space)
docker system prune -a -f --volumes

# Get IP of a container
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' container_name

# Export container filesystem
docker export my-container | gzip > container-backup.tar.gz

# Find which containers are using a specific image
docker ps --filter ancestor=nginx

# Check if a container is healthy
docker inspect --format='{{.State.Health.Status}}' my-container

# Get environment variables of a running container
docker inspect --format='{{range .Config.Env}}{{println .}}{{end}}' my-container
```

---

## 🧠 Things to Remember

- `docker run` = `docker create` + `docker start`
- `-d` = detached (background), `-it` = interactive terminal
- `-p host:container` = port mapping
- `-v host:container` = volume mount
- Container ID can be shortened — first 3+ unique characters work
- `docker ps -q` gives just IDs — useful for scripting

---

**Next:** [🖼️ Docker Images →](../04-Images/images.md)
