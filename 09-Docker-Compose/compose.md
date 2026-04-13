# 🎼 Docker Compose

> **"Ek container easy hai. Dus containers? Docker Compose ke bina mushkil hai."**

---

## 🤔 What is Docker Compose?

Docker Compose is a tool for **defining and running multi-container Docker applications**. Instead of running 5 separate `docker run` commands with long flags, you define everything in a single `docker-compose.yml` file and bring it all up with one command.

> 🎵 **Real-life analogy:** If each container is a musician, Docker Compose is the **conductor** — it tells everyone when to start, how loud to play, and how to harmonize together.

---

## 🆕 Docker Compose v2 vs v1

```bash
# v1 (deprecated — don't use)
docker-compose up -d     # hyphen, separate binary

# v2 (current — built into Docker CLI)
docker compose up -d     # no hyphen, plugin
```

Always use v2 (`docker compose`, no hyphen).

---

## 📄 docker-compose.yml Structure

```yaml
# Version declaration (optional in v2 but good practice)
# version: "3.9"   # Can omit in newer Docker versions

services:           # Define your containers here
  service-name:
    image: ...      # Use existing image OR
    build: ...      # Build from Dockerfile
    ports: ...      # Port mappings
    volumes: ...    # Mount volumes
    environment: ... # Env variables
    depends_on: ...  # Dependency order
    networks: ...    # Network membership
    restart: ...     # Restart policy
    healthcheck: ... # Health check config

volumes:            # Named volumes
  volume-name:

networks:           # Custom networks
  network-name:
```

---

## 🟢 Simple Example: Node.js + MongoDB

### Project structure:
```
my-app/
├── docker-compose.yml
├── Dockerfile
├── .env
└── src/
    └── index.js
```

### `.env` (never commit to git!):
```env
MONGO_ROOT_USER=admin
MONGO_ROOT_PASS=supersecret
MONGO_DB=myapp
NODE_ENV=production
PORT=3000
```

### `docker-compose.yml`:
```yaml
services:

  # Node.js API
  api:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: my-api
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=${NODE_ENV}
      - PORT=${PORT}
      - MONGO_URL=mongodb://${MONGO_ROOT_USER}:${MONGO_ROOT_PASS}@mongodb:27017/${MONGO_DB}
    depends_on:
      mongodb:
        condition: service_healthy
    networks:
      - app-network
    restart: unless-stopped
    volumes:
      - ./logs:/app/logs

  # MongoDB Database
  mongodb:
    image: mongo:7.0
    container_name: my-mongodb
    environment:
      - MONGO_INITDB_ROOT_USERNAME=${MONGO_ROOT_USER}
      - MONGO_INITDB_ROOT_PASSWORD=${MONGO_ROOT_PASS}
      - MONGO_INITDB_DATABASE=${MONGO_DB}
    volumes:
      - mongo-data:/data/db
      - ./mongo-init.js:/docker-entrypoint-initdb.d/init.js:ro
    networks:
      - app-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

  # Mongo Express (GUI — dev only)
  mongo-express:
    image: mongo-express:1.0
    container_name: mongo-ui
    ports:
      - "8081:8081"
    environment:
      - ME_CONFIG_MONGODB_ADMINUSERNAME=${MONGO_ROOT_USER}
      - ME_CONFIG_MONGODB_ADMINPASSWORD=${MONGO_ROOT_PASS}
      - ME_CONFIG_MONGODB_SERVER=mongodb
    depends_on:
      - mongodb
    networks:
      - app-network
    profiles:
      - dev   # Only starts in dev profile: docker compose --profile dev up

networks:
  app-network:
    driver: bridge

volumes:
  mongo-data:
    driver: local
```

---

## 🔧 docker-compose.yml — Every Option Explained

