# Salta: Docker Container Won't Start

## Scenario

The SadServers lab **"Salta"** presents a Docker troubleshooting problem.

There is a dockerized Node.js web application located in:

```text
/home/admin/app
```

The goal is to create a Docker container so the web application is reachable on port `8888`.

The validation command is:

```bash
curl localhost:8888
```

Expected result:

```text
Hello World!
```

The lab also requires that there should be only one running Docker container.

## Requirement

Create and run a Docker container for the Node.js application.

The final state must be:

```text
Application directory: /home/admin/app
Host port: 8888
Expected response: Hello World!
Running Docker containers: only one
```

## Initial State

The application directory existed and contained Docker-related files:

```text
/home/admin/app
├── Dockerfile
├── package.json
├── package-lock.json
└── server.js
```

Docker was installed, but normal user access to Docker returned a permission error:

```text
permission denied while trying to connect to the Docker daemon socket
```

Using `sudo docker` worked.

There was also one stopped container from a previous failed run:

```text
Status: Exited (1)
```

## Troubleshooting Path

```text
Connect to the SadServers instance
↓
Verify Docker access
↓
Inspect existing containers
↓
Check the application directory
↓
Inspect the failed container logs
↓
Find why the container exited
↓
Fix the Dockerfile startup command
↓
Rebuild the Docker image
↓
Run the container with port 8888 published
↓
Identify host port conflict
↓
Stop the conflicting host service
↓
Run the corrected container
↓
Validate curl localhost:8888
```

## Verification Before Fix

Check current user and host:

```bash
whoami
hostname
```

Check Docker version:

```bash
docker --version
```

Check running containers:

```bash
docker ps
```

Because normal Docker access failed, test with sudo:

```bash
sudo docker ps
```

Check all containers, including stopped containers:

```bash
sudo docker ps -a
```

Check the application directory:

```bash
ls -lah /home/admin/app
```

## First Finding

There was a stopped container:

```text
Status: Exited (1)
```

This means the container was created before, but the application process failed during startup.

## Inspect the Failed Container

Check container logs:

```bash
sudo docker logs <container_id>
```

The logs showed:

```text
Error: Cannot find module '/usr/src/app/serve.js'
```

This means Node.js was trying to run:

```text
serve.js
```

but that file did not exist.

## Inspect the Dockerfile

View the Dockerfile:

```bash
cat /home/admin/app/Dockerfile
```

The Dockerfile contained:

```dockerfile
CMD [ "node", "serve.js" ]
```

This was incorrect because the application file was named:

```text
server.js
```

## Inspect the Application Files

Check `package.json`:

```bash
cat /home/admin/app/package.json
```

The start script showed:

```json
"start": "node server.js"
```

Check the actual application file:

```bash
cat /home/admin/app/server.js
```

The app listens on port `8888` and returns:

```text
Hello World!
```

## First Failure

The Dockerfile startup command was wrong.

It tried to start:

```text
serve.js
```

but the correct file was:

```text
server.js
```

## Fix 1: Correct the Dockerfile

Edit the Dockerfile:

```bash
nano /home/admin/app/Dockerfile
```

Change this:

```dockerfile
CMD [ "node", "serve.js" ]
```

To this:

```dockerfile
CMD [ "node", "server.js" ]
```

## Fix 2: Remove the Old Failed Container

Remove the old exited container:

```bash
sudo docker rm <container_id>
```

## Fix 3: Rebuild the Image

Move into the application directory:

```bash
cd /home/admin/app
```

Build the corrected image:

```bash
sudo docker build -t salta-app .
```

The image built successfully.

## Next Finding: Port Already in Use

When trying to run the container:

```bash
sudo docker run -d --name salta-app -p 8888:8888 salta-app
```

Docker returned:

```text
bind: address already in use
```

This means host port `8888` was already occupied.

## Verify Who Owns Port 8888

Check the listening process:

```bash
sudo ss -tulpn | grep :8888
```

The output showed that host-level `nginx` was listening on port `8888`.

## Second Failure

Host `nginx` was already using port `8888`, so Docker could not publish the required host port.

## Fix 4: Stop the Conflicting Host Service

Stop nginx:

```bash
sudo systemctl stop nginx
```

Verify port `8888` is free:

```bash
sudo ss -tulpn | grep :8888
```

Expected result:

```text
No output
```

## Fix 5: Remove Created Container and Run Again

A failed container named `salta-app` may exist in `Created` state.

Remove it:

```bash
sudo docker rm salta-app
```

Run the corrected container:

```bash
sudo docker run -d --name salta-app -p 8888:8888 salta-app
```

## Validation

Check running containers:

```bash
sudo docker ps
```

Expected result:

```text
Only one running container should exist.
```

Test the application:

```bash
curl localhost:8888
```

Expected result:

```text
Hello World!
```

The SadServers validation passed successfully.

## Final Result

The Docker container was fixed and started successfully.

The Node.js application responded correctly on port `8888`.

The final validation returned:

```text
Hello World!
```

## Lessons Learned

- A container that exits immediately usually means its main process failed.
- `docker logs <container>` is one of the first commands to check when a container exits.
- Exit status `1` usually means the application crashed or failed to start.
- Dockerfile `CMD` must point to the correct startup file or command.
- `package.json` can help confirm how a Node.js app is supposed to start.
- A successful image build does not guarantee the container will run successfully.
- `bind: address already in use` means the host port is already occupied.
- `ss -tulpn` helps identify which process owns a port.
- A running host service can block Docker from publishing a required port.
- Always validate with the same command used by the lab test.

## Knowledge Check

### Question 1

The container exited with an error saying it could not find `/usr/src/app/serve.js`. What was the real problem?

A. Docker daemon was stopped  
B. The Dockerfile `CMD` pointed to the wrong JavaScript file  
C. Port `8888` was blocked by the firewall  
D. The image did not build  

**Answer:** B

### Question 2

Which command helped identify that port `8888` was already being used by host `nginx`?

A. `docker images`  
B. `docker logs`  
C. `sudo ss -tulpn | grep :8888`  
D. `cat package.json`  

**Answer:** C

### Question 3

Which command correctly runs the fixed application container on host port `8888`?

A. `sudo docker run -d --name salta-app -p 8888:8888 salta-app`  
B. `sudo docker run salta-app`  
C. `sudo docker build -p 8888:8888 salta-app`  
D. `sudo docker start nginx`  

**Answer:** A
