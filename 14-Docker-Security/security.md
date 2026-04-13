# 🔐 Docker Security

> **"Secure container banana utna hi zaroori hai jitna banaa."**

---

## Why Docker Security Matters

Containers share the host OS kernel. A misconfigured container can:
- Escape to the host system
- Access other containers' data
- Expose secrets to attackers
- Be exploited via vulnerable base images

Security is not optional in production.

---

## 1️⃣ Never Run Containers as Root

By default, containers run as root inside the container. If an attacker escapes the container, they're root on the host too.

```dockerfile
# ❌ Bad — runs as root
FROM node:18-alpine
WORKDIR /app
COPY . .
CMD ["node", "app.js"]

# ✅ Good — non-root user
FROM node:18-alpine
WORKDIR /app

# Create dedicated user
RUN addgroup -g 1001 -S appgroup && \
    adduser -S -u 1001 -G appgroup appuser

COPY --chown=appuser:appgroup . .
USER appuser
CMD ["node", "app.js"]
```

```bash
# Verify container isn't running as root
docker exec my-container whoami
# → appuser  ✅ (not root)

# Run with explicit user at runtime
docker run --user 1001:1001 my-app
```

---

## 2️⃣ Use Minimal Base Images

Every package in your image is a potential attack surface.

```dockerfile
# ❌ Full OS — hundreds of unnecessary packages
FROM ubuntu:22.04

# ✅ Alpine — minimal attack surface
FROM node:18-alpine

# ✅✅ Distroless — no shell, no package manager, maximum security
FROM gcr.io/distroless/nodejs18-debian11
```

| Base Image | Size | Shell? | Attack Surface |
|-----------|------|--------|----------------|
| `ubuntu:22.04` | ~80MB | ✅ bash | High |
| `node:18-slim` | ~240MB | ✅ bash | Medium |
| `node:18-alpine` | ~127MB | ✅ sh | Low |
| `distroless/nodejs18` | ~70MB | ❌ none | Minimal |

---

## 3️⃣ Never Bake Secrets into Images

Secrets in images are visible to anyone with access to the image.

```dockerfile
# ❌ NEVER — visible in docker history and image layers
ENV DB_PASSWORD=supersecret
ARG API_KEY=abc123

# ❌ NEVER — even if you delete it in next layer, it's in the layer cache
RUN echo "password=secret" > /app/config
RUN rm /app/config   # too late — it's in the previous layer!
```

**Safe approaches:**

```bash
# ✅ Runtime environment variables
docker run -e DB_PASSWORD=secret my-app

# ✅ Env file (not committed to git)
docker run --env-file .env my-app
echo ".env" >> .gitignore

# ✅ Docker Secrets (Swarm mode)
echo "supersecret" | docker secret create db_password -
docker service create \
  --secret db_password \
  --name my-service \
  my-app

# Inside container: secret is at /run/secrets/db_password
```

---

## 4️⃣ Read-Only Filesystems

Prevent malicious code from writing to the container filesystem.

```bash
# Run with read-only root filesystem
docker run --read-only my-app

# Allow writes only to specific directories (tmpfs)
docker run \
  --read-only \
  --tmpfs /tmp \
  --tmpfs /app/logs \
  my-app
```

```yaml
# In docker-compose.yml
services:
  api:
    image: my-api
    read_only: true
    tmpfs:
      - /tmp
      - /app/logs
```

---

## 5️⃣ Limit Container Capabilities

Linux capabilities are fine-grained privileges. Drop all and add only what's needed.

```bash
# Drop ALL capabilities, add only what's needed
docker run \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \   # allow binding to ports < 1024
  my-app

# Common capabilities to drop
docker run \
  --cap-drop NET_ADMIN \
  --cap-drop SYS_ADMIN \
  --cap-drop SETUID \
  --cap-drop SETGID \
  my-app
```

```yaml
# In docker-compose.yml
services:
  api:
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
```

