# Docker Guide

## Purpose

This guide explains Docker concepts in a beginner-friendly way, using the same troubleshooting mindset I use in my labs.

The goal is not only to memorize Docker commands, but to understand what Docker is doing behind the scenes so I can troubleshoot containers properly.

---

## What Docker Does

Docker gives a standard way to build, ship, and run applications.

Instead of saying:

```text
It works on my machine.
```

Docker helps make the application run the same way on:

```text
Developer laptop
↓
CI/CD runner
↓
Test server
↓
Production server
↓
Kubernetes node
```

Docker is commonly used for:

- Running microservices
- Creating local development environments
- Testing applications in CI/CD pipelines
- Packaging applications with their dependencies
- Running containers under Kubernetes through container runtimes

The main idea is:

```text
Package the application once
↓
Run it consistently anywhere Docker is supported
```

---

## Images vs Containers

One of the most important Docker concepts is the difference between an image and a container.

### Image

An image is a read-only template.

It contains:

- Application files
- Dependencies
- Runtime libraries
- Metadata
- Default command or entrypoint

Images are usually built from a `Dockerfile` or pulled from a registry such as Docker Hub.

Example:

```bash
docker pull nginx:alpine
```

In this example:

```text
nginx  = image repository
alpine = image tag
```

### Container

A container is a running instance of an image.

If an image is like a class or blueprint, a container is the actual running object.

A container includes:

- A process
- A writable layer
- Runtime configuration
- Environment variables
- Network settings
- Mounted volumes

Example:

```bash
docker run -d --name nginx_1 nginx:alpine
```

This creates a running container named `nginx_1` from the `nginx:alpine` image.

---

## Simple Mental Model

```text
Image
↓
docker run
↓
Container
↓
Running process
```

An image does not run by itself.

A container is what actually runs.

---

## How a Container Starts

When I run:

```bash
docker run nginx:alpine
```

Docker performs several steps.

```text
Resolve image
↓
Pull image if missing locally
↓
Create container filesystem
↓
Apply namespaces and cgroups
↓
Start the container process
↓
Keep running until the main process exits
```

### 1. Image Resolve

Docker checks whether the image already exists locally.

If not, it pulls the image from a registry.

```bash
docker images
```

### 2. Container Create

Docker creates a container from the image.

This prepares the filesystem, metadata, and runtime configuration.

```bash
docker create nginx:alpine
```

### 3. Isolation Setup

Docker uses Linux kernel features to isolate the container.

Important features include:

- Namespaces
- Cgroups
- Overlay filesystem
- Container runtime

### 4. Process Start

The container starts its main process.

Inside the container, this process usually runs as PID 1.

If PID 1 exits, the container stops.

### 5. Lifecycle

A container keeps running as long as its main process keeps running.

If the main process exits, the container exits.

---

## Containers Are Not Virtual Machines

Containers are not full virtual machines.

A virtual machine has its own operating system kernel.

A container shares the host kernel.

```text
Virtual Machine:
App
↓
Guest OS
↓
Hypervisor
↓
Host OS

Container:
App
↓
Container isolation
↓
Host kernel
```

This is why containers are usually lighter and faster than VMs.

However, containers are not the same as full machines. They provide process-level isolation, not a completely separate operating system.

---

## Container Internals

A container is a normal Linux process with extra isolation and control applied by the kernel.

To troubleshoot containers properly, I need to understand these core internals:

```text
Namespaces
Cgroups
Overlay filesystem
Container runtime
Docker daemon
```

---

## Linux Namespaces

Namespaces give containers their own view of system resources.

They make the container feel isolated from the host.

Common namespaces:

```text
PID namespace      = process isolation
Network namespace  = network isolation
Mount namespace    = filesystem view isolation
UTS namespace      = hostname isolation
IPC namespace      = shared memory isolation
User namespace     = UID/GID mapping
```

### PID Namespace

Inside a container, process IDs start from 1.

The process that runs first inside the container becomes PID 1.

Example:

```bash
docker exec -it nginx_1 ps aux
```

The container may show its own PID 1, but the host sees a different process ID.

### Network Namespace

Each container usually gets its own network stack.

This includes:

- Interfaces
- IP address
- Routing table
- Ports

Published ports are mapped from host to container.

Example:

```bash
docker run -d -p 8080:80 nginx:alpine
```

This means:

```text
Host port 8080
↓
Container port 80
```

### Mount Namespace

Containers get their own filesystem view.

The container may see `/` as its root filesystem, but that root filesystem is not the same as the host `/`.

### UTS Namespace

This controls hostname isolation.

A container can have its own hostname.

### IPC Namespace

This isolates shared memory and inter-process communication.

### User Namespace

This can map container users to different host users.

This is important for rootless Docker or Podman.

---

## Cgroups

Cgroups control and measure resource usage.

They can limit:

- CPU
- Memory
- Number of processes
- I/O usage

Example:

```bash
docker run -d --memory 512m --cpus 1.5 nginx:alpine
```

