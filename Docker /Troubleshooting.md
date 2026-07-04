# Docker Troubleshooting

## Purpose

This document is my Docker troubleshooting guide.

The goal is not only to know commands, but to understand how to investigate Docker problems step by step.

My troubleshooting rule:

```text
Do not guess.
Identify the symptom.
Build the dependency path.
Verify one layer at a time.
Find the first failure.
Fix the first failure.
Validate the result.
```

---

## Docker Troubleshooting Mindset

Most Docker problems can be solved by checking the path from the host to the running application.

```text
Docker installed
↓
Docker daemon running
↓
Docker CLI can talk to daemon
↓
Image exists or can be pulled
↓
Container exists
↓
Container is running
↓
Application process is alive
↓
Ports are published correctly
↓
Volumes and permissions are correct
↓
Application responds
```

When something breaks, I should not immediately restart or rebuild.

I should ask:

```text
What is the symptom?
What must be true for this to work?
Which layer fails first?
```

---

# 1. Cannot Connect to the Docker Daemon

## Symptom

Docker commands fail with an error similar to:

```text
permission denied while trying to connect to the Docker daemon socket
```

or:

```text
Cannot connect to the Docker daemon
```

## What This Usually Means

Docker CLI cannot communicate with the Docker daemon.

Possible causes:

```text
Docker service is not running
Current user does not have permission to access Docker socket
Docker socket does not exist
User is not in the docker group
```

## Dependency Path

```text
Docker CLI
↓
Docker socket
↓
Docker daemon
↓
Docker engine responds
```

## Verification

Check Docker service:

```bash
sudo systemctl status docker
```

Check if Docker is active:

```bash
sudo systemctl is-active docker
```

Check Docker socket:

```bash
ls -l /var/run/docker.sock
```

Test with sudo:

```bash
sudo docker ps
```

## Fix

Start Docker if it is stopped:

```bash
sudo systemctl enable --now docker
```

If Docker works with `sudo` but not as a normal user, add the user to the docker group:

```bash
sudo usermod -aG docker $USER
```

Then log out and log back in.

## Validation

```bash
docker ps
```

or:

```bash
sudo docker ps
```

Expected result:

```text
CONTAINER ID   IMAGE   COMMAND   CREATED   STATUS   PORTS   NAMES
```

---

# 2. Container Exits Immediately

## Symptom

A container starts and then quickly stops.

Example:

```bash
docker ps
```

does not show it, but:

```bash
docker ps -a
```

shows the container as `Exited`.

## What This Usually Means

The main process inside the container stopped.

A container only stays running while its main process is running.

Common reasons:

```text
Application crashed
Wrong command or entrypoint
Missing configuration
Missing environment variables
Script is not executable
Image has no long-running process
```

## Dependency Path

```text
Container created
↓
Entrypoint or CMD starts
↓
Main process stays alive
↓
Container remains running
```

## Verification

Show all containers:

```bash
docker ps -a
```

Check logs:

```bash
docker logs <container>
```

Inspect exit code:

```bash
docker inspect <container> | jq '.[0].State.ExitCode'
```

Inspect full state:

```bash
docker inspect <container> | jq '.[0].State'
```

## Debug by Starting a Shell

If the container exits immediately, override the entrypoint and start a shell:

```bash
docker run -it --entrypoint sh <image>
```

If `sh` is not available, try:

```bash
docker run -it --entrypoint /bin/bash <image>
```

This helps inspect files, environment variables, and startup scripts inside the image.

## Validation

After fixing the issue:

```bash
docker run -d --name <container> <image>
docker ps
```

Expected:

```text
Container status is Up
```

---

# 3. Exit Codes for Stopped Containers

Exit codes help explain why a container stopped.

Check exit code:

```bash
docker inspect <container> | jq '.[0].State.ExitCode'
```

## Common Exit Codes

```text
0     = clean exit
1     = application error
126   = command found but not executable
127   = command not found
137   = killed, often SIGKILL or out-of-memory
143   = graceful stop by SIGTERM
```

---

## Exit Code 0 – Clean Exit

## Meaning

