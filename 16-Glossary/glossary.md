# 📖 Docker Glossary

> **"Koi word naya lage? Yahan dhundo."**

Alphabetically sorted. Bookmark this page.

---

**Alpine Linux**
A minimal Linux distribution (~5MB) commonly used as a Docker base image. Uses `musl libc` and `apk` package manager. Preferred for production images due to small size and reduced attack surface.

---

**ARG**
A Dockerfile instruction that defines a build-time variable. Only available during `docker build`, not at runtime. Pass with `--build-arg`. Do not use for secrets — they appear in build logs.

---

**Bind Mount**
A type of Docker storage where a specific host filesystem path is mounted into a container. Changes on the host reflect immediately in the container and vice versa. Ideal for development (live code reload).

---

**Bridge Network**
Docker's default network driver. Creates an internal virtual network on the host. Containers on the default bridge communicate via IP only. Custom bridge networks enable DNS resolution by container name.

---

**Build Context**
The set of files sent to the Docker daemon during `docker build`. By default, it's the current directory (`.`). Use `.dockerignore` to exclude unnecessary files and reduce context size.

---

**Capability (Linux)**
A fine-grained unit of root privilege. Docker containers run with a subset of capabilities. Use `--cap-drop ALL` and `--cap-add` only what's needed for security hardening.

---

**cgroups (Control Groups)**
A Linux kernel feature that limits and isolates resource usage (CPU, memory, I/O) per process group. Docker uses cgroups to enforce container resource limits.

---

**CMD**
A Dockerfile instruction that sets the default command to run when a container starts. Overridable at `docker run`. If both `ENTRYPOINT` and `CMD` are set, `CMD` provides default arguments to `ENTRYPOINT`.

---

**Container**
A running instance of a Docker image. It is an isolated process with its own filesystem (from the image), network, and process space. Shares the host OS kernel. Ephemeral by default.

---

**Container Registry**
A storage and distribution system for Docker images. Examples: Docker Hub (public), Amazon ECR, Google GCR, Azure ACR, GitHub Container Registry, and self-hosted registries.

---

**containerd**
The container runtime that Docker uses under the hood. An industry-standard runtime that manages the container lifecycle — pulling images, starting/stopping containers. Also used directly by Kubernetes.

---

**COPY**
A Dockerfile instruction that copies files or directories from the build context into the image filesystem. Preferred over `ADD` for simple file copying.

---

**Dangling Image**
An untagged Docker image (shows as `<none>` in `docker images`). Usually created when you build a new image with the same tag — the old one becomes dangling. Clean with `docker image prune`.

---

**Docker Compose**
A tool for defining and running multi-container applications using a `docker-compose.yml` file. Manages the full lifecycle: start, stop, logs, exec, scale.

---

**Docker Content Trust (DCT)**
A security feature that uses cryptographic signatures to ensure the integrity and publisher of images. Enable with `export DOCKER_CONTENT_TRUST=1`.

---

**Docker Daemon (dockerd)**
The background service that manages Docker objects (images, containers, networks, volumes). Listens on a Unix socket (`/var/run/docker.sock`) and responds to Docker CLI commands via REST API.

---

**Docker Desktop**
A GUI application for Mac and Windows that provides Docker Engine, Docker CLI, Docker Compose, and a visual interface for managing containers and images.

---

**Docker Engine**
The core Docker technology — the combination of Docker Daemon (`dockerd`) and Docker CLI (`docker`). What gets installed when you install Docker.

---

**Docker Hub**
The default public container registry operated by Docker, Inc. Hosts official images (`nginx`, `node`, `python`) and community images. Free public repos, paid private repos.

---

**Docker Scout**
Docker's built-in vulnerability scanning tool. Analyzes images for known CVEs. Run with `docker scout cves image:tag`.

---

**docker-compose.override.yml**
A file that is automatically merged with `docker-compose.yml` when running `docker compose` commands. Used to define development-specific overrides without modifying the base compose file.

---

**Dockerfile**
A plain text file containing step-by-step instructions to build a Docker image. Each instruction creates a layer. Read by `docker build`.

---

