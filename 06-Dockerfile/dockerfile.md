# 📝 Dockerfile — Complete Guide

> **"Dockerfile = apni app ki recipe. Jo likhoge, wahi container banega."**

---

## 🤔 What is a Dockerfile?

A Dockerfile is a **plain text file** containing a sequence of instructions that Docker uses to automatically build an image. It's the blueprint for your container.

```
Dockerfile  ──[docker build]──►  Image  ──[docker run]──►  Container
```

---

## 📖 All Dockerfile Instructions

### `FROM` — Base Image
Every Dockerfile must start with `FROM`. It defines the starting point.

```dockerfile
# Official image
FROM ubuntu:22.04

# Minimal Alpine
FROM alpine:3.18

# Specific Node.js version
FROM node:18-alpine

# Multi-stage build (explained later)
FROM node:18-alpine AS builder

# Start from absolute scratch (for Go binaries)
FROM scratch
```

---

### `WORKDIR` — Set Working Directory

```dockerfile
# Set the working directory for subsequent instructions
WORKDIR /app

# Creates directory if it doesn't exist
# All RUN, COPY, CMD etc. will execute from /app
WORKDIR /usr/src/app
```

---

### `COPY` vs `ADD`

```dockerfile
# COPY — Simple, explicit (preferred)
COPY package.json .
COPY src/ ./src/
COPY . .

# COPY with --chown (set file ownership)
COPY --chown=node:node . .

# ADD — Like COPY but can also:
# 1. Extract tar files automatically
# 2. Download from URLs (avoid this — use curl/wget in RUN instead)
ADD app.tar.gz /app/       # auto-extracts
ADD https://example.com/file.txt /tmp/  # downloads (bad practice)

# Best Practice: Use COPY unless you specifically need ADD's features
```

---

### `RUN` — Execute Commands

```dockerfile
# Shell form (runs as /bin/sh -c "command")
RUN apt-get update

# Exec form (no shell — faster, safer)
RUN ["apt-get", "update"]

# Chain commands with && to reduce layers
RUN apt-get update && \
    apt-get install -y curl wget git && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Node.js dependencies
RUN npm ci --only=production

# Python dependencies
RUN pip install --no-cache-dir -r requirements.txt
```

---

### `CMD` vs `ENTRYPOINT`

This is the most confusing part — let's clear it up.

```
ENTRYPOINT = What the container IS
CMD        = Default arguments for ENTRYPOINT (can be overridden at runtime)
```

```dockerfile
# CMD — Default command (overridable)
CMD ["node", "server.js"]
CMD ["nginx", "-g", "daemon off;"]
CMD ["/bin/sh"]

# Override CMD at runtime:
# docker run my-app node other-script.js

# ENTRYPOINT — Container's main purpose (not overridable without --entrypoint)
ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]   # default args to nginx

# At runtime: docker run nginx-img -c /etc/nginx/custom.conf
# → Runs: nginx -c /etc/nginx/custom.conf

# Shell form (not recommended in production)
CMD node server.js     # starts as /bin/sh -c "node server.js"
ENTRYPOINT node server.js   # PID 1 issues, signal handling problems
```

| | CMD | ENTRYPOINT |
|---|-----|------------|
| **Purpose** | Default command | Defines the executable |
| **Overridable?** | Yes (`docker run img custom-cmd`) | Only with `--entrypoint` flag |
| **Recommended form** | Exec form `["cmd", "arg"]` | Exec form `["executable"]` |

---

### `ENV` — Environment Variables

```dockerfile
# Set single variable
ENV NODE_ENV=production

# Set multiple variables
ENV NODE_ENV=production \
    PORT=3000 \
    APP_VERSION=1.0.0

# Can be overridden at runtime
# docker run -e NODE_ENV=development my-app
```

---

### `ARG` — Build-Time Arguments

```dockerfile
# ARG is only available during build, not in the running container
ARG NODE_VERSION=18
FROM node:${NODE_VERSION}-alpine

ARG BUILD_ENV=production
RUN echo "Building for: $BUILD_ENV"

# Pass at build time:
# docker build --build-arg NODE_VERSION=20 --build-arg BUILD_ENV=staging .
```

| | ARG | ENV |
|---|-----|-----|
| **Scope** | Build time only | Build + Runtime |
| **Override** | `--build-arg` flag | `-e` flag at runtime |
| **Visible in container?** | No | Yes |

---

### `EXPOSE` — Document Ports

```dockerfile
# EXPOSE documents which port the app listens on
# Does NOT actually publish the port!
EXPOSE 3000
EXPOSE 80
EXPOSE 443
EXPOSE 5432/tcp

# To actually publish, use -p at runtime:
# docker run -p 8080:3000 my-app
```

---

### `VOLUME` — Mount Points

```dockerfile
# Declare that /data should be externally mountable
VOLUME ["/data"]
VOLUME /var/log/myapp

# Creates anonymous volume automatically if not mounted
# Best to use named volumes at runtime instead:
# docker run -v mydata:/data my-app
```

---

### `USER` — Set Running User

```dockerfile
# Never run as root in production!
RUN useradd --uid 1001 --create-home appuser
USER appuser

# OR use existing users
USER node      # node:alpine images have 'node' user built in
USER nginx     # nginx image has 'nginx' user
```

---

### `HEALTHCHECK` — Container Health

```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:3000/health || exit 1

# Disable inherited health check
HEALTHCHECK NONE
```

---

### `LABEL` — Metadata

```dockerfile
LABEL maintainer="yourname@email.com"
LABEL version="1.0.0"
LABEL description="My awesome Node.js API"
LABEL org.opencontainers.image.source="https://github.com/user/repo"
```