The container completed its task successfully and exited.

This is normal for short-lived jobs.

Example:

```bash
docker run alpine echo "hello"
```

The command runs, prints output, and exits.

## Troubleshooting Question

```text
Was this container supposed to keep running?
```

If yes, then the image or command may not be designed as a long-running service.

---

## Exit Code 1 – Application Error

## Meaning

The application crashed or returned a general error.

## Verification

```bash
docker logs <container>
docker inspect <container> | jq '.[0].State.ExitCode'
```

## Common Causes

```text
Bad configuration
Missing file
Database connection failure
Unhandled application exception
Missing environment variable
```

## Fix Direction

Read the logs first.

Do not guess.

---

## Exit Code 126 – Not Executable

## Meaning

Docker found the command, but it could not execute it.

Common example:

```text
Startup script exists but does not have execute permission
```

## Verification

Run a shell inside the image:

```bash
docker run --rm -it --entrypoint sh <image>
```

Check file permissions:

```bash
ls -l /path/to/script.sh
```

## Fix

In the Dockerfile:

```dockerfile
RUN chmod +x /path/to/script.sh
```

---

## Exit Code 127 – Command Not Found

## Meaning

The command or binary does not exist in the image, or it is not in `PATH`.

## Verification

```bash
docker run --rm <image> which <command>
```

or:

```bash
docker run --rm -it --entrypoint sh <image>
```

Then check:

```bash
which <command>
echo $PATH
```

## Fix

Correct the `CMD` or `ENTRYPOINT`, or install the missing package.

---

## Exit Code 137 – Force Killed or Out of Memory

## Meaning

The container was killed with SIGKILL.

This often happens when the container exceeds its memory limit.

## Verification

Check if Docker marked it as OOM killed:

```bash
docker inspect <container> | jq '.[0].State.OOMKilled'
```

Check memory limit:

```bash
docker inspect <container> | jq '.[0].HostConfig.Memory'
```

Check kernel logs:

```bash
dmesg | grep -i oom
```

## Fix Direction

If `OOMKilled` is true:

```text
Increase memory limit
Reduce memory usage
Fix memory leak
```

If `OOMKilled` is false but exit code is 137, someone or something may have force-killed the container.

---

## Exit Code 143 – Graceful Stop

## Meaning

The container received SIGTERM and exited cleanly.

This often happens after:

```bash
docker stop <container>
```

Docker sends SIGTERM first, waits, then sends SIGKILL if the process does not exit.

## Troubleshooting Question

```text
Did someone stop the container intentionally?
Did an orchestrator restart it?
Did the application handle SIGTERM correctly?
```

---

# 4. SIGTERM Handling Problems

## Symptom

A container does not stop gracefully, or it gets killed after timeout.

## Why This Happens

The application may not receive SIGTERM properly.

This is common when the Dockerfile uses shell form.

## Less Ideal Example

```dockerfile
CMD sh -c "myapp"
```

In this case, `sh` may become PID 1, and signals may not reach `myapp` properly.

## Better Example

```dockerfile
CMD ["myapp"]
```

This runs the application directly as PID 1.

## Why It Matters

PID 1 inside a container has special behavior.

If the real application does not receive signals, shutdown may be messy.

## Fix Direction

Prefer exec form:

```dockerfile
CMD ["myapp"]
```

or:

```dockerfile
ENTRYPOINT ["myapp"]
```

---

# 5. Port Already in Use

## Symptom

Docker fails to start a container with a port publishing error.

Example:

```text
bind: address already in use
```

## What This Means

The host port is already being used by another process or container.

Example:

```bash
docker run -p 8080:80 nginx
```

fails because port `8080` is already taken on the host.

## Dependency Path

```text
Host port
↓
Available
↓
Docker publishes port
↓
Container receives traffic
```

## Verification

Check what owns the port:

```bash
sudo ss -tulpn | grep :8080
```

or:

```bash
sudo lsof -i :8080
```

Check Docker containers:

```bash
docker ps
```

## Fix Options

Stop the conflicting process:

```bash
sudo kill <PID>
```

