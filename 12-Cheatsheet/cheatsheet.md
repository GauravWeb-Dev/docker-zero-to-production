# 📋 Docker Cheatsheet — Quick Revision

> **"Exam se pehle yahi dekho. Interview se pehle bhi yahi dekho."**

---

## 🚀 Container Lifecycle

```bash
docker run -d -p 8080:80 --name web nginx     # create + start
docker start web                               # start stopped
docker stop web                                # graceful stop
docker restart web                             # restart
docker kill web                                # force stop
docker rm web                                  # remove stopped
docker rm -f web                               # force remove running
docker pause web / docker unpause web          # pause/resume
```

## 📋 Inspection & Monitoring

```bash
docker ps                                      # running containers
docker ps -a                                   # all containers
docker ps -q                                   # IDs only
docker logs -f --tail=100 web                  # follow logs
docker stats                                   # live resource usage
docker top web                                 # processes in container
docker inspect web                             # full JSON details
docker port web                                # port mappings
docker diff web                                # filesystem changes
```

## 🐚 Container Interaction

```bash
docker exec -it web bash                       # shell in container
docker exec -it web sh                         # sh (alpine)
docker exec web nginx -t                       # run command
docker exec -it --user root web bash           # as root
docker cp file.txt web:/app/                   # copy to container
docker cp web:/app/log.txt ./                  # copy from container
docker attach web                              # attach (careful!)
```

## 🖼️ Images

```bash
docker images                                  # list images
docker pull nginx:1.25                         # pull image
docker push user/app:v1                        # push to registry
docker rmi nginx                               # remove image
docker image prune                             # remove dangling
docker image prune -a                          # remove all unused
docker tag my-app user/my-app:v1              # tag image
docker save -o app.tar my-app                  # export to file
docker load -i app.tar                         # import from file
docker image history nginx                     # show layers
docker inspect nginx                           # image details
docker search nginx                            # search hub
```

## 🔨 Build

```bash
docker build -t app:v1 .                       # build from Dockerfile
docker build -f Dockerfile.prod -t app:prod .  # custom Dockerfile
docker build --no-cache -t app .               # bypass cache
docker build --build-arg ENV=prod -t app .     # build args
docker build --target production -t app .      # multi-stage target
docker buildx build --platform linux/amd64,linux/arm64 \
  -t user/app:latest --push .                  # multi-arch
```

## 🌐 Networking

```bash
docker network ls                              # list networks
docker network create my-net                   # create network
docker network inspect my-net                  # inspect
docker network connect my-net web              # connect container
docker network disconnect my-net web           # disconnect
docker network rm my-net                       # remove
docker network prune                           # remove unused
```

## 💾 Volumes

```bash
docker volume ls                               # list volumes
docker volume create my-vol                    # create volume
docker volume inspect my-vol                   # inspect
docker volume rm my-vol                        # remove
docker volume prune                            # remove unused
```

## 🎼 Docker Compose

```bash
docker compose up -d                           # start all (detached)
docker compose up -d --build                   # start + rebuild
docker compose down                            # stop + remove containers
docker compose down -v                         # also remove volumes
docker compose ps                              # status
docker compose logs -f                         # all logs
docker compose logs -f api                     # service logs
docker compose exec api bash                   # shell in service
docker compose run --rm api node seed.js       # one-off command
docker compose pull                            # pull latest images
docker compose build --no-cache                # rebuild images
docker compose config                          # validate compose file
docker compose --profile dev up -d             # start with profile
docker compose up -d --scale api=3             # scale service
```

## 🧹 Cleanup

```bash
docker container prune                         # remove stopped containers
docker image prune                             # remove dangling images
docker image prune -a                          # remove all unused images
docker volume prune                            # remove unused volumes
docker network prune                           # remove unused networks
docker system prune                            # remove all unused resources
docker system prune -a -f --volumes            # nuclear cleanup
docker system df                               # disk usage summary
```

## 🔐 Registry

```bash
docker login                                   # login to Docker Hub
docker login registry.example.com             # login to private registry
docker logout                                  # logout
docker tag my-app user/my-app:v1.0            # tag for push
docker push user/my-app:v1.0                  # push to Hub
docker pull user/my-app:v1.0                  # pull from Hub
```

## 🔍 Useful One-Liners

```bash
# Get IP of container
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' container

# Stop all running containers
docker stop $(docker ps -q)

# Remove all stopped containers
docker rm $(docker ps -aq -f status=exited)

# Remove all untagged images
docker rmi $(docker images -f "dangling=true" -q)

# Watch container resource usage
watch -n 1 docker stats --no-stream

# Get env vars of running container
docker inspect --format='{{range .Config.Env}}{{println .}}{{end}}' container

# Check container health
docker inspect --format='{{.State.Health.Status}}' container

# Copy entire directory
docker cp container:/app/logs ./local-logs/

# Execute SQL in postgres container
docker exec -it postgres psql -U postgres -d mydb -c "SELECT * FROM users;"

# Tail logs since 1 hour ago
docker logs --since=1h -f my-app
```

## ⚡ `docker run` Flags Quick Reference

| Flag | Meaning | Example |
|------|---------|---------|
| `-d` | Detached (background) | `docker run -d nginx` |
| `-it` | Interactive + TTY | `docker run -it ubuntu bash` |
| `-p` | Port mapping | `-p 8080:80` |
| `-v` | Volume mount | `-v data:/data` |
| `-e` | Environment variable | `-e KEY=value` |
| `--name` | Container name | `--name web` |
| `--rm` | Remove on exit | `docker run --rm alpine` |
| `--network` | Network | `--network my-net` |
| `--restart` | Restart policy | `--restart=unless-stopped` |
| `--memory` | Memory limit | `--memory=512m` |
| `--cpus` | CPU limit | `--cpus=1.0` |
| `--user` | Run as user | `--user 1001:1001` |
| `-w` | Working directory | `-w /app` |
| `--env-file` | Load env from file | `--env-file .env` |
| `-P` | Publish all ports | `docker run -P nginx` |

## 📝 Dockerfile Quick Reference

```dockerfile
FROM node:18-alpine          # base image
WORKDIR /app                 # working directory
COPY package*.json ./        # copy files
RUN npm ci                   # run command (creates layer)
COPY . .                     # copy rest of code
ENV NODE_ENV=production      # environment variable
ARG BUILD_ENV=prod           # build argument
EXPOSE 3000                  # document port
VOLUME ["/data"]             # declare volume
USER node                    # run as user
HEALTHCHECK CMD curl -f http://localhost:3000/health
LABEL version="1.0"          # metadata
CMD ["node", "app.js"]       # default command
ENTRYPOINT ["node"]          # main executable
```

---

**Next:** [🐛 Common Errors & Fixes →](../13-Common-Errors/errors.md)