---

### `ONBUILD` — Trigger for Child Images

```dockerfile
# Triggers when someone uses this as a base image
ONBUILD COPY package*.json ./
ONBUILD RUN npm install
# Used mainly for base/builder images
```

---

## 🟢 Complete Real-World Example: Node.js API

This is the **Express REST API** we'll use throughout this repo.

### Project Structure:
```
my-api/
├── Dockerfile
├── .dockerignore
├── package.json
├── package-lock.json
└── src/
    └── index.js
```

### `src/index.js`:
```javascript
const express = require('express');
const app = express();
const PORT = process.env.PORT || 3000;

app.use(express.json());

app.get('/health', (req, res) => {
    res.json({ status: 'OK', uptime: process.uptime() });
});

app.get('/api/users', (req, res) => {
    res.json([
        { id: 1, name: 'Alice' },
        { id: 2, name: 'Bob' }
    ]);
});

app.listen(PORT, () => {
    console.log(`API running on port ${PORT}`);
});
```

### `package.json`:
```json
{
  "name": "my-api",
  "version": "1.0.0",
  "main": "src/index.js",
  "scripts": { "start": "node src/index.js" },
  "dependencies": { "express": "^4.18.2" }
}
```

### `Dockerfile` (Production-Ready):
```dockerfile
# ── Stage 1: Builder ──────────────────────────────
FROM node:18-alpine AS builder

# Set working directory
WORKDIR /app

# Copy dependency files first (layer cache optimization)
COPY package*.json ./

# Install all dependencies (including devDependencies for build step)
RUN npm ci

# Copy source code
COPY . .

# ── Stage 2: Production ───────────────────────────
FROM node:18-alpine AS production

# Add labels
LABEL maintainer="devops@company.com"
LABEL version="1.0.0"

# Create non-root user for security
RUN addgroup -g 1001 -S nodejs && \
    adduser -S -u 1001 -G nodejs nodeuser

# Set working directory
WORKDIR /app

# Copy only production dependencies
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

# Copy built application from builder stage
COPY --from=builder --chown=nodeuser:nodejs /app/src ./src

# Switch to non-root user
USER nodeuser

# Document port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD wget -qO- http://localhost:3000/health || exit 1

# Start the app
CMD ["node", "src/index.js"]
```

### `.dockerignore`:
```
# Node modules (reinstalled inside container)
node_modules
npm-debug.log
yarn-error.log

# Version control
.git
.gitignore

# Development files
.env
.env.local
.env.*.local

# Documentation
README.md
docs/

# Tests
__tests__/
*.test.js
*.spec.js

# Build artifacts
dist/
build/
coverage/

# Docker files (not needed inside container)
Dockerfile*
docker-compose*
.dockerignore

# OS files
.DS_Store
Thumbs.db
```

### Build and Run:
```bash
docker build -t my-api:v1 .
docker run -d -p 3000:3000 --name my-api my-api:v1

# Test
curl http://localhost:3000/health
curl http://localhost:3000/api/users
```

---

## 🏗️ Multi-Stage Builds — The Power Feature

Multi-stage builds let you use multiple `FROM` statements. Each stage can use a different base image, and you copy only what you need to the final stage.

### Go Application Example (smallest possible image):
```dockerfile
# Build stage — full Go environment
FROM golang:1.21-alpine AS builder
WORKDIR /build
COPY go.* .
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o app .

# Final stage — absolute minimum
FROM scratch
COPY --from=builder /build/app /app
EXPOSE 8080
ENTRYPOINT ["/app"]

# Result: ~10MB image vs ~300MB with full Go environment!
```

### React Frontend Example:
```dockerfile
# Build stage — install deps and build
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build   # creates /app/dist

# Final stage — just serve the static files
FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]

# Result: ~25MB vs ~500MB+ full Node image
```

---

## ✅ Dockerfile Best Practices

```dockerfile
# 1. Use specific versions, not :latest
FROM node:18.19.0-alpine3.18   # ✅
FROM node:latest                # ❌

# 2. Minimize layers — chain RUN commands
RUN apt-get update && \
    apt-get install -y curl && \
    rm -rf /var/lib/apt/lists/*

# 3. Order instructions by frequency of change (least → most)
FROM node:18-alpine          # rarely changes
WORKDIR /app                 # rarely changes
COPY package*.json ./        # changes when deps change
RUN npm ci                   # changes when deps change
COPY . .                     # changes with every code commit
CMD ["node", "app.js"]

# 4. Always use .dockerignore

# 5. Never store secrets in Dockerfile
ENV DB_PASSWORD=secret123   # ❌ visible in docker history!
# Use docker secrets or env files at runtime instead

# 6. Use non-root user
USER node   # ✅

# 7. Use COPY not ADD (unless extracting tar)
COPY . .   # ✅
ADD . .    # ❌ (unless needed)

# 8. Use multi-stage builds for compiled languages

# 9. Add HEALTHCHECK for production images

# 10. Set WORKDIR explicitly (don't cd in RUN)
WORKDIR /app   # ✅
RUN cd /app    # ❌
```

---

## ✅ Things to Remember

- Dockerfile = recipe, Image = product, Container = running instance
- Every `RUN`, `COPY`, `ADD` creates a new layer
- `.dockerignore` is as important as `.gitignore` — always include it
- CMD can be overridden; ENTRYPOINT defines the container's purpose
- Multi-stage builds dramatically reduce image size
- Never bake secrets into images — use runtime env vars

---

**Next:** [💾 Docker Volumes →](../07-Docker-Volumes/volumes.md)