or stop the conflicting container:

```bash
docker stop <container>
```

or use a different host port:

```bash
docker run -d -p 8081:80 nginx
```

## Important Format

```text
-p host_port:container_port
```

Example:

```bash
docker run -d -p 8081:80 nginx
```

Means:

```text
Host port 8081
↓
Container port 80
```

---

# 6. Cannot Access Service on Published Port

## Symptom

The container is running, but the service is not reachable from the host.

Example:

```bash
curl http://localhost:8080
```

fails.

## Possible Causes

```text
Port was not published
Wrong host port
Application listens only on 127.0.0.1 inside the container
Firewall blocks traffic
Application is not running inside the container
```

## Dependency Path

```text
Host request
↓
Published host port
↓
Docker NAT
↓
Container port
↓
Application listens inside container
```

## Verification

Check published ports:

```bash
docker port <container>
```

Check container process/listener:

```bash
docker exec <container> ss -tulpn
```

or:

```bash
docker exec <container> netstat -tulpn
```

Test from inside the container:

```bash
docker exec <container> curl -s localhost:<port>
```

Inspect Docker port mapping:

```bash
docker inspect <container> | jq '.[0].NetworkSettings.Ports'
```

## Fix Direction

Make sure the app listens on:

```text
0.0.0.0
```

not only:

```text
127.0.0.1
```

inside the container.

Also verify the `-p` mapping is correct.

---

# 7. Volume Permission Denied

## Symptom

The container cannot read or write to a mounted directory.

Example:

```text
Permission denied
```

## What This Usually Means

The host directory ownership or permissions do not match the user inside the container.

Common causes:

```text
Host path owned by root
Container runs as non-root user
UID/GID mismatch
SELinux blocks the mount
Wrong mount path
```

## Dependency Path

```text
Host directory
↓
Docker mount
↓
Container path
↓
Container user permission
↓
Application writes successfully
```

## Verification

Check host directory:

```bash
ls -la /host/path
```

Inspect container user:

```bash
docker inspect <container> | jq '.[0].Config.User'
```

Inspect mounts:

```bash
docker inspect <container> | jq '.[0].Mounts'
```

Check from inside the container:

```bash
docker exec -it <container> sh
```

Then:

```bash
id
ls -la /container/path
```

## Fix Options

Change ownership on the host:

```bash
sudo chown -R <uid>:<gid> /host/path
```

Run container with a specific user:

```bash
docker run --user <uid>:<gid> ...
```

For Compose:

```yaml
services:
  app:
    user: "1000:1000"
```

On SELinux systems, try a relabel option:

```bash
docker run -v /host/path:/container/path:Z ...
```

---

# 8. Image Pull Fails

## Symptom

Docker cannot pull an image.

Example:

```bash
docker pull nginx:alpine
```

fails.

## Possible Causes

```text
No internet access
DNS failure
Proxy issue
Registry authentication required
Rate limit from registry
Image name or tag is wrong
TLS inspection or certificate problem
```

## Verification

Test pulling a known image:

```bash
docker pull nginx:alpine
```

Test DNS:

```bash
nslookup registry-1.docker.io
```

Test internet from host:

```bash
curl -I https://registry-1.docker.io
```

Check Docker daemon logs:

```bash
journalctl -u docker
```

## Fix Direction

For private registry:

```bash
docker login
```

For DNS issues, check daemon DNS configuration:

```bash
cat /etc/docker/daemon.json
```

Example DNS config:

```json
{
  "dns": ["8.8.8.8", "1.1.1.1"]
}
```

Restart Docker after daemon config changes:

```bash
sudo systemctl restart docker
```

---

# 9. No Space Left on Device

## Symptom

Docker commands fail with:

```text
No space left on device
```

## What Usually Consumes Space

```text
Images
Stopped containers
Build cache
Volumes
Container logs
```

Docker data is commonly stored under:

```text
/var/lib/docker
```

## Verification

Check disk usage:

```bash
df -h
```

Check Docker disk usage:

```bash
docker system df
```

Check large Docker logs:

