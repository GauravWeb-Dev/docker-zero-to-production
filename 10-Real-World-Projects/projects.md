# 🛠️ Real-World Docker Projects

> **"Theory padh li? Ab real project banao — tab samajhoge."**

---

## Projects Overview

| # | Project | Stack | Level |
|---|---------|-------|-------|
| 1 | Node.js API Containerization | Node + Express | 🟡 Intermediate |
| 2 | Nginx + MySQL Deployment | Nginx + MySQL | 🟡 Intermediate |
| 3 | 3-Tier App with Docker Compose | React + Node + PG + Redis + Nginx | 🔴 Advanced |

---

## Project 1: Node.js Express API

### `src/index.js`
```javascript
const express = require('express');
const app = express();
const PORT = process.env.PORT || 3000;

app.use(express.json());

app.get('/health', (req, res) => {
    res.json({ status: 'OK', uptime: process.uptime(), env: process.env.NODE_ENV });
});

app.get('/api/users', (req, res) => {
    res.json([{ id: 1, name: 'Alice' }, { id: 2, name: 'Bob' }]);
});

app.listen(PORT, '0.0.0.0', () => console.log(`API on port ${PORT}`));
```

### `Dockerfile` (Multi-stage)
```dockerfile
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .

FROM node:18-alpine AS production
RUN addgroup -g 1001 -S nodejs && adduser -S -u 1001 -G nodejs nodeuser
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force
COPY --from=builder --chown=nodeuser:nodejs /app/src ./src
USER nodeuser
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD wget -qO- http://localhost:3000/health || exit 1
CMD ["node", "src/index.js"]
```

### `.dockerignore`
```
node_modules
.git
.env
*.test.js
coverage/
docker-compose*.yml
Dockerfile*
```

### `docker-compose.yml`
```yaml
services:
  api:
    build:
      context: .
      target: production
    container_name: node-api
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
    restart: unless-stopped
```

### `docker-compose.override.yml` (auto-applied in dev)
```yaml
services:
  api:
    build:
      target: builder
    volumes:
      - ./src:/app/src
    environment:
      - NODE_ENV=development
    command: ["npx", "nodemon", "src/index.js"]
```

### Deploy
```bash
# Dev
docker compose up -d --build

# Production only
docker compose -f docker-compose.yml up -d --build

# Test
curl http://localhost:3000/health
curl http://localhost:3000/api/users
```

---

## Project 2: Nginx + MySQL

### `docker-compose.yml`
```yaml
services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./html:/usr/share/nginx/html:ro
    depends_on:
      - mysql
    networks:
      - web-net
    restart: unless-stopped

  mysql:
    image: mysql:8.0
    environment:
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - MYSQL_DATABASE=${MYSQL_DATABASE}
      - MYSQL_USER=${MYSQL_USER}
      - MYSQL_PASSWORD=${MYSQL_PASSWORD}
    volumes:
      - mysql-data:/var/lib/mysql
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    networks:
      - web-net
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5

  phpmyadmin:
    image: phpmyadmin:latest
    ports:
      - "8080:80"
    environment:
      - PMA_HOST=mysql
    depends_on:
      - mysql
    networks:
      - web-net
    profiles:
      - admin

networks:
  web-net:

volumes:
  mysql-data:
```

### `init.sql`
```sql
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100) UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
INSERT INTO users (name, email) VALUES ('Alice', 'alice@example.com');
```

### Deploy
```bash
# Set env vars
export MYSQL_ROOT_PASSWORD=rootpass MYSQL_DATABASE=mydb MYSQL_USER=user MYSQL_PASSWORD=pass

docker compose up -d
docker compose --profile admin up -d   # with phpMyAdmin

# Access
# Web:        http://localhost
# phpMyAdmin: http://localhost:8080
```

---

## Project 3: 3-Tier Production App

### Architecture
```
Internet → Nginx (80) → [React Static | Node API (3000)]
                                            │
                              [PostgreSQL (5432) | Redis (6379)]
```

### `docker-compose.yml`
```yaml
services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - api
    networks:
      - public
    restart: unless-stopped

  api:
    build:
      context: ./api
      target: production
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgresql://appuser:${DB_PASS}@postgres:5432/appdb
      - REDIS_URL=redis://:${REDIS_PASS}@redis:6379
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - public
      - private
    restart: unless-stopped

  postgres:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=appdb
      - POSTGRES_USER=appuser
      - POSTGRES_PASSWORD=${DB_PASS}
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - private
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser -d appdb"]
      interval: 10s
      retries: 5

  redis:
    image: redis:7-alpine
    command: ["redis-server", "--requirepass", "${REDIS_PASS}"]
    volumes:
      - redis-data:/data
    networks:
      - private
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASS}", "ping"]
      interval: 10s
      retries: 3

networks:
  public:
  private:
    internal: true

volumes:
  postgres-data:
  redis-data:
```

### Nginx config for reverse proxy
```nginx
events { worker_connections 1024; }
http {
  include /etc/nginx/mime.types;
  upstream api { server api:3000; }
  server {
    listen 80;
    location / { root /usr/share/nginx/html; try_files $uri /index.html; }
    location /api {
      proxy_pass http://api;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
    }
  }
}
```

### Deploy
```bash
# Create .env
echo "DB_PASS=securepass\nREDIS_PASS=redispass" > .env

docker compose up -d
docker compose ps      # check all healthy
docker compose logs -f # watch logs

curl http://localhost/api/health
```

---

## DevOps Best Practices Summary

- Always health-check databases before starting the app (`condition: service_healthy`)
- Never hardcode passwords — always use `.env` + `env_file`
- Use `--restart=unless-stopped` for all production services
- Use named volumes for all databases — never bind-mount DB data directories
- Run containers as non-root users
- Use multi-stage Dockerfiles to minimize image size
- Keep frontend and backend on separate networks

---

**Next:** [🎯 Interview Questions →](../11-Interview-Questions/interview.md)