---

## 6️⃣ Scan Images for Vulnerabilities

```bash
# Docker Scout (built into Docker)
docker scout cves nginx:latest
docker scout quickview nginx:latest

# Trivy (popular open-source scanner)
# Install: https://trivy.dev
trivy image nginx:latest
trivy image --severity HIGH,CRITICAL my-app:latest

# Snyk
snyk container test nginx:latest

# In CI/CD pipeline (GitHub Actions)
# - name: Scan image
#   uses: aquasecurity/trivy-action@master
#   with:
#     image-ref: my-app:latest
#     severity: HIGH,CRITICAL
#     exit-code: 1   # fail pipeline on findings
```

---

## 7️⃣ Network Security

```bash
# 1. Don't expose unnecessary ports
docker run -p 5432:5432 postgres  # ❌ DB exposed to internet
docker run postgres                # ✅ only accessible from Docker network

# 2. Use internal networks for backend services
docker network create --internal backend-net

# 3. Bind to localhost only (not 0.0.0.0) for development
docker run -p 127.0.0.1:5432:5432 postgres
```

```yaml
# docker-compose.yml — secure network design
services:
  nginx:
    ports:
      - "80:80"         # only nginx exposed
    networks: [public]

  api:
    networks: [public, private]   # talks to both nginx and db

  db:
    networks: [private]   # not exposed at all
    # NO ports: section

networks:
  public:
  private:
    internal: true        # no internet access
```

---

## 8️⃣ Secure the Docker Daemon

```bash
# 1. Never expose Docker socket to containers
docker run -v /var/run/docker.sock:/var/run/docker.sock untrusted-image  # ❌ DANGEROUS

# 2. Use TLS for remote Docker daemon access
dockerd --tls --tlscert /path/to/cert.pem --tlskey /path/to/key.pem

# 3. Enable Docker Content Trust (image signing)
export DOCKER_CONTENT_TRUST=1
docker pull nginx   # only pulls if image is signed

# 4. Use daemon.json to restrict options
cat /etc/docker/daemon.json
{
  "userns-remap": "default",        # enable user namespace remapping
  "no-new-privileges": true,        # prevent privilege escalation
  "live-restore": true,
  "log-driver": "json-file",
  "log-opts": { "max-size": "10m" }
}
```

---

## 9️⃣ Security Checklist

```
Container Security Checklist
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[ ] Running as non-root user (USER instruction in Dockerfile)
[ ] Using minimal base image (Alpine or Distroless)
[ ] No secrets in image (use runtime env vars or Docker Secrets)
[ ] Read-only filesystem where possible
[ ] Capabilities dropped (--cap-drop ALL)
[ ] Image scanned for CVEs (Trivy, Scout)
[ ] No unnecessary ports exposed
[ ] Backend services on internal network
[ ] Docker socket not mounted in containers
[ ] Resource limits set (--memory, --cpus)
[ ] HEALTHCHECK defined
[ ] .dockerignore excludes sensitive files
[ ] Specific image tags (not :latest)
[ ] Regular base image updates
```

---

## 🔟 Security in docker-compose.yml (Production Template)

```yaml
services:
  api:
    build:
      context: .
      target: production
    user: "1001:1001"
    read_only: true
    tmpfs:
      - /tmp
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
    security_opt:
      - no-new-privileges:true
    environment:
      - NODE_ENV=production
    env_file:
      - .env
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: "1.0"
    networks:
      - public
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:3000/health"]
      interval: 30s
      timeout: 5s
      retries: 3
```

---

## ✅ Things to Remember

- Never run as root in production containers
- Never put secrets in Dockerfiles or images
- Scan images regularly — base images get new CVEs constantly
- Expose only the minimum ports necessary
- Use internal networks for databases and backend services
- Drop all Linux capabilities and add back only what's needed

---

**Next:** [🚀 CI/CD Integration →](../15-CI-CD-Integration/cicd.md)