```bash
sudo du -sh /var/lib/docker/containers/*/*-json.log 2>/dev/null
```

## Cleanup Commands

Remove stopped containers:

```bash
docker container prune
```

Remove unused images:

```bash
docker image prune
```

Remove unused volumes:

```bash
docker volume prune
```

More aggressive cleanup:

```bash
docker system prune
```

Be careful with:

```bash
docker system prune -a
```

because it can remove all unused images, including images you may want to keep.

## Prevent Log Growth

Configure log rotation.

Example daemon configuration:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

---

# 10. Container OOM Killed

## Symptom

Container exits with code `137`, and `OOMKilled` is true.

## Verification

```bash
docker inspect <container> | jq '.[0].State.OOMKilled'
```

Check memory usage:

```bash
docker stats
```

Check host logs:

```bash
dmesg | grep -i oom
```

## Fix Direction

Increase memory limit:

```bash
docker run --memory 1g <image>
```

or in Compose:

```yaml
services:
  app:
    mem_limit: 1g
```

Also investigate memory leaks or runaway processes.

---

# 11. Compose Service Unhealthy or Not Starting

## Symptom

A Docker Compose service is not starting or is marked unhealthy.

## Verification

Check service state:

```bash
docker compose ps
```

Check logs:

```bash
docker compose logs <service>
```

Check Compose config:

```bash
docker compose config
```

## Common Causes

```text
Bad environment variable
Wrong file path
Wrong volume mount
Wrong build context
Dependency not ready
Healthcheck failing
```

## Important Note About depends_on

`depends_on` controls startup order.

It does not always mean the dependency is ready.

Example problem:

```text
Database container started
but database is not ready to accept connections yet
```

For readiness, use health checks.

---

# 12. Container Cannot Reach Internet or Host

## Symptom

Container cannot resolve DNS or connect to external services.

## Verification

Test DNS inside the container:

```bash
docker exec <container> nslookup google.com
```

or:

```bash
docker exec <container> ping -c1 8.8.8.8
```

Check Docker networks:

```bash
docker network ls
```

Inspect container network:

```bash
docker inspect <container> | jq '.[0].NetworkSettings.Networks'
```

## Possible Causes

```text
DNS issue
Bridge network issue
Firewall issue
VPN interference
Proxy configuration missing
Custom iptables rules
```

## Fix Direction

Check Docker daemon DNS settings:

```bash
cat /etc/docker/daemon.json
```

Restart Docker if daemon networking configuration is changed:

```bash
sudo systemctl restart docker
```

For reaching host services from a container, use:

```text
host.docker.internal
```

or add a host gateway entry if needed.

Example:

```bash
docker run --add-host=host.docker.internal:host-gateway ...
```

---

# 13. Debugging Dockerfile Build Issues

Dockerfile problems can happen at build time or runtime.

## Build-Time Failure

The image fails during:

```bash
docker build .
```

## Runtime Failure

The image builds successfully, but the container fails when started.

These are different problems.

Do not troubleshoot them the same way.

---

## Core Build Debugging Commands

Build without cache:

```bash
docker build --no-cache -t myimage .
```

Show detailed BuildKit output:

```bash
DOCKER_BUILDKIT=1 docker build --progress=plain -t myimage . 2>&1 | tee build.log
```

Inspect image layers:

```bash
docker history --no-trunc myimage
```

Inspect image metadata:

```bash
docker image inspect myimage
```

---

## Debug Last Successful Layer

If a build fails, Docker may show the last successful layer or intermediate image.

You can start a shell from a successful layer or image:

```bash
docker run --rm -it <image_or_layer_id> /bin/sh
```

Then manually test the failing command.

This helps confirm whether the issue is:

```text
Missing package
Wrong file path
Bad permissions
Network issue
Wrong working directory
```

---

# 14. Layer and Cache Issues

## Symptom

The image builds, but it seems to use old code or stale dependencies.

## Cause

Docker cache may be reused because the Dockerfile order does not invalidate the right layer.

## Bad Pattern

```dockerfile
RUN npm install
COPY . .
```

If dependency files were not copied before `npm install`, Docker cache may not behave as expected.

