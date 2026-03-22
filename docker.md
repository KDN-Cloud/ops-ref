# docker / compose cheat sheet

> Includes notes on **Dockhand** (`fnsys/dockhand`) where relevant to KDN Lab.

## containers

```bash
docker ps                          # list running containers
docker ps -a                       # list all containers (including stopped)
docker ps -q                       # list running container IDs only
docker run <image>                 # run container (foreground)
docker run -d <image>              # run detached (background)
docker run -it <image> bash        # interactive shell
docker run --rm <image>            # auto-remove on exit
docker run --name myapp <image>    # named container
docker run -p 8080:80 <image>      # map host:container port
docker run -e KEY=value <image>    # set env var
docker run -v /host:/container <image>  # bind mount
docker start <name|id>             # start stopped container
docker stop <name|id>              # graceful stop (SIGTERM)
docker kill <name|id>              # force stop (SIGKILL)
docker restart <name|id>           # restart container
docker rm <name|id>                # remove stopped container
docker rm -f <name|id>             # force remove running container
```

## exec & inspect

```bash
docker exec -it <name> bash        # shell into running container
docker exec -it <name> sh          # (if bash not available)
docker exec <name> <cmd>           # run command in container
docker logs <name>                 # view logs
docker logs -f <name>              # follow logs
docker logs --tail 100 <name>      # last 100 lines
docker logs --since 1h <name>      # logs from last hour
docker inspect <name>              # full JSON metadata
docker inspect -f '{{.NetworkSettings.IPAddress}}' <name>  # specific field
docker stats                       # live resource usage
docker stats --no-stream           # one-shot stats snapshot
docker top <name>                  # running processes in container
docker cp <name>:/path/file .      # copy file from container
docker cp ./file <name>:/path/     # copy file to container
docker diff <name>                 # changed files vs image
```

## images

```bash
docker images                      # list local images
docker images -a                   # include intermediate layers
docker pull <image>                # pull from registry
docker pull <image>:<tag>          # pull specific tag
docker push <image>:<tag>          # push to registry
docker rmi <image>                 # remove image
docker rmi -f <image>              # force remove
docker tag <src> <dest>            # tag an image
docker build -t <name>:<tag> .     # build from Dockerfile
docker build --no-cache -t <n> .  # build without cache
docker history <image>             # show image layers
docker inspect <image>             # image metadata
```

## cleanup

```bash
docker system prune                # remove stopped containers, unused networks, dangling images
docker system prune -a             # also remove unused images
docker system prune --volumes      # also remove unused volumes
docker container prune             # remove all stopped containers
docker image prune                 # remove dangling images
docker image prune -a              # remove all unused images
docker volume prune                # remove unused volumes
docker network prune               # remove unused networks
docker system df                   # disk usage summary
```

## volumes

```bash
docker volume create <name>        # create named volume
docker volume ls                   # list volumes
docker volume inspect <name>       # volume details
docker volume rm <name>            # remove volume
docker volume prune                # remove unused volumes
# bind mount (host path)
docker run -v /host/path:/container/path <image>
# named volume
docker run -v myvolume:/container/path <image>
# read-only
docker run -v /host/path:/container/path:ro <image>
```

## networks

```bash
docker network ls                  # list networks
docker network create <name>       # create bridge network
docker network create --driver overlay <name>  # overlay (swarm)
docker network inspect <name>      # network details
docker network connect <net> <container>       # connect container
docker network disconnect <net> <container>    # disconnect
docker network rm <name>           # remove network
```

## docker compose

```bash
docker compose up                  # start all services (foreground)
docker compose up -d               # start detached
docker compose up --build          # build images then start
docker compose up <service>        # start specific service
docker compose down                # stop and remove containers + networks
docker compose down -v             # also remove volumes
docker compose down --rmi all      # also remove images
docker compose stop                # stop without removing
docker compose start               # start stopped services
docker compose restart             # restart all services
docker compose restart <service>   # restart specific service
docker compose pull                # pull latest images
docker compose build               # build images
docker compose build --no-cache    # build without cache
docker compose logs                # view all logs
docker compose logs -f             # follow logs
docker compose logs <service>      # logs for one service
docker compose ps                  # list containers
docker compose exec <service> bash # shell into service
docker compose run --rm <service> <cmd>  # one-off command
docker compose config              # validate and print resolved compose file
docker compose top                 # show processes
```

### compose file structure

```yaml
name: myapp

services:
  app:
    image: myimage:latest
    container_name: myapp
    restart: unless-stopped
    ports:
      - "8080:80"
    environment:
      - KEY=value
    env_file:
      - .env
    volumes:
      - ./data:/app/data
      - myvolume:/app/cache
    networks:
      - frontend
      - backend
    depends_on:
      db:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    labels:
      - "com.example.description=My app"

  db:
    image: postgres:16
    restart: unless-stopped
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - dbdata:/var/lib/postgresql/data
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  myvolume:
  dbdata:

networks:
  frontend:
  backend:
    internal: true          # no external access
```