This tells Docker to limit the container to:

```text
Memory: 512 MB
CPU: 1.5 CPUs
```

If a container uses too much memory, it may be killed by the kernel.

A common exit code for memory-related failure is:

```text
137
```

That usually means the process was killed, often due to out-of-memory behavior.

To inspect container resource usage:

```bash
docker stats
```

---

## Overlay Filesystem

Docker images are made of layers.

Most image layers are read-only.

When a container starts, Docker adds a writable layer on top.

```text
Base image layer        read-only
Package install layer   read-only
Application layer       read-only
Container layer         writable
```

When a running container creates or modifies files, those changes go into the writable container layer.

Important idea:

```text
Changes inside a container do not modify the original image.
```

If the container is removed, the writable layer is removed too.

That is why persistent data should not live only inside the container layer.

Use volumes or bind mounts for data that must survive.

---

## Volumes and Bind Mounts

Containers are temporary by nature.

If I need to keep data, I should use storage outside the container writable layer.

### Named Volume

A named volume is managed by Docker.

Example:

```bash
docker volume create app_data
```

Use it with a container:

```bash
docker run -d -v app_data:/var/lib/mysql mysql
```

Docker stores named volumes under its Docker data directory.

### Bind Mount

A bind mount maps a host path into a container.

Example:

```bash
docker run -d -v /host/path:/container/path nginx:alpine
```

This means:

```text
Host directory
↓
Visible inside container
```

Bind mounts are useful when I want the container to use files from the host.

### tmpfs Mount

A tmpfs mount stores data in memory.

It is useful for temporary data or secrets that should not be written to disk.

---

## Volume Troubleshooting Mindset

When a container cannot read or write files, ask:

```text
Is the correct volume mounted?
↓
Is the host path correct?
↓
Does the container user have permission?
↓
Is the file owned by the expected UID/GID?
↓
Is SELinux affecting access?
```

Common command:

```bash
docker inspect <container>
```

Look for the `Mounts` section.

---

## Dockerfile Essentials

A `Dockerfile` defines how an image is built.

Common instructions:

```dockerfile
FROM
RUN
COPY
ADD
ENV
EXPOSE
CMD
ENTRYPOINT
```

### FROM

Sets the base image.

Example:

```dockerfile
FROM nginx:alpine
```

### RUN

Runs commands during image build.

Example:

```dockerfile
RUN apk add --no-cache curl
```

### COPY

Copies files from the build context into the image.

Example:

```dockerfile
COPY index.html /usr/share/nginx/html/index.html
```

### ADD

Also copies files, but has extra behavior such as extracting archives.

In most cases, `COPY` is easier to understand and preferred.

### ENV

Sets environment variables.

Example:

```dockerfile
ENV APP_ENV=production
```

### EXPOSE

Documents which port the container expects to use.

Important:

```text
EXPOSE does not publish the port to the host.
```

To publish a port, use `-p` during `docker run`.

### CMD

Defines the default command when the container starts.

### ENTRYPOINT

Defines the main executable for the container.

---

## Dockerfile Layering

Each Dockerfile instruction can create a new image layer.

Layer ordering matters.

Put less-changing steps first:

```dockerfile
FROM python:3.12-slim
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY app/ /app/
```

Why?

Because Docker can cache earlier layers.

If only application code changes, Docker does not need to reinstall all dependencies every time.

---

## .dockerignore

The `.dockerignore` file excludes files from the build context.

Common entries:

```text
.git
node_modules
__pycache__
*.log
.env
dist
build
```

This helps keep images smaller and avoids accidentally copying sensitive or unnecessary files.

---

## Networking

Docker containers usually run on a bridge network by default.

Common network types:

```text
bridge
host
none
user-defined bridge
```

### Bridge Network

The default network mode.

Containers get private IPs, and host access is done through published ports.

Example:

```bash
docker run -d -p 8080:80 nginx:alpine
```

### Host Network

The container shares the host network stack.

Example:

```bash
docker run --network host nginx:alpine
```

This removes port mapping isolation.

### None Network

The container has no network access.

Example:

```bash
docker run --network none alpine
```

### User-Defined Bridge

Useful when containers need to talk to each other by name.

Example:

```bash
docker network create app_net
docker run -d --name web --network app_net nginx:alpine
docker run -it --network app_net alpine sh
```

Inside the same user-defined bridge network, containers can resolve each other by name.

---

## Port Publishing

Publishing a port maps a host port to a container port.

Example:

```bash
docker run -d -p 8080:80 nginx:alpine
```

Meaning:

```text
Host port 8080
↓
Container port 80
```

From the host, test:

```bash
curl http://localhost:8080
```

Important troubleshooting idea:

```text
A container can be running but still not reachable if the port is not published correctly.
```

---

## Docker Compose

Docker Compose is used to define and run multi-container applications.

A Compose file usually defines:

- Services
- Images
- Build context
- Ports
- Volumes
- Networks
- Environment variables
- Dependencies

