# Containers Reference (Docker & Docker Compose)

## Docker Installation (Ubuntu)

```bash
# Install Docker Engine (official method)
curl -fsSL https://get.docker.com | sh

# Add non-root user to docker group (no sudo needed for docker commands)
usermod -aG docker deploy
# User must log out and back in for group change to take effect

# Verify installation
docker --version
docker run hello-world

# Enable Docker on boot
systemctl enable docker
```

---

## Docker Basics

### Container Lifecycle
```bash
# Run a container
docker run -d --name myapp -p 8080:80 nginx          # detached, port mapping
docker run -d --name myapp -p 8080:80 --restart unless-stopped nginx  # auto-restart
docker run -it ubuntu:24.04 bash                      # interactive shell

# Stop / start / restart
docker stop myapp
docker start myapp
docker restart myapp

# Remove
docker stop myapp && docker rm myapp
docker rm -f myapp                                     # force remove running container

# List containers
docker ps                           # running only
docker ps -a                        # all (including stopped)

# Logs
docker logs myapp                   # all logs
docker logs -f myapp                # follow (like tail -f)
docker logs --tail 100 myapp        # last 100 lines
docker logs --since 1h myapp        # last hour

# Execute command in running container
docker exec -it myapp bash          # interactive shell
docker exec myapp cat /etc/nginx/nginx.conf   # single command

# Inspect container
docker inspect myapp
docker stats                        # real-time resource usage (CPU, mem, net, IO)
docker top myapp                    # processes inside container
```

### Images
```bash
# Pull an image
docker pull nginx:latest
docker pull node:22-alpine

# List images
docker images

# Build from Dockerfile
docker build -t myapp:latest .
docker build -t myapp:v1.2 -f Dockerfile.prod .

# Remove images
docker rmi nginx:latest
docker rmi $(docker images -q --filter "dangling=true")   # remove untagged

# Tag and push to registry
docker tag myapp:latest registry.example.com/myapp:latest
docker push registry.example.com/myapp:latest
```

### Volumes (persistent data)
```bash
# Named volume (Docker manages the path)
docker run -d -v mydata:/var/lib/mysql --name db mysql:8

# Bind mount (map host directory)
docker run -d -v /var/www/html:/usr/share/nginx/html --name web nginx

# List volumes
docker volume ls

# Inspect volume
docker volume inspect mydata

# Remove unused volumes
docker volume prune
```

### Environment Variables & Secrets
```bash
# Pass env vars
docker run -d -e MYSQL_ROOT_PASSWORD=secret -e MYSQL_DATABASE=mydb mysql:8

# From env file
docker run -d --env-file .env myapp:latest

# Docker secrets (Swarm mode only — for Compose, use env_file)
```

---

## Docker Compose

### Install
```bash
# Docker Compose v2 is included with Docker Engine (as "docker compose")
docker compose version

# If not available, install the plugin
apt install docker-compose-plugin -y
```

### Basic Compose File (`docker-compose.yml`)
```yaml
version: "3.9"

services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgres://user:pass@db:5432/mydb
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped
    volumes:
      - ./uploads:/app/uploads

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: mydb
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    volumes:
      - redisdata:/data

volumes:
  pgdata:
  redisdata:
```

### Compose Commands
```bash
# Start all services
docker compose up -d

# Start specific service
docker compose up -d app

# Stop all services
docker compose down

# Stop and remove volumes (destroys data)
docker compose down -v

# Rebuild and restart
docker compose up -d --build

# View logs
docker compose logs -f
docker compose logs -f app          # specific service

# List running services
docker compose ps

# Execute command in service
docker compose exec app bash
docker compose exec db psql -U user mydb

# Scale a service (if no host port conflict)
docker compose up -d --scale worker=3

# Pull latest images
docker compose pull
```

### Production Compose Patterns

#### Nginx Reverse Proxy + App + DB
```yaml
version: "3.9"

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
      - static_files:/var/www/static:ro
    depends_on:
      - app
    restart: unless-stopped

  app:
    build: .
    expose:
      - "3000"                      # internal only — Nginx proxies to this
    env_file: .env
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped
    volumes:
      - static_files:/app/static
      - uploads:/app/uploads

  db:
    image: postgres:16-alpine
    env_file: .env.db
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready"]
      interval: 10s
      retries: 5
    restart: unless-stopped

volumes:
  pgdata:
  static_files:
  uploads:
```

---

## Container Networking

### Network Basics
```bash
# List networks
docker network ls

# Create a custom network
docker network create mynet

# Run container on custom network
docker run -d --name app --network mynet myapp:latest

# Connect existing container to network
docker network connect mynet existing_container

# Inspect network
docker network inspect mynet
```

### DNS Resolution
- Containers on the same Docker network can reach each other by **service name**
- `app` can connect to `db` using hostname `db` (not `localhost`)
- This is why Compose services use service names in connection strings

### Expose vs Ports
```yaml
services:
  app:
    ports:
      - "3000:3000"       # published — accessible from host and outside
    expose:
      - "3000"            # internal only — accessible from other containers
```

---

## Image Cleanup & Disk Management

Docker can consume significant disk space over time.

### Check Docker Disk Usage
```bash
docker system df                    # overview: images, containers, volumes, cache
docker system df -v                 # detailed breakdown
```

### Cleanup Commands
```bash
# Remove stopped containers, unused networks, dangling images, build cache
docker system prune

# Also remove all unused images (not just dangling)
docker system prune -a

# Remove only specific types
docker container prune              # stopped containers
docker image prune                  # dangling images
docker image prune -a               # all unused images
docker volume prune                 # unused volumes (WARNING: deletes data)
docker builder prune                # build cache

# Remove images older than 24 hours
docker image prune -a --filter "until=24h"
```

### Automated Cleanup Cron
```bash
# Weekly cleanup — remove stopped containers and dangling images
echo "0 3 * * 0 root docker system prune -f >> /var/log/docker-cleanup.log 2>&1" > /etc/cron.d/docker-cleanup
```

---

## Dockerfile Best Practices

```dockerfile
# Use specific version tags, not :latest
FROM node:22-alpine

# Set working directory
WORKDIR /app

# Copy dependency files first (better caching)
COPY package.json package-lock.json ./
RUN npm ci --only=production

# Copy app source
COPY . .

# Build step (if needed)
RUN npm run build

# Run as non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# Expose port (documentation — doesn't actually publish)
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1

# Start command
CMD ["node", "dist/index.js"]
```

### Multi-Stage Build (smaller images)
```dockerfile
# Build stage
FROM node:22-alpine AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

# Production stage
FROM node:22-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./
USER node
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

---

## Docker Logging

```bash
# Configure logging driver in /etc/docker/daemon.json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}

# Restart Docker to apply
systemctl restart docker

# Per-container log limits (in Compose)
services:
  app:
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
```

---

## Docker Security

```bash
# Never run containers as root (use USER in Dockerfile)
# Never use --privileged unless absolutely necessary
# Use read-only filesystem where possible
docker run --read-only --tmpfs /tmp myapp:latest

# Limit resources
docker run -d --memory=512m --cpus=1.0 myapp:latest

# In Compose:
services:
  app:
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
        reservations:
          cpus: '0.5'
          memory: 256M

# Scan images for vulnerabilities
docker scout cves myapp:latest       # Docker Scout (built-in)
trivy image myapp:latest             # Trivy (popular alternative)
```
