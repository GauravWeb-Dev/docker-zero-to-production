# 🌐 Docker Networking

> **"Containers ko baat karni hai? Network banao. Akele rehna hai? Isolate karo."**

---

## 🤔 Why Docker Networking?

Containers are isolated by default. For them to communicate — with each other, with the host, or with the outside world — Docker provides multiple networking options.

> 📞 **Real-life analogy:** Imagine each container is a person in a separate room. By default, they can't hear each other. Networks are the phone lines you install to let them talk.

---

## 🌐 Network Types Overview

```
┌──────────────────────────────────────────────────────────────┐
│                   DOCKER NETWORK TYPES                       │
│                                                              │
│  ┌─────────────┐  ┌────────────┐  ┌─────────────────────┐   │
│  │   bridge    │  │    host    │  │  none / custom      │   │
│  │ (default)   │  │            │  │                     │   │
│  │             │  │ Container  │  │ none = no network   │   │
│  │ Internal    │  │ shares     │  │                     │   │
│  │ virtual     │  │ host's     │  │ custom = user-      │   │
│  │ network     │  │ network    │  │ defined bridge      │   │
│  │             │  │ stack      │  │ (recommended)       │   │
│  └─────────────┘  └────────────┘  └─────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

---

## 1️⃣ Bridge Network (Default)

When you run a container without specifying a network, it connects to the default `bridge` network.

```bash
# See default networks
docker network ls
# NETWORK ID   NAME      DRIVER    SCOPE
# abc123def    bridge    bridge    local
# def456ghi    host      host      local
# ghi789jkl    none      null      local

# Containers on bridge can communicate via IP only (not by name)
docker run -d --name container1 nginx
docker run -d --name container2 nginx

docker inspect container1 | grep IPAddress
# "IPAddress": "172.17.0.2"

# container2 can reach container1 via IP:
docker exec container2 curl 172.17.0.2
# But NOT by name: curl container1 ❌
```

**Limitations of default bridge:**
- Containers can only communicate by IP (no DNS name resolution)
- All containers on bridge can talk to each other — no isolation

---

## 2️⃣ Custom Bridge Network (Recommended)

Create your own bridge network. Containers get **automatic DNS resolution** — they can reach each other by name!

```bash
# Create a custom bridge network
docker network create my-network
docker network create --driver bridge my-network   # explicit driver

# Create with subnet and gateway
docker network create \
  --driver bridge \
  --subnet=192.168.100.0/24 \
  --gateway=192.168.100.1 \
  production-net

# Run containers on the custom network
docker run -d --name api-server --network my-network node-api
docker run -d --name db --network my-network postgres

# Now api-server can reach db by name!
docker exec api-server curl http://db:5432   # works! ✅
docker exec api-server ping db               # works! ✅

# Connect an existing container to a network
docker network connect my-network existing-container

# Disconnect
docker network disconnect my-network container-name
```

### Custom Network vs Default Bridge:

| Feature | Default Bridge | Custom Bridge |
|---------|---------------|---------------|
| **DNS resolution** | ❌ No (IP only) | ✅ Yes (by name) |
| **Isolation** | All containers share it | Scoped to network |
| **Recommended?** | Development only | Production |
| **Container comms** | Via IP | Via name or IP |

---

## 3️⃣ Host Network

Container shares the host's network stack directly — no isolation, no NAT.

```bash
docker run -d --network host nginx
# Nginx is now listening on host's port 80 directly
# No -p port mapping needed (or possible)

# Check: host port 80 is occupied by nginx
curl localhost:80
```

**When to use:**
- Maximum network performance (no NAT overhead)
- When you need to bind to specific host interfaces
- Monitoring tools that need full host network access

**Caution:**
- No network isolation — container can see all host ports
- Linux only (not available on Docker Desktop for Mac/Windows)

---

## 4️⃣ None Network

Completely isolated — no network interface except loopback.

```bash
docker run -d --network none --name isolated alpine sleep 1000

