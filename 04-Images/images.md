# 🖼️ Docker Images

> **"Image matlab blueprint — container uska ghar."**

---

## 🤔 What is a Docker Image?

A Docker image is a **read-only template** that contains:
- A base OS layer (Ubuntu, Alpine, Debian, or scratch)
- Application runtime (Node.js, Python, Java)
- Your application code
- Dependencies and libraries
- Configuration files
- Environment setup

> 🏗️ **Real-life analogy:** A Docker image is like the **architectural blueprint of a house**. The blueprint itself doesn't let you live in it — but every house (container) built from it is identical.

---

## 📚 Image Layers — The Most Important Concept

Docker images are built in **layers**. Each instruction in a Dockerfile creates a new layer.

```
┌──────────────────────────────────┐
│    Layer 5: CMD ["node","app"]   │  ← Your command
├──────────────────────────────────┤
│    Layer 4: COPY . /app          │  ← Your source code
├──────────────────────────────────┤
│    Layer 3: RUN npm install      │  ← Dependencies installed
├──────────────────────────────────┤
│    Layer 2: WORKDIR /app         │  ← Working directory set
├──────────────────────────────────┤
│    Layer 1: FROM node:18-alpine  │  ← Base image (pulled from Hub)
└──────────────────────────────────┘
         READ-ONLY LAYERS
```

### Why Layers Matter:

1. **Caching:** If Layer 3 hasn't changed, Docker reuses the cached layer → faster builds
2. **Sharing:** If two images share `FROM ubuntu:22.04`, that layer is stored once on disk → saves space
3. **Diffing:** Docker only stores the difference (diff) between layers → efficient storage

```bash
# View layers of an image
docker image history nginx

# Output:
# IMAGE          CREATED        CREATED BY                              SIZE
# a99a39d070bf   3 weeks ago    /bin/sh -c #(nop)  CMD ["nginx" "-g"…   0B
# <missing>      3 weeks ago    /bin/sh -c #(nop)  STOPSIGNAL SIGQUIT   0B
# <missing>      3 weeks ago    /bin/sh -c set -x && ...               61.1MB
# <missing>      3 weeks ago    /bin/sh -c #(nop)  FROM debian:bullse…  80.4MB
```

### Layer Caching Strategy (Important!)

```dockerfile
# ❌ BAD — Code changes invalidate npm install cache
FROM node:18-alpine
WORKDIR /app
COPY . .              # Any code change = re-run npm install
RUN npm install
CMD ["node", "app.js"]

# ✅ GOOD — npm install is cached unless package.json changes
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./   # Copy only package files first
RUN npm install         # This layer is cached if package.json unchanged
COPY . .                # Now copy rest of code
CMD ["node", "app.js"]
```

---

## 🔄 Image Lifecycle

```
Docker Hub / Registry
        │
        │ docker pull
        ▼
  Local Image Store
  (/var/lib/docker)
        │
        │ docker run
        ▼
    Container
    (adds a thin writable layer on top)
        │
        │ docker commit (optional)
        ▼
  New Local Image
        │
        │ docker push
        ▼
  Docker Hub / Registry
```

---

## 🏷️ Image Naming & Tags

### Format:
```
[registry/][username/]image-name[:tag]
```

### Examples:
```bash
nginx                          # official image, latest tag
nginx:1.25                     # official image, specific version
nginx:1.25-alpine              # alpine variant
ubuntu:22.04                   # OS image with version
node:18-alpine                 # Node.js on Alpine Linux
python:3.11-slim               # slim Python variant
yourusername/my-app:v1.0       # your Docker Hub image
yourusername/my-app:latest     # your image, latest tag
registry.company.com/api:prod  # private registry image
```

### Tag Best Practices:
```bash
# Never rely on :latest in production — it changes!
# Always pin to specific versions

# Development
docker pull node:18-alpine

# Production — pin exact digest for 100% reproducibility
docker pull node:18-alpine@sha256:abc123def456...
```

---

## 📋 Working with Images

```bash
# List all local images
docker images
# or
docker image ls

# Filter images
docker images nginx
docker images --filter=dangling=true   # untagged images

# Pull specific version
docker pull node:18-alpine

# Remove image
docker rmi nginx:latest

# Remove all dangling (untagged) images
docker image prune

# Remove all unused images (not used by any container)
docker image prune -a

# Get image details
docker inspect nginx

# See layer sizes
docker image history nginx --no-trunc

# Search Docker Hub
docker search --filter stars=50 nginx

# Tag an image (create alias)
docker tag nginx:latest my-nginx:production

# Export image to file (for offline transfer)
docker save nginx:latest -o nginx.tar
gzip nginx.tar

# Import image from file
docker load -i nginx.tar.gz
```

---

## 🌐 Image Variants — Choosing the Right Base

| Variant | Example | Size | Use Case |
|---------|---------|------|----------|
| `full` | `node:18` | ~1GB | Development, debugging |
| `slim` | `node:18-slim` | ~250MB | Production, smaller footprint |
| `alpine` | `node:18-alpine` | ~120MB | Production, smallest size |
| `scratch` | `FROM scratch` | 0B | Go binaries, ultra-minimal |
| `distroless` | `gcr.io/distroless/nodejs18` | ~70MB | Maximum security |

```bash
# Compare image sizes
docker images | grep node

# node:18         latest   ... 1.09GB
# node:18-slim    latest   ... 243MB
# node:18-alpine  latest   ... 127MB
```

> 💡 **DevOps tip:** Start with `alpine` for production. If you hit issues (missing tools, glibc vs musl), switch to `slim`. Only use `full` for development environments.

---

## 🔐 Official vs Community Images

### Official Images (Verified by Docker)
- Single-word names: `nginx`, `node`, `python`, `mysql`, `redis`
- Maintained by Docker or the software vendor
- Regularly updated for security patches
- Documented on Docker Hub

### Verified Publisher Images
- `bitnami/nginx`, `elastic/elasticsearch`
- Verified companies publishing their software

### Community Images
- `username/image-name`
- Anyone can publish — inspect before trusting
- Check: pull count, star rating, last update date

```bash
# Always check before pulling unknown images
docker search redis --format "table {{.Name}}\t{{.Stars}}\t{{.Official}}"
```

---

## 🔍 Image Inspection & Debugging

```bash
# Full metadata of an image
docker inspect python:3.11-slim

# Key fields to look at:
# - Config.Cmd       → default command
# - Config.Env       → environment variables
# - Config.ExposedPorts → exposed ports
# - RootFS.Layers    → number of layers

# Explore image filesystem without running it
docker run --rm -it python:3.11-slim sh
ls /usr/local/lib/python3.11/

# Check image vulnerabilities (requires Docker Scout)
docker scout cves nginx:latest
```

---

## ✅ Things to Remember

- Images are **read-only**; containers add a writable layer on top
- Layer caching = faster builds; understand the order of Dockerfile instructions
- Use specific tags in production (`nginx:1.25.3` not `nginx:latest`)
- Alpine images are smaller but use `musl` libc (not `glibc`) — some packages may behave differently
- `docker image prune -a` is a great disk-space saver on servers

---

**Next:** [📦 Docker Containers →](../05-Containers/containers.md)