```yaml
services:
  my-service:

    # --- Image or Build ---
    image: nginx:1.25           # Use pre-built image
    build:                      # OR build from Dockerfile
      context: ./app            # Build context (directory)
      dockerfile: Dockerfile.prod
      args:
        NODE_ENV: production    # Build arguments
      target: production        # Multi-stage build target

    # --- Container settings ---
    container_name: my-app
    hostname: app-server
    restart: unless-stopped     # no | always | unless-stopped | on-failure

    # --- Ports ---
    ports:
      - "3000:3000"             # host:container
      - "127.0.0.1:8080:80"   # bind to specific interface

    # --- Environment ---
    environment:
      - NODE_ENV=production     # list format
      DB_HOST: postgres         # map format
    env_file:
      - .env                    # Load from file
      - .env.production

    # --- Volumes ---
    volumes:
      - named-vol:/data         # named volume
      - ./config:/app/config:ro # bind mount, read-only
      - /tmp:/tmp               # absolute path

    # --- Networks ---
    networks:
      - frontend
      - backend

    # --- Dependencies ---
    depends_on:
      db:
        condition: service_healthy   # wait until healthy
      redis:
        condition: service_started   # just wait until started

    # --- Health check ---
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

    # --- Resource limits ---
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 512M
        reservations:
          memory: 256M

    # --- Logging ---
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

    # --- Extra options ---
    working_dir: /app
    command: ["node", "server.js"]   # override CMD
    entrypoint: ["/entrypoint.sh"]   # override ENTRYPOINT
    user: "1001:1001"
    read_only: true                   # read-only filesystem
    stdin_open: true                  # -i flag
    tty: true                         # -t flag

    # --- Profiles (dev/prod separation) ---
    profiles:
      - dev
```

---

## 🎼 docker-compose.override.yml — Dev/Prod Separation

This is a pattern that senior DevOps engineers use but beginners rarely see.

```yaml
# docker-compose.yml (base — production)
services:
  api:
    image: my-api:latest
    environment:
      - NODE_ENV=production

# docker-compose.override.yml (auto-merged in development!)
# Automatically applied when you run: docker compose up
services:
  api:
    build: .                    # Build locally in dev
    environment:
      - NODE_ENV=development
    volumes:
      - ./src:/app/src          # Live code reload in dev
    command: ["nodemon", "src/index.js"]
```

```bash
# Development (auto-merges override.yml)
docker compose up -d

# Production (only base file)
docker compose -f docker-compose.yml up -d

# Production with separate prod override
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

---

## 💻 Docker Compose Commands

```bash
# Start all services (build if needed)
docker compose up -d

# Start and force rebuild
docker compose up -d --build

# Start specific service only
docker compose up -d api

# Stop all services (containers remain)
docker compose stop

# Stop and remove containers
docker compose down

# Stop, remove containers, AND volumes (destructive!)
docker compose down -v

# Stop, remove containers, images, AND volumes
docker compose down -v --rmi all

# View status
docker compose ps

# Follow logs from all services
docker compose logs -f

# Follow logs from one service
docker compose logs -f api

# Execute command in running service
docker compose exec api bash
docker compose exec api node --version

# Run a one-off command in a new container
docker compose run --rm api node scripts/seed.js

# Scale a service (run multiple instances)
docker compose up -d --scale api=3

# Pull latest images
docker compose pull

# Build images
docker compose build
docker compose build --no-cache api

# Validate compose file
docker compose config
```

---

## 🚀 Full Production Stack: Node.js + PostgreSQL + Redis + Nginx

```yaml
services:

  # Reverse proxy
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
      - static-files:/usr/share/nginx/html:ro
    depends_on:
      - api
    networks:
      - frontend
    restart: unless-stopped

  # Application API
  api:
    build:
      context: .
      target: production
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgresql://app_user:${DB_PASS}@postgres:5432/appdb
      - REDIS_URL=redis://redis:6379
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - frontend
      - backend
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 512M

  # PostgreSQL database
  postgres:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=appdb
      - POSTGRES_USER=app_user
      - POSTGRES_PASSWORD=${DB_PASS}
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    networks:
      - backend
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app_user -d appdb"]
      interval: 10s
      timeout: 5s
      retries: 5
    deploy:
      resources:
        limits:
          memory: 256M

  # Redis cache
  redis:
    image: redis:7-alpine
    command: ["redis-server", "--appendonly", "yes", "--requirepass", "${REDIS_PASS}"]
    volumes:
      - redis-data:/data
    networks:
      - backend
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "--pass", "${REDIS_PASS}", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true    # No internet access for backend

volumes:
  postgres-data:
  redis-data:
  static-files:
```

---

## ✅ Things to Remember

- `docker compose up -d --build` is your daily driver command
- `docker compose down -v` deletes volumes — data is gone!
- `depends_on` doesn't wait for the service to be *ready* — use `healthcheck` + `condition: service_healthy`
- `docker-compose.override.yml` is auto-merged — great for dev/prod separation
- Always use `.env` file for secrets and `env_file:` option to load it
- `docker compose config` validates your compose file before running

---

**Next:** [🛠️ Real-World Projects →](../10-Real-World-Projects/projects.md)
