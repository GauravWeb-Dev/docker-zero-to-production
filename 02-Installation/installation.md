# ⚙️ Docker Installation Guide

> **"Pehle install karo, phir duniya conquer karo."**

---

## 🐧 Method 1: Ubuntu / Debian Linux (Recommended)

### Option A: One-Line Install (Fastest)
```bash
curl -fsSL https://get.docker.com | sh
```

### Option B: Step-by-Step Install (Understanding each step)

```bash
# Step 1: Update your system
sudo apt-get update
sudo apt-get upgrade -y

# Step 2: Install prerequisites
sudo apt-get install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

# Step 3: Add Docker's official GPG key
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
    sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Step 4: Set up the Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Step 5: Install Docker Engine
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io \
    docker-buildx-plugin docker-compose-plugin

# Step 6: Start and enable Docker service
sudo systemctl start docker
sudo systemctl enable docker

# Step 7: Add your user to docker group (avoid sudo every time)
sudo usermod -aG docker $USER
newgrp docker

# Step 8: Verify installation
docker --version
docker compose version
docker run hello-world
```

### Expected Output:
```
Docker version 25.0.x, build xxxxxxx
Docker Compose version v2.x.x
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

---

## 🪟 Method 2: Windows Installation

### Prerequisites
- Windows 10/11 (64-bit, Home or Pro)
- WSL2 enabled (Windows Subsystem for Linux)
- Virtualization enabled in BIOS

### Steps:

```powershell
# Step 1: Enable WSL2 (Run PowerShell as Administrator)
wsl --install
# Restart your PC after this

# Step 2: Set WSL2 as default
wsl --set-default-version 2
```

Then:
1. Download **Docker Desktop** from [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop/)
2. Run the installer (`Docker Desktop Installer.exe`)
3. Follow the setup wizard — enable WSL2 integration
4. Start Docker Desktop from the Start Menu
5. Open PowerShell or CMD and verify:

```powershell
docker --version
docker run hello-world
```

> 💡 **Tip:** Docker Desktop on Windows uses WSL2 under the hood — you can run Docker commands from both PowerShell AND your WSL2 Ubuntu terminal.

---

## ☁️ Method 3: AWS EC2 Setup (Production)

This is how DevOps engineers install Docker on cloud servers.

### Launch an EC2 Instance:
- AMI: Ubuntu 22.04 LTS
- Instance Type: t2.micro (free tier) or t3.medium for real workloads
- Security Group: Open port 22 (SSH), 80 (HTTP), 443 (HTTPS), 8080 (custom)

### SSH into EC2 and Install Docker:
```bash
# Connect to your EC2 instance
ssh -i your-key.pem ubuntu@your-ec2-public-ip

# Update system
sudo apt-get update -y

# Install Docker (one line)
curl -fsSL https://get.docker.com | sh

# Add ubuntu user to docker group
sudo usermod -aG docker ubuntu

# Disconnect and reconnect for group change to take effect
exit
ssh -i your-key.pem ubuntu@your-ec2-public-ip

# Verify
docker run hello-world
```

### Install Docker Compose on EC2:
```bash
# Docker Compose is now a plugin (included with Docker Engine)
docker compose version

# If using older standalone version:
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" \
    -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose --version
```

---

## 🍎 Method 4: macOS

1. Download **Docker Desktop for Mac** from [docker.com](https://www.docker.com/products/docker-desktop/)
2. Choose the correct chip: **Intel** or **Apple Silicon (M1/M2/M3)**
3. Drag Docker to Applications
4. Open Docker Desktop from Applications
5. Verify in Terminal:

```bash
docker --version
docker run hello-world
```

---

## 🔍 Post-Installation Checks

```bash
# Check Docker version
docker --version
# → Docker version 25.0.x, build xxxxxxx

# Check Docker Compose version
docker compose version
# → Docker Compose version v2.x.x

# Check Docker daemon status
sudo systemctl status docker
# → Active: active (running)

# Run a test container
docker run hello-world
# → "Hello from Docker!" message

# Check system info
docker info
# Shows: containers, images, storage driver, kernel version, etc.

# Check running containers
docker ps

# See Docker disk usage
docker system df
```

---

## 🔧 Useful Configuration

### Configure Docker to start on boot:
```bash
sudo systemctl enable docker
sudo systemctl enable containerd
```

### Change Docker's default storage location (if disk space is limited):
```bash
# Edit daemon configuration
sudo nano /etc/docker/daemon.json

# Add this content:
{
  "data-root": "/mnt/docker-data"
}

# Restart Docker
sudo systemctl restart docker
```

### Set resource limits (daemon.json):
```json
{
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Hard": 64000,
      "Soft": 64000
    }
  },
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

---

## ❗ Common Installation Issues

| Error | Cause | Fix |
|-------|-------|-----|
| `Got permission denied` | User not in docker group | `sudo usermod -aG docker $USER && newgrp docker` |
| `Cannot connect to Docker daemon` | Docker not started | `sudo systemctl start docker` |
| `docker: command not found` | Installation failed | Re-run installation script |
| `port is already allocated` | Port conflict | Change port mapping or kill conflicting process |

---

## ✅ Things to Remember

- Always add your user to the `docker` group — avoid using `sudo docker` every time
- `newgrp docker` applies group change without logout; logging out/in also works
- On EC2, always open necessary ports in Security Group
- `docker info` is your best friend for debugging installation issues

---

**Next:** [💻 Docker Commands →](../03-Docker-Commands/commands.md)