Modern Docker Compose is usually run as:

```bash
docker compose up -d
```

Not:

```bash
docker-compose up -d
```

Example Compose structure:

```yaml
services:
  web:
    image: nginx:alpine
    ports:
      - "8080:80"
```

Start services:

```bash
docker compose up -d
```

Check services:

```bash
docker compose ps
```

View logs:

```bash
docker compose logs
```

Stop services:

```bash
docker compose down
```

---

## Registry and Image Lifecycle

Docker images are usually pulled from registries.

Examples of registries:

- Docker Hub
- Amazon ECR
- Google Artifact Registry
- Harbor
- GitHub Container Registry

Common image commands:

```bash
docker pull nginx:alpine
docker images
docker tag
docker push
docker rmi
```

Images can take a lot of disk space, especially on CI/CD runners and development servers.

Useful cleanup commands:

```bash
docker system df
docker image prune
docker system prune
```

Be careful with prune commands because they can remove unused containers, images, networks, and caches.

---

## Docker Daemon and CLI

Docker has two major parts:

```text
docker CLI
↓
dockerd daemon
↓
containerd
↓
runc
↓
Linux kernel features
```

### Docker CLI

This is the command I type:

```bash
docker ps
```

### Docker Daemon

The daemon is the background service that manages:

- Images
- Containers
- Volumes
- Networks

The Docker socket is usually:

```text
/var/run/docker.sock
```

If I see permission errors such as:

```text
permission denied while trying to connect to the Docker daemon socket
```

I should check:

```text
Is Docker running?
Is my user allowed to access Docker?
Am I using sudo?
Is the user in the docker group?
```

Validation:

```bash
sudo docker ps
```

---

## containerd

Docker uses `containerd` as a lower-level container runtime component.

Docker does not directly handle every low-level container operation itself.

A simplified view:

```text
docker CLI
↓
dockerd
↓
containerd
↓
runc
↓
namespaces, cgroups, root filesystem
↓
container process
```

Kubernetes also uses container runtimes such as containerd through the Container Runtime Interface.

In Kubernetes, users usually do not run:

```bash
docker run
```

Instead, Kubernetes schedules pods and the container runtime starts containers.

---

## Podman

Podman is related to Docker but works differently.

Podman provides a Docker-compatible command style, but it does not require a long-running Docker daemon.

This is common in RHEL, Fedora, and some enterprise Linux environments.

Example:

```bash
podman run
podman ps
podman build
```

Sometimes a package such as `podman-docker` provides a `docker` command that actually runs Podman underneath.

Important troubleshooting lesson:

```text
Podman can emulate Docker commands,
but it is not always the same as Docker CE.
```

Some labs or automated checkers may specifically require real Docker packages such as:

```text
docker-ce
docker-compose-plugin
```

So I should verify what the task requires before using Podman as a substitute.

---

## Production Practices

In production, containers should be simple, predictable, and observable.

Important practices:

### One Main Process Per Container

A container should usually run one main application process.

If multiple processes are needed, use a proper init process or split the workload into separate containers.

### Logs to stdout/stderr

Containers should write logs to stdout and stderr.

Then Docker can collect logs using:

```bash
docker logs <container>
```

### Health Checks

Images or Compose files can define health checks.

This helps detect whether the application inside the container is actually working.

### Resource Limits

Set CPU and memory limits on shared hosts.

Example:

```bash
docker run --memory 512m --cpus 1 nginx:alpine
```

### Non-Root User

When possible, containers should not run as root.

In Dockerfiles, this can be done with:

```dockerfile
USER appuser
```

### Read-Only Filesystem

For better security, containers can run with a read-only root filesystem.

Writable locations can be provided through volumes or tmpfs mounts.

---

## Systematic Docker Troubleshooting Mindset

When a container problem happens, do not guess.

Use this process:

```text
Identify the symptom
↓
Build the Docker dependency path
↓
Verify one layer at a time
↓
Find the first failure
↓
Fix the first failure
↓
Validate the result
```

Common Docker dependency path:

```text
Docker installed
↓
Docker service running
↓
Image available
↓
Container created
↓
Container running
↓
Ports published correctly
↓
Volumes mounted correctly
↓
Application responding
```

---

## Common First Questions

When troubleshooting Docker, ask:

```text
Is Docker installed?
Is Docker running?
Does the image exist?
Was the container created?
Is the container running or exited?
What do the logs say?
Is the port published?
Is the application listening inside the container?
Are volumes mounted correctly?
Are permissions correct?
```

Useful commands:

```bash
docker --version
sudo systemctl status docker
docker images
docker ps
docker ps -a
docker logs <container>
docker inspect <container>
docker exec -it <container> sh
docker system df
```

---

## Learning Goal

My goal is not only to know Docker commands.

My goal is to understand the Docker path:

```text
Image
↓
Container
↓
Process
↓
Network
↓
Storage
↓
Application
```

When something breaks, I want to know where to look, how to verify it, and how to fix the first failed layer.
