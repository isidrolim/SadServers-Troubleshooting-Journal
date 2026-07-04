# Docker Cheatsheet

## Purpose

This cheatsheet is my quick Docker command reference.

It is organized by task type so I can quickly find the command I need while troubleshooting or practicing labs.

My reminder:

```text
Do not just copy commands.
Understand what layer I am checking:
image, container, logs, network, volume, daemon, or disk.
```

---

# Images

Images are read-only templates used to create containers.

| Command | Purpose |
|---|---|
| `docker pull nginx:1.25` | Download an image from a registry |
| `docker images` | List local images |
| `docker build -t myapp:1.0 .` | Build an image from a Dockerfile in the current directory |
| `docker rmi IMAGE_ID` | Remove an image |
| `docker history myapp:1.0` | Show image layer history |
| `docker inspect nginx:1.25` | Show image metadata in JSON format |

## Common Image Workflow

```bash
docker pull nginx:alpine
docker images
docker inspect nginx:alpine
```

---

# Containers – Run and Lifecycle

Containers are running or stopped instances of images.

| Command | Purpose |
|---|---|
| `docker run -d --name web -p 8080:80 nginx` | Run a container in the background and publish a port |
| `docker run -it --rm alpine sh` | Start an interactive shell and remove the container after exit |
| `docker ps` | Show running containers |
| `docker ps -a` | Show all containers, including stopped containers |
| `docker stop web` | Gracefully stop a container |
| `docker start web` | Start a stopped container |
| `docker rm web` | Remove a stopped container |
| `docker rm -f web` | Force remove a running container |
| `docker restart web` | Restart a container |

## Example: Run Nginx

```bash
docker run -d --name web -p 8080:80 nginx:alpine
docker ps
```

Meaning:

```text
Host port 8080
↓
Container port 80
```

---

# Exec, Logs, and Inspect

Use these commands when a container is running but you need to investigate what is happening inside it.

| Command | Purpose |
|---|---|
| `docker logs -f web` | Follow container logs live |
| `docker logs --tail 100 web` | Show the last 100 log lines |
| `docker exec -it web sh` | Open a shell inside a running container |
| `docker exec web cat /etc/hosts` | Run one command inside a container |
| `docker inspect web` | Show full container configuration and state |
| `docker top web` | Show processes running inside a container |

## Common Debug Flow

```bash
docker ps -a
docker logs web
docker inspect web
docker exec -it web sh
```

---

# Volumes and Mounts

Volumes are used when data must survive container removal.

| Command | Purpose |
|---|---|
| `docker volume ls` | List Docker volumes |
| `docker volume inspect vol1` | Show volume mount path on the host |
| `docker run -v /host/data:/data myapp` | Bind mount a host directory into the container |
| `docker run -v myvol:/data myapp` | Mount a Docker-managed named volume |

## Bind Mount vs Named Volume

```text
Bind mount:
Host path is explicitly chosen by me.

Named volume:
Docker manages the storage location.
```

## Example

```bash
docker volume create app_data
docker run -d -v app_data:/data myapp
```

---

# Networking

Use these commands when checking container connectivity or published ports.

| Command | Purpose |
|---|---|
| `docker network ls` | List Docker networks |
| `docker network inspect bridge` | Show network details and container IPs |
| `docker port web` | Show published port mappings |
| `ss -tulpn \| grep 8080` | Confirm host port is listening |

## Port Publishing Format

```text
-p host_port:container_port
```

Example:

```bash
docker run -d --name web -p 8080:80 nginx
```

Meaning:

```text
Host port 8080
↓
Container port 80
```

## Port Troubleshooting

```bash
docker ps
docker port web
sudo ss -tulpn | grep :8080
```

---

# Docker Compose

Docker Compose is used for multi-container applications.

| Command | Purpose |
|---|---|
| `docker compose up -d` | Start the stack in detached mode |
| `docker compose ps` | Show service status |
| `docker compose logs -f web` | Follow logs for a service |
| `docker compose down` | Stop and remove Compose containers and networks |
| `docker compose build` | Rebuild images |
| `docker compose exec web sh` | Open a shell inside a service container |

## Common Compose Workflow

```bash
docker compose up -d
docker compose ps
docker compose logs -f
docker compose down
```

## Important Note

Modern Docker uses:

```bash
docker compose
```

not always:

```bash
docker-compose
```

---

# Docker Daemon and Disk

These commands help check Docker service health and disk usage.

| Command | Purpose |
|---|---|
| `systemctl status docker` | Check Docker daemon status |
| `docker info` | Show Docker daemon summary and storage driver |
| `docker system df` | Show Docker disk usage |
| `docker system df -v` | Show detailed disk usage per object |

## Service Checks

```bash
sudo systemctl status docker
sudo systemctl is-active docker
docker info
```

## Disk Checks

```bash
df -h
docker system df
docker system df -v
```

---

# Cleanup and Prune

Prune commands remove unused Docker objects.

Be careful, especially with volumes.

## Safety Rule