# Container has no external connectivity
docker exec isolated ping google.com
# ping: bad address 'google.com' — no network!
```

**Use case:** Security-sensitive workloads that must have zero network access.

---

## 5️⃣ Overlay Network (Swarm / Multi-host)

Spans multiple Docker hosts. Used with Docker Swarm.

```bash
# Initialize swarm
docker swarm init

# Create overlay network
docker network create --driver overlay my-overlay

# Services on this network can communicate across hosts
docker service create --name web --network my-overlay nginx
```

---

## 🔍 Network Commands

```bash
# List all networks
docker network ls

# Inspect a network (see containers, subnet, etc.)
docker network inspect bridge
docker network inspect my-network

# Create network
docker network create my-net
docker network create --driver bridge \
                       --subnet=10.10.0.0/16 \
                       --ip-range=10.10.1.0/24 \
                       my-net

# Connect container to network
docker network connect my-net container-name

# Disconnect container from network
docker network disconnect my-net container-name

# Remove network
docker network rm my-net

# Remove all unused networks
docker network prune
```

---

## 🔗 Container-to-Container Communication

### Scenario: API Server talking to Database

```bash
# Create isolated network for the app
docker network create app-network

# Start the database
docker run -d \
  --name postgres-db \
  --network app-network \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=myapp \
  postgres:15

# Start the API server (can reach postgres-db by name)
docker run -d \
  --name api-server \
  --network app-network \
  -e DATABASE_URL=postgresql://postgres:secret@postgres-db:5432/myapp \
  -p 3000:3000 \
  my-api:latest

# Test: API can reach DB
docker exec api-server ping postgres-db
docker exec api-server psql postgresql://postgres:secret@postgres-db:5432/myapp -c "\l"
```

---

## 🌍 Exposing Containers to the Outside World

```bash
# Map container port to host port
docker run -d -p 80:80 nginx        # host:80 → container:80
docker run -d -p 8080:80 nginx      # host:8080 → container:80
docker run -d -p 443:443 nginx      # HTTPS

# Map to specific host interface
docker run -d -p 127.0.0.1:8080:80 nginx   # only localhost
docker run -d -p 0.0.0.0:8080:80 nginx     # all interfaces

# Map all exposed ports automatically
docker run -d -P nginx   # capital P = auto port assignment

# UDP port
docker run -d -p 5353:5353/udp dns-server

# Multiple ports
docker run -d \
  -p 80:80 \
  -p 443:443 \
  nginx
```

---

## 🔐 Network Security Best Practices

```bash
# 1. Create separate networks per application stack
docker network create frontend-net    # web → nginx
docker network create backend-net     # nginx → api → db

# 2. Only connect containers to networks they need
docker run -d --name nginx --network frontend-net nginx
docker run -d --name api \
  --network frontend-net \
  --network backend-net \
  my-api   # nginx can reach it AND it can reach db

docker run -d --name db \
  --network backend-net \   # only backend, not exposed to frontend
  postgres

# 3. Use internal networks (no outside internet access)
docker network create --internal private-net

# 4. Never expose DB ports to the public
docker run -d -p 5432:5432 postgres  # ❌ DB exposed to internet!
docker run -d postgres               # ✅ Only accessible from app network
```

---

## 🐳 Docker Compose Networking (Preview)

Docker Compose automatically creates a network per project:

```yaml
services:
  api:
    build: .
    networks:
      - backend

  db:
    image: postgres
    networks:
      - backend

  nginx:
    image: nginx
    ports:
      - "80:80"
    networks:
      - frontend
      - backend

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true   # no internet access
```

See full Compose documentation in the [Compose section](../09-Docker-Compose/compose.md).

---

## ✅ Things to Remember

- Default bridge network = no DNS (containers talk via IP only)
- Custom bridge network = automatic DNS (containers talk by name) — always use this
- `--network host` = no isolation, maximum performance, Linux only
- Never expose database ports (3306, 5432, 27017) to the public
- Use separate networks for frontend, backend, and database tiers
- Docker Compose automatically creates a network for your stack

---

**Next:** [🎼 Docker Compose →](../09-Docker-Compose/compose.md)