### env file (.env)

```bash
# .env (in same dir as compose file)
DB_PASSWORD=secret
IMAGE_TAG=latest

# reference in compose
image: myapp:${IMAGE_TAG}
```

### multiple compose files

```bash
# override / merge files
docker compose -f docker-compose.yml -f docker-compose.override.yml up

# common pattern: base + env-specific
docker compose -f compose.yml -f compose.prod.yml up -d
```

## dockerfile essentials

```dockerfile
FROM ubuntu:24.04

WORKDIR /app

# copy and install deps first (cache layer)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# then copy source
COPY . .

ENV PORT=8080
EXPOSE 8080

ARG BUILD_DATE
LABEL build-date=$BUILD_DATE

HEALTHCHECK --interval=30s --timeout=5s \
  CMD curl -f http://localhost:$PORT/health || exit 1

USER nonroot

ENTRYPOINT ["python"]
CMD ["app.py"]
```

### dockerfile best practices

```dockerfile
# combine RUN commands to reduce layers
RUN apt-get update && apt-get install -y \
    curl \
    vim \
  && rm -rf /var/lib/apt/lists/*

# use specific tags, not latest
FROM node:20-alpine

# .dockerignore (put next to Dockerfile)
# node_modules
# .git
# *.log
# .env
```

## networking tips

```bash
# find container IP
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' <name>

# containers on same network can reach each other by service name
# e.g. from 'app' container: curl http://db:5432

# host networking (Linux only — container shares host network stack)
docker run --network host <image>

# expose to localhost only
ports:
  - "127.0.0.1:8080:80"
```

## useful one-liners

```bash
# stop all running containers
docker stop $(docker ps -q)

# remove all stopped containers
docker rm $(docker ps -aq -f status=exited)

# remove all images
docker rmi $(docker images -q)

# shell into container as root (even if USER set)
docker exec -u 0 -it <name> bash

# show env vars inside container
docker exec <name> env

# watch compose logs across all services
docker compose logs -f --tail=50

# restart a service and follow its logs immediately
docker compose restart <service> && docker compose logs -f <service>

# check which container is using a port
docker ps --format "table {{.Names}}\t{{.Ports}}"

# pull all images in a compose file
docker compose pull && docker compose up -d
```

---

## dockhand & hawser agent (fnsys/dockhand)

> Dockhand is the container orchestration layer used in KDN Lab. Hawser is the per-host agent that connects each VM back to the Dockhand server. Each host runs one Hawser container with a unique `AGENT_NAME`.

### hawser agent — docker run

```bash
docker run -d \
  --name hawser \
  --restart unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -p 2376:2376 \
  -e TOKEN=<agent-token> \
  -e AGENT_NAME=<unique-hostname> \
  ghcr.io/finsys/hawser:latest
```

### hawser agent — docker compose (recommended)

Store as `compose.yml` alongside an `.env` file per host. Managed by Dockhand itself or started manually on first boot.

```yaml
# compose.yml
services:
  hawser:
    image: ghcr.io/finsys/hawser:latest
    container_name: hawser
    restart: unless-stopped
    ports:
      - "2376:2376"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - TOKEN=${HAWSER_TOKEN}
      - AGENT_NAME=${AGENT_NAME}
```

```bash
# .env (per host — never commit this file, add .env to .gitignore)
HAWSER_TOKEN=        # generate with: `openssl rand -base64 32`
AGENT_NAME={UNIQUE_NAME}
```

> **Generating a token:** run `openssl rand -base64 32` and paste the output as `HAWSER_TOKEN`. Generate a unique token per host and register it in the Dockhand server UI under the corresponding agent.

```bash
# start
docker compose up -d

# follow logs
docker compose logs -f hawser
```

### update hawser agent (recreate)

```bash
# stop and remove the existing container
docker stop hawser && docker rm hawser

# pull latest image
docker pull ghcr.io/finsys/hawser:latest

# restart via compose (preferred)
docker compose up -d

# or restart via run script if not using compose
bash hawser-agent-run.sh
```

### key behaviors

- `AGENT_NAME` must be **unique per host** — reusing names causes identity conflicts after restarts
- Stack names in Dockhand are independent of the compose `name:` field (e.g. a stack named `rxv4` in Dockhand deploys regardless of what `name:` is set to in the compose file)
- Stacks live under `envs/homelab-stacks/` (homelab VMs) or `envs/vps-stacks/` (Vultr VPS) in the kdnlab repo
- The Docker socket mount (`/var/run/docker.sock`) is required — Hawser manages containers on the host on behalf of Dockhand
- Port `2376` is the Hawser agent listener — ensure it's reachable from the Dockhand server