Before deleting, list first.

```text
List first
↓
Understand what is unused
↓
Prune carefully
```

| Command | Purpose |
|---|---|
| `docker container prune` | Remove all stopped containers |
| `docker rm CONTAINER` | Remove one stopped container |
| `docker rm -f CONTAINER` | Force remove a running container |
| `docker image prune` | Remove dangling images |
| `docker image prune -a` | Remove images not used by any container |
| `docker rmi IMAGE` | Remove one image by name or ID |
| `docker volume prune` | Remove unused volumes |
| `docker volume rm VOL` | Remove one named volume |
| `docker network prune` | Remove unused custom networks |
| `docker builder prune` | Clear BuildKit build cache |
| `docker system prune` | Remove stopped containers, dangling images, and unused networks |
| `docker system prune -a` | Remove the above plus all unused images |
| `docker system prune -a --volumes` | Remove unused images and unused volumes |
| `docker compose down -v` | Stop Compose stack and remove Compose volumes |

## Safer Cleanup Order

```bash
# 1. Remove stopped containers
docker container prune -f

# 2. Remove dangling image layers
docker image prune -f

# 3. Remove unused images not referenced by containers
docker image prune -a -f

# 4. Remove unused networks
docker network prune -f

# 5. Remove build cache
docker builder prune -f

# 6. Remove volumes only if data is disposable
docker volume prune -f
```

## List Before Bulk Delete

```bash
docker ps -a --filter status=exited
docker images -f dangling=true
docker volume ls -f dangling=true
```

---

# Useful Inspect Filters

`docker inspect` gives a lot of JSON output. Filters help extract only what I need.

## Container Exit Code

```bash
docker inspect web | jq -r '.[0].State.ExitCode'
```

## Container Status

```bash
docker inspect web | jq -r '.[0].State.Status'
```

## Container IP Address

```bash
docker inspect web | jq -r '.[0].NetworkSettings.Networks[].IPAddress'
```

## Container Command

```bash
docker inspect web | jq '.[0].Config.Cmd'
```

## Container Mounts

```bash
docker inspect web | jq '.[0].Mounts'
```

## Published Ports

```bash
docker inspect web | jq '.[0].NetworkSettings.Ports'
```

---

# Common Troubleshooting Commands

## Check Docker Layer

```bash
docker --version
sudo systemctl is-active docker
sudo docker ps
```

## Check Container State

```bash
docker ps
docker ps -a
docker inspect CONTAINER
```

## Check Logs

```bash
docker logs CONTAINER
docker logs -f CONTAINER
docker logs --tail 100 CONTAINER
```

## Enter Container

```bash
docker exec -it CONTAINER sh
docker exec -it CONTAINER bash
```

## Check Port Mapping

```bash
docker port CONTAINER
sudo ss -tulpn | grep :PORT
```

## Check Resource Usage

```bash
docker stats
docker top CONTAINER
```

## Check Disk Usage

```bash
docker system df
df -h
```

---

# Pro Tips

## 1. Container Exited

If a container exits, check logs and exit code first.

```bash
docker ps -a
docker logs CONTAINER
docker inspect CONTAINER | jq -r '.[0].State.ExitCode'
```

Do not immediately recreate the container before understanding why it exited.

---

## 2. Port Mapping Mistake

This is a common mistake:

```text
-p 8080:80
```

means:

```text
Host port 8080
↓
Container port 80
```

The left side is the host.

The right side is the container.

---

## 3. PID 1 Matters

The main process inside the container runs as PID 1.

If PID 1 exits, the container stops.

If PID 1 does not handle signals correctly, `docker stop` may not work gracefully.

---

## 4. Use Explicit Image Tags

Avoid relying on:

```text
latest
```

Use explicit tags instead:

```bash
nginx:alpine
nginx:1.25
mysql:8.0
```

This helps avoid unexpected changes.

---

## 5. Disk Full Troubleshooting

If disk space is full, check Docker disk usage first:

```bash
docker system df -v
```

Then clean carefully:

```bash
docker container prune
docker image prune -a
docker volume prune
```

Only prune volumes if the data is truly disposable.

---

# My Docker Command Memory Map

```text
Images       → docker images, docker pull, docker build, docker rmi
Containers   → docker run, docker ps, docker stop, docker start, docker rm
Logs         → docker logs
Inside       → docker exec
Details      → docker inspect
Ports        → docker port, ss -tulpn
Volumes      → docker volume ls, docker volume inspect
Networks     → docker network ls, docker network inspect
Disk         → docker system df, docker prune
Compose      → docker compose up, ps, logs, down
```

---

# Final Reminder

Every Docker command should answer a question.

```text
Question: Is Docker running?
Command: systemctl status docker

Question: Is the container running?
Command: docker ps

Question: Why did it exit?
Command: docker logs and docker inspect

Question: Is the port published?
Command: docker port and ss

Question: Is disk full?
Command: docker system df
```

The goal is not only command memorization.

The goal is to know which command answers which troubleshooting question.