## Better Pattern

```dockerfile
COPY package*.json ./
RUN npm install
COPY . .
```

Why this is better:

```text
Dependency files change less often than application source code.
Docker can cache dependency install separately.
Application code changes do not always force dependency reinstall.
```

## Verification

Inspect layers:

```bash
docker history --no-trunc myimage
```

Inspect root filesystem layers:

```bash
docker image inspect myimage | jq '.[0].RootFS.Layers'
```

---

# 15. Package Install Failures in Dockerfile

## Symptom

Package install fails during build.

Example:

```dockerfile
RUN apt-get install -y curl
```

fails because package lists are stale or missing.

## Bad Pattern

```dockerfile
RUN apt-get update
RUN apt-get install -y curl
```

This can cause stale cache issues.

## Better Pattern

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    wget \
    && rm -rf /var/lib/apt/lists/*
```

Why this is better:

```text
Updates package index and installs packages in one layer.
Avoids stale package cache.
Removes package lists to reduce image size.
```

## Pin Versions When Needed

```dockerfile
RUN apt-get update && apt-get install -y curl=7.88.1-10
```

This helps avoid unexpected package version changes between builds.

---

# 16. File, Permission, and .dockerignore Issues

## Symptom

A file expected inside the image is missing or not executable.

## Verification

Run the image and inspect the file:

```bash
docker run --rm myimage ls -la /app
```

Check a script:

```bash
docker run --rm myimage stat /app/start.sh
```

## Common Causes

```text
File was not copied
Wrong COPY path
.dockerignore excluded the file
Script missing execute permission
Wrong ownership
```

## Fix Examples

Copy the script:

```dockerfile
COPY start.sh /app/
```

Make it executable:

```dockerfile
RUN chmod +x /app/start.sh
```

Set ownership:

```dockerfile
COPY --chown=appuser:appuser . /app
```

Check `.dockerignore`:

```bash
cat .dockerignore
```

---

# 17. ARG, ENV, and Secrets

## ARG

`ARG` is available during image build.

Example:

```dockerfile
ARG BUILD_ENV=production
```

## ENV

`ENV` is available inside the running container.

Example:

```dockerfile
ENV APP_ENV=production
```

## Important Difference

```text
ARG = build time
ENV = runtime
```

## Inspect Runtime Environment

```bash
docker run --rm myimage env
```

Inspect image environment:

```bash
docker inspect myimage | jq '.[0].Config.Env'
```

## Secret Warning

Do not bake secrets into images.

Bad:

```dockerfile
ENV API_KEY=supersecret
```

This can remain in image layers.

Better approach:

```text
Use Docker secrets
Use environment variables at runtime
Use mounted secret files
Use CI/CD secret injection
```

---

# 18. Multi-Stage Build Issues

## Purpose

Multi-stage builds help create smaller production images.

Example:

```dockerfile
FROM node:20 AS builder
WORKDIR /app
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
```

## Common Mistake

Copying the wrong path from the builder stage.

Bad:

```dockerfile
COPY --from=builder /dist /usr/share/nginx/html
```

If the build output is actually under:

```text
/app/dist
```

then the correct path is:

```dockerfile
COPY --from=builder /app/dist /usr/share/nginx/html
```

## Debug Builder Stage

Build only the builder target:

```bash
docker build --target builder -t debug-builder .
```

Run a shell:

```bash
docker run --rm -it debug-builder /bin/sh
```

Then inspect:

```bash
ls -la /app
ls -la /app/dist
```

---

# 19. ENTRYPOINT and CMD Debugging

## Verification

Inspect image startup config:

```bash
docker inspect myimage | jq '.[0].Config | {Entrypoint, Cmd}'
```

## Example

```dockerfile
ENTRYPOINT ["myapp"]
CMD ["--config", "/etc/default.conf"]
```

Meaning:

```text
ENTRYPOINT = main executable
CMD        = default arguments
```

## Troubleshooting Question

```text
Is the startup command correct?
Does the binary exist?
Are the arguments correct?
Is the script executable?
```

Prefer exec form:

```dockerfile
CMD ["myapp"]
```

instead of shell form:

```dockerfile
CMD myapp
```

Exec form handles signals better.

---

# 20. Network and DNS During Build

## Symptom

Docker build cannot download packages or reach external services.

## Verification

Test DNS from a temporary container:

```bash
docker run --rm busybox nslookup google.com
```

Build with host networking if needed:

```bash
docker build --network=host .
```

Set DNS during build:

```bash
docker build --dns 8.8.8.8 .
```

Use proxy build arguments if behind a proxy:

```bash
docker build \
  --build-arg HTTP_PROXY=http://proxy:3128 \
  --build-arg HTTPS_PROXY=http://proxy:3128 .
```

---

# 21. Debugging Workflow

When a Docker issue happens, follow this sequence.

## Step 1: Check Container State

```bash
docker ps -a
```

Inspect container details:

```bash
docker inspect <container>
```

Useful inspect summary:

```bash
docker inspect <container> | jq '{status: .[0].State.Status, exit: .[0].State.ExitCode, oom: .[0].State.OOMKilled}'
```

---

## Step 2: Check Logs

```bash
docker logs <container>
```

Follow logs live:

```bash
docker logs -f <container>
```

Tail recent logs:

```bash
docker logs --tail 50 <container>
```

---

## Step 3: Enter the Container

For running containers:

```bash
docker exec -it <container> /bin/sh
```

If bash exists:

```bash
docker exec -it <container> /bin/bash
```

Inside the container, check:

```bash
ps aux
env
ls -la
ss -tulpn
curl localhost:<port>
```

---

## Step 4: Override Entrypoint

If the container exits immediately, start a shell instead of the normal entrypoint:

```bash
docker run -it --entrypoint sh <image>
```

or:

```bash
docker run -it --entrypoint /bin/bash <image>
```

This lets me inspect the image before the normal startup command runs.

---

## Step 5: Check Events, Logs, and Processes

Docker events:

```bash
docker events --filter container=<container> --since 1h
```

Container logs:

```bash
docker logs --tail 50 <container>
```

Process list inside container:

```bash
docker top <container>
```

---

## Step 6: Check CPU and Memory

Live stats for all containers:

```bash
docker stats
```

Stats for one container:

```bash
docker stats <container>
```

Processes inside a container:

```bash
docker top <container>
```

---

## Step 7: Check Network and Mounts

List Docker networks:

```bash
docker network ls
```

Show container published ports:

```bash
docker port <container>
```

Inspect networks:

```bash
docker inspect <container> | jq '.[0].NetworkSettings.Networks'
```

Inspect mounts:

```bash
docker inspect <container> | jq '.[0].Mounts'
```

Test from inside container:

```bash
docker exec <container> sh -c 'wget -qO- localhost:<port> || curl -s localhost:<port>'
```

---

# Quick Troubleshooting Matrix

| Symptom | First Check | Useful Command |
|---|---|---|
| Docker daemon error | Is Docker running? | `sudo systemctl status docker` |
| Permission denied socket | Can sudo access Docker? | `sudo docker ps` |
| Container exited | Why did PID 1 stop? | `docker logs <container>` |
| Port already in use | Who owns the port? | `ss -tulpn \| grep :PORT` |
| Published port not working | Is port mapped correctly? | `docker port <container>` |
| Volume permission denied | Who owns host path? | `ls -la /host/path` |
| Image pull fails | DNS/auth/rate limit? | `docker pull <image>` |
| No disk space | Docker disk usage? | `docker system df` |
| OOM killed | Was container memory killed? | `docker inspect <container>` |
| Compose unhealthy | Logs and healthcheck | `docker compose logs <service>` |

---

# My Docker Troubleshooting Rule

For every Docker issue, I will follow this flow:

```text
What is the symptom?
↓
What Docker layer must work?
↓
How do I verify that layer?
↓
What is the first failed check?
↓
What is the smallest safe fix?
↓
How do I prove it is fixed?
```

The goal is not just to fix containers.

The goal is to become calm, structured, and evidence-based when something breaks.