**ENTRYPOINT**
A Dockerfile instruction that defines the container's main executable. Unlike `CMD`, it is not easily overridden at runtime (requires `--entrypoint` flag). Use with `CMD` for default arguments.

---

**ENV**
A Dockerfile instruction that sets environment variables available both during build and at container runtime. Can be overridden at runtime with `-e`.

---

**EXPOSE**
A Dockerfile instruction that documents which port(s) the application listens on. Does not actually publish the port — use `-p` at `docker run` for that.

---

**FROM**
The first instruction in any Dockerfile. Specifies the base image to start from. Supports multi-stage builds with multiple `FROM` statements.

---

**Health Check**
A command Docker runs periodically inside a container to determine if it's working correctly. Defined with `HEALTHCHECK` in Dockerfile or `healthcheck:` in Compose. States: `starting`, `healthy`, `unhealthy`.

---

**Host Network**
A Docker network mode where the container shares the host's network stack directly. No network isolation, no NAT. Maximum performance. Linux only (not available on Docker Desktop for Mac/Windows).

---

**Image**
A read-only, layered template used to create containers. Contains the OS, runtime, application code, and dependencies. Stored in a registry.

---

**Image Layer**
A single read-only filesystem change created by one Dockerfile instruction (`RUN`, `COPY`, `ADD`). Layers are stacked and shared between images for efficiency.

---

**Internal Network**
A Docker network created with `--internal` flag. Containers on this network have no internet access — only container-to-container communication. Ideal for database networks.

---

**Layer Cache**
Docker's mechanism to reuse unchanged image layers from previous builds. Dramatically speeds up builds. A cache miss on any layer invalidates all subsequent layers.

---

**Multi-stage Build**
A Dockerfile pattern using multiple `FROM` statements. Allows using a full build environment in early stages and copying only the final artifacts to a minimal final stage, reducing image size significantly.

---

**Namespace**
A Linux kernel feature that provides isolation. Docker uses multiple namespaces: PID (process isolation), Network (network stack), Mount (filesystem), UTS (hostname), IPC (inter-process communication).

---

**Named Volume**
A Docker-managed persistent storage volume with a name. Docker controls where data is stored on the host (`/var/lib/docker/volumes/`). Data persists after container removal.

---

**Overlay Network**
A Docker network driver that spans multiple Docker hosts. Used with Docker Swarm for multi-host container communication.

---

**Overlay Filesystem (OverlayFS)**
The storage driver Docker uses to implement image layers. Efficiently stacks read-only image layers with a writable container layer using Copy-on-Write.

---

**Port Mapping**
Publishing a container's internal port to the host. Syntax: `-p hostPort:containerPort`. E.g., `-p 8080:80` maps host port 8080 to container port 80.

---

**Profile (Compose)**
A Docker Compose feature that lets you group services. Start specific groups with `docker compose --profile dev up`. Useful for separating dev-only tools (like admin UIs) from core services.

---

**Registry**
See *Container Registry*.

---

**Restart Policy**
A container setting that determines when Docker automatically restarts it. Options: `no` (default), `always`, `unless-stopped`, `on-failure[:max-retries]`.

---

**RUN**
A Dockerfile instruction that executes a command during image build and creates a new layer. Commonly used for installing packages, setting up directories, running build scripts.

---

**Tag**
A label applied to a Docker image to identify a specific version. Format: `image:tag`. If omitted, Docker defaults to `:latest`. Use specific tags in production.

---

**tmpfs Mount**
A Docker mount type that stores data in host memory (RAM). Data is lost when the container stops. Used for sensitive data that shouldn't touch disk.

---

**USER**
A Dockerfile instruction that sets the user (and optionally group) for subsequent `RUN`, `CMD`, and `ENTRYPOINT` instructions. Best practice: always set a non-root user.

---

**Volume**
Persistent storage in Docker. Data survives container removal. Two main types: Named Volumes (Docker-managed) and Bind Mounts (host path).

---

**WORKDIR**
A Dockerfile instruction that sets the working directory for subsequent `RUN`, `COPY`, `ADD`, `CMD`, and `ENTRYPOINT` instructions. Creates the directory if it doesn't exist.

---

*Back to [README →](../README.md)*
