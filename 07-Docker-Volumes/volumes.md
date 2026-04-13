# 💾 Docker Volumes & Storage

> **"Data save karna hai? Volume use karo. Nahi karna? Phir bhi volume use karo."**

---

## 🤔 Why Do We Need Volumes?

By default, Docker containers are **stateless and ephemeral**:
- All files created inside a container are stored in the container's writable layer
- When the container is removed (`docker rm`), ALL data is permanently lost
- Data cannot be easily shared between containers

> 🏦 **Real-life analogy:** A container without a volume is like a bank that keeps money in RAM — the moment you turn it off, everything is gone. Volumes are the hard drives that persist data safely.

---

## 📦 Types of Docker Storage

```
┌──────────────────────────────────────────────────────────┐
│                    DATA STORAGE OPTIONS                  │
│                                                          │
│  ┌─────────────────┐  ┌─────────────┐  ┌─────────────┐  │
│  │  Named Volumes  │  │ Bind Mounts │  │  tmpfs      │  │
│  │                 │  │             │  │  (memory)   │  │
│  │ Managed by      │  │ Host path   │  │             │  │
│  │ Docker          │  │ mapped to   │  │ Temporary,  │  │
│  │                 │  │ container   │  │ in-memory   │  │
│  │ Best for: DB    │  │             │  │             │  │
│  │ persistent data │  │ Best for:   │  │ Best for:   │  │
│  │                 │  │ Dev, config │  │ Secrets,    │  │
│  │ ✅ Recommended  │  │ files       │  │ sensitive   │  │
│  └─────────────────┘  └─────────────┘  │ data        │  │
│                                        └─────────────┘  │
└──────────────────────────────────────────────────────────┘
```

---

## 1️⃣ Named Volumes (Recommended for Production)

Named volumes are managed entirely by Docker. Docker decides where to store them on the host.

### Creating and Using Named Volumes:

```bash
# Create a named volume
docker volume create mysql-data

# Use it when running a container
docker run -d \
  --name mysql-db \
  -e MYSQL_ROOT_PASSWORD=secret \
  -v mysql-data:/var/lib/mysql \
  mysql:8.0

# Even if container is removed, data persists!
docker rm -f mysql-db
docker run -d \
  --name mysql-db-new \
  -e MYSQL_ROOT_PASSWORD=secret \
  -v mysql-data:/var/lib/mysql \   # same volume = same data!
  mysql:8.0
```

### Volume Commands:

```bash
# List all volumes
docker volume ls

# Inspect a volume (see where data is stored)
docker volume inspect mysql-data
# Output shows: Mountpoint: /var/lib/docker/volumes/mysql-data/_data

# Remove a volume
docker volume rm mysql-data

# Remove ALL unused volumes
docker volume prune
docker volume prune -f   # skip prompt

# Backup a volume
docker run --rm \
  -v mysql-data:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/mysql-backup.tar.gz /data

# Restore a volume
docker run --rm \
  -v mysql-data:/data \
  -v $(pwd):/backup \
  alpine tar xzf /backup/mysql-backup.tar.gz -C /
```

---

## 2️⃣ Bind Mounts

Bind mounts map a specific path on your **host machine** to a path inside the container.

```bash
# Syntax: -v /host/path:/container/path
docker run -d \
  -v /home/user/myapp:/app \
  -p 3000:3000 \
  node:18-alpine \
  node /app/index.js

# Read-only bind mount
docker run -d \
  -v /etc/nginx/nginx.conf:/etc/nginx/nginx.conf:ro \
  nginx

# Mount current directory (development workflow)
docker run -d \
  -v $(pwd):/app \
  -p 3000:3000 \
  node:18-alpine \
  node /app/index.js
```

### Bind Mount in Docker Compose (for development):
```yaml
services:
  api:
    image: node:18-alpine
    volumes:
      - ./src:/app/src      # source code hot-reload
      - ./config:/app/config:ro   # read-only config
    working_dir: /app
    command: node src/index.js
```

---

## 3️⃣ tmpfs Mounts (In-Memory)

Stored in host memory only. Data is lost when container stops. Use for sensitive data.

```bash
docker run -d \
  --tmpfs /tmp:rw,noexec,nosuid,size=100m \
  nginx

# Or with --mount syntax:
docker run -d \
  --mount type=tmpfs,destination=/tmp,tmpfs-size=100m \
  nginx
```

