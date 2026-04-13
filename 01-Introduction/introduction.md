# 📖 Introduction to Docker

> **"Ek baar samajh lo, phir sab easy lagega."**

---

## 🤔 What is Docker?

Docker is an open-source **containerization platform** that allows developers to package applications and their dependencies into lightweight, portable units called **containers**.

Before Docker, deploying an application meant:
- Manually installing runtimes (Node, Python, Java) on every server
- Praying that versions matched between dev, staging, and production
- Writing 20-page "setup documents" that were outdated the moment they were written

Docker solved all of this.

---

## 🏛️ Brief History

| Year | Event |
|------|-------|
| 2008 | Linux introduces **LXC** (Linux Containers) — the foundation |
| 2013 | **Solomon Hykes** releases Docker at PyCon — the world changes |
| 2014 | Docker 1.0 released. Google, Red Hat, IBM join the ecosystem |
| 2015 | Docker Compose introduced for multi-container apps |
| 2017 | Kubernetes becomes the standard orchestrator for Docker containers |
| 2020+ | Docker Desktop, Docker Extensions — mainstream developer tool |

---

## 🖥️ Virtual Machines vs Docker Containers

This is the most important concept to understand first.

```
┌──────────────────────────────────────────────────────────┐
│         VIRTUAL MACHINE (Traditional)                    │
│                                                          │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐                 │
│  │  App A   │ │  App B   │ │  App C   │                 │
│  ├──────────┤ ├──────────┤ ├──────────┤                 │
│  │  Libs    │ │  Libs    │ │  Libs    │                 │
│  ├──────────┤ ├──────────┤ ├──────────┤                 │
│  │ Guest OS │ │ Guest OS │ │ Guest OS │  ← 3 full OSes! │
│  └──────────┘ └──────────┘ └──────────┘                 │
│  ┌─────────────────────────────────────┐                 │
│  │           Hypervisor                │                 │
│  ├─────────────────────────────────────┤                 │
│  │           Host OS                   │                 │
│  ├─────────────────────────────────────┤                 │
│  │           Hardware                  │                 │
│  └─────────────────────────────────────┘                 │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│         DOCKER CONTAINERS (Modern)                       │
│                                                          │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐                 │
│  │  App A   │ │  App B   │ │  App C   │                 │
│  ├──────────┤ ├──────────┤ ├──────────┤                 │
│  │  Libs    │ │  Libs    │ │  Libs    │                 │
│  └──────────┘ └──────────┘ └──────────┘                 │
│  ┌─────────────────────────────────────┐                 │
│  │         Docker Engine               │                 │
│  ├─────────────────────────────────────┤                 │
│  │           Host OS (shared kernel)   │  ← Only 1 OS!  │
│  ├─────────────────────────────────────┤                 │
│  │           Hardware                  │                 │
│  └─────────────────────────────────────┘                 │
└──────────────────────────────────────────────────────────┘
```

### Comparison Table

| Feature | Virtual Machine | Docker Container |
|---------|----------------|-----------------|
| **Boot Time** | 1–5 minutes | Milliseconds |
| **Size** | GBs (includes full OS) | MBs (shares host kernel) |
| **Isolation** | Full (separate kernel) | Process-level |
| **Performance** | ~70% of bare metal | ~95–99% of bare metal |
| **Portability** | Low (hypervisor-dependent) | High (runs anywhere) |
| **Resource Usage** | Heavy | Lightweight |
| **Use Case** | Full OS isolation needed | App packaging & microservices |
| **Startup Cost** | High | Very Low |

> 💡 **Hinglish tip:** VM matlab poori nayi bike khareedna sirf ek pakka uthane ke liye. Container matlab wahi kaam ek auto-rickshaw se karna — faster, cheaper, aur kaam bhi ho jata hai!

---

## 🏗️ Docker Architecture (Detailed)

Docker follows a **client-server architecture**.

### Components:

#### 1. Docker Client (`docker` CLI)
- The tool you type commands into: `docker run`, `docker build`, etc.
- Communicates with Docker Daemon via REST API
- Can connect to remote Docker Daemons too

#### 2. Docker Daemon (`dockerd`)
- The background service that does the actual work
- Manages images, containers, networks, volumes
- Listens on a Unix socket: `/var/run/docker.sock`

#### 3. Docker Engine
- The combination of Docker Client + Docker Daemon
- What you install when you install Docker

#### 4. Docker Registry
- A storage system for Docker images
- **Docker Hub** is the default public registry (like GitHub for code, but for images)
- You can run your own **private registry**
- Cloud options: AWS ECR, Google GCR, Azure ACR

### Workflow: What happens when you run `docker run nginx`?

```
You type:  docker run nginx
                │
                ▼
        Docker Client sends request
        to Docker Daemon via API
                │
                ▼
        Daemon checks: Do I have
        the 'nginx' image locally?
                │
         ┌──────┴──────┐
         │ YES          │ NO
         │              ▼
         │      Pull from Docker Hub
         │      (nginx:latest)
         │              │
         └──────┬───────┘
                ▼
        Create container from image
                │
                ▼
        Start container process
        (nginx web server running!)
```

---

## 🌍 Real-World Use Cases

### 1. Microservices Architecture
Large companies (Netflix, Amazon) break their apps into small services. Each service runs in its own container.

```
User Service Container  →  Auth Service Container
         ↓                          ↓
Payment Service Container  →  DB Container
```

### 2. CI/CD Pipelines
Every commit triggers: build container → run tests in container → deploy container to prod.

### 3. Development Environments
New developer joins the team:
- **Without Docker:** 2 days setting up environment
- **With Docker:** `docker compose up` → ready in 5 minutes

### 4. Legacy App Migration
Old Java app from 2008? Package it in a container. Now it runs on modern infrastructure without touching the code.

### 5. Machine Learning
Data scientists package their ML models + dependencies in containers so they run identically on any GPU server.

---

## 🎯 Things to Remember

- Docker is NOT a VM — it shares the host OS kernel
- Docker containers are **ephemeral** by default (data is lost when container stops — use Volumes to persist)
- Docker Hub is to images what GitHub is to code
- One process per container is the best practice (separation of concerns)
- Docker runs natively on Linux; on Windows/Mac it uses a lightweight Linux VM under the hood

---

## ✅ Summary

```
Docker = Package (image) + Run (container) + Share (registry)

Image      → Blueprint / Template (like a class in OOP)
Container  → Running instance of image (like an object)
Registry   → Storage for images (Docker Hub, ECR, etc.)
```

---

**Next:** [⚙️ Installation Guide →](../02-Installation/installation.md)