---

## 🆚 Volume vs Bind Mount Comparison

| Feature | Named Volume | Bind Mount |
|---------|-------------|------------|
| **Who manages it?** | Docker | You (host path) |
| **Host path** | `/var/lib/docker/volumes/` | Any path you specify |
| **Portability** | High (works on any Docker host) | Low (depends on host path existing) |
| **Best for** | Databases, persistent app data | Dev workflows, config files |
| **Performance** | Slightly better | Good |
| **Backup** | `docker run` trick | Direct host file access |
| **Windows/Mac** | Works great | Path issues possible |

---

## 🔧 Volume Syntax Variations

### Old `-v` syntax:
```bash
# Named volume
-v volume-name:/container/path

# Bind mount
-v /host/path:/container/path

# Bind mount, read-only
-v /host/path:/container/path:ro

# Anonymous volume
-v /container/path
```

### New `--mount` syntax (explicit, preferred):
```bash
# Named volume
--mount type=volume,source=mysql-data,target=/var/lib/mysql

# Bind mount
--mount type=bind,source=/host/path,target=/app

# Read-only bind mount
--mount type=bind,source=/host/config,target=/app/config,readonly

# In-memory
--mount type=tmpfs,destination=/tmp
```

---

## 🗄️ Real-World Examples

### MySQL with persistent storage:
```bash
docker run -d \
  --name production-mysql \
  -e MYSQL_ROOT_PASSWORD=secure_password \
  -e MYSQL_DATABASE=myapp \
  -v mysql-prod-data:/var/lib/mysql \
  --restart=unless-stopped \
  mysql:8.0
```

### PostgreSQL with backup:
```bash
# Run postgres
docker run -d \
  --name pg-db \
  -e POSTGRES_PASSWORD=secret \
  -v pg-data:/var/lib/postgresql/data \
  postgres:15

# Backup database
docker exec pg-db pg_dump -U postgres mydb > backup.sql

# Volume backup (entire data dir)
docker run --rm \
  -v pg-data:/source \
  -v $(pwd):/dest \
  alpine tar czf /dest/pg-backup.tar.gz -C /source .
```

### Nginx with custom config (bind mount):
```bash
docker run -d \
  --name web \
  -p 80:80 \
  -v /etc/nginx/nginx.conf:/etc/nginx/nginx.conf:ro \
  -v /var/www/html:/usr/share/nginx/html:ro \
  nginx
```

### Development: Live code reloading:
```bash
# Node.js with nodemon for auto-restart on file changes
docker run -d \
  --name dev-api \
  -p 3000:3000 \
  -v $(pwd):/app \
  -w /app \
  node:18-alpine \
  sh -c "npm install -g nodemon && nodemon src/index.js"
```

---

## 🔒 Volume Permissions

```bash
# Common issue: container runs as non-root, can't write to mounted volume
docker run -d \
  --user 1001:1001 \   # container user ID
  -v mydata:/data \
  myapp

# Fix: set permissions on the volume directory
docker run --rm \
  -v mydata:/data \
  alpine chown -R 1001:1001 /data

# In Dockerfile: set ownership when copying files
COPY --chown=node:node . .
```

---

## 💡 Best Practices

```bash
# 1. Always use named volumes for databases (not bind mounts)
-v postgres-data:/var/lib/postgresql/data  # ✅
-v /home/ubuntu/pgdata:/var/lib/postgresql/data  # ❌ (environment-dependent)

# 2. Use bind mounts for development (live code reload)
-v $(pwd)/src:/app/src  # ✅ great for dev

# 3. Mount config files as read-only
-v ./nginx.conf:/etc/nginx/nginx.conf:ro  # ✅

# 4. Never use volumes for secrets — use Docker Secrets or env vars
# (volumes are visible on the filesystem)

# 5. Regular volume backups for production databases

# 6. Use docker-compose.yml for volume definitions (easier management)
```

---

## ✅ Things to Remember

- Container data is **ephemeral** — lost when container removed
- Named volumes = Docker manages location — best for databases
- Bind mounts = you control path — best for development
- `docker volume prune` can delete important data — use with care!
- Always back up volumes before `docker system prune`
- Volume data persists even after `docker rm -f container`

---

**Next:** [🌐 Docker Networking →](../08-Docker-Networking/networking.md)
