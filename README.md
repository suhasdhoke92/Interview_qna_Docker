# Interview_qna_Docker

This repository provides a comprehensive guide to Docker, a platform for containerizing applications. It covers common Docker troubleshooting scenarios, key Dockerfile instructions, best practices for managing containers, and day-to-day operations.

## Docker Container Exits Immediately After Starting

### Issue
A Docker container exits immediately after starting, often with an exit code indicating failure or completion.

### Causes
- **Short-Lived Process**: Containers run a single process defined by `CMD` or `ENTRYPOINT`. If the command completes quickly (e.g., `echo "Hello"`), the container exits.
- **Command Failure**: The command specified in `CMD` or `ENTRYPOINT` fails, causing the container to exit.
- **Non-Foreground Process**: Some applications (e.g., `nginx`) may run as a daemon unless configured to stay in the foreground.

### Troubleshooting
1. **Check Logs**: Use `docker logs <container_id>` to inspect errors or output from the command.
2. **Verify Command**: Ensure the `CMD` or `ENTRYPOINT` is a long-running process. For example:
   ```dockerfile
   CMD ["nginx", "-g", "daemon off;"]
   ```
   This keeps NGINX running in the foreground, preventing the container from exiting.
3. **Interactive Mode**: Run the container with an interactive shell to debug:
   ```bash
   docker run -it <image_name> /bin/bash
   ```
   Inspect the running command or environment inside the container.

### Fix
- Modify the Dockerfile to use a long-running command (e.g., `CMD ["nginx", "-g", "daemon off;"]`).
- If the command failed, fix the underlying issue (e.g., missing dependencies, incorrect configuration).
- For temporary debugging, override the entrypoint with an interactive shell:
   ```bash
   docker run -it --entrypoint /bin/bash <image_name>
   ```

## Purpose of EXPOSE in Dockerfile

- **Purpose**: The `EXPOSE` instruction in a Dockerfile documents the ports an application inside the container listens on (e.g., `EXPOSE 80` for a web server).
- **Not a Publishing Mechanism**: It does not automatically publish ports to the host. Publishing requires `docker run -p <host_port>:<container_port>` (e.g., `docker run -p 80:80`).
- **Use Cases**:
  - **Documentation**: Informs developers about the ports the application uses.
  - **Docker Compose**: Helps Docker Compose understand which ports to map or link between containers (e.g., backend on port 8080, database on port 3306).
- Example:
  ```dockerfile
  EXPOSE 80
  ```
  This indicates the application listens on port 80, but you still need:
  ```bash
  docker run -p 80:80 <image_name>
  ```

## Port Not Accessible After Port Mapping

### Issue
You run a container with port mapping (e.g., `docker run -p 80:80`), but accessing `localhost:<port>` in a browser or API client fails or times out.

### Troubleshooting
1. **Incorrect Port Mapping**: Verify the port mapping. For example, if the app listens on port 8080 inside the container, use:
   ```bash
   docker run -p 80:8080 <image_name>
   ```
2. **Port Conflict**: Check if the host port is already in use:
   ```bash
   netstat -tuln | grep 80
   lsof -i :80
   ```
   If occupied, use a different host port (e.g., `docker run -p 1100:8080`).
3. **Firewall**: Ensure no firewall is blocking the port on the host (e.g., `ufw` or cloud security groups).
4. **Application Binding**: The app might be binding to `127.0.0.1` (localhost inside the container), which restricts access to internal loopback. Update the app to bind to `0.0.0.0`:
   ```javascript
   // From
   app.listen(80, '127.0.0.1');
   // To
   app.listen(80, '0.0.0.0');
   ```
5. **Inspect Container**: Use `docker logs <container_id>` or `docker inspect <container_id>` to check for errors or misconfigurations.

### Fix
- Correct the port mapping in the `docker run` command.
- Free up the host port or use an alternative port.
- Adjust firewall rules or security group settings.
- Update the application to bind to `0.0.0.0`.

## Data Lost When Container Stops and Restarts

### Issue
Data stored in a container is lost when it stops or restarts because containers are ephemeral by default.

### Solution
Use **Docker volumes** or **bind mounts** for persistent storage:

1. **Docker Volume**:
   Create a volume:
   ```bash
   docker volume create mydata
   ```
   Run the container with the volume:
   ```bash
   docker run -v mydata:/app/data mysql
   ```
   The volume persists data in `/app/data` across container restarts.

2. **Bind Mount**:
   Mount a host directory to the container:
   ```bash
   docker run -v /host/path:/app/data mysql
   ```
   Data is stored on the host filesystem, ensuring persistence.

### Use Case
For databases (e.g., MySQL) or applications requiring persistent data, use volumes to retain data across container lifecycles.

## Changes Not Reflected After Rebuilding Docker Image

### Issue
You modified code, rebuilt the Docker image, but the changes are not reflected in the running container.

### Cause
Docker uses **layer caching** to speed up builds. If the Dockerfile or context hasn’t changed significantly, Docker reuses cached layers, ignoring updates.

### Troubleshooting
1. Verify the Dockerfile and code changes are correct.
2. Check `docker history <image_name>` to confirm if new layers were created.
3. Ensure the build context includes updated files (e.g., `docker build .`).

### Fix
- **Disable Cache**: Rebuild the image without using the cache:
  ```bash
  docker build --no-cache -t <image_name> .
  ```
- **Clear Cache**: If caching persists, clear unused images and caches:
  ```bash
  docker system prune
  ```
- **Update Dockerfile**: Ensure the changed files are copied or included correctly in the build context.

### Prevention
- Use specific file paths in `COPY` or `ADD` to avoid unnecessary caching.
- Leverage `.dockerignore` to exclude irrelevant files, reducing cache-related issues.

## App Crashes with "Permission Denied" in Container

### Issue
An application works locally but fails with a "permission denied" error inside a Docker container.

### Causes
- **User Permissions**: Locally, the app may run with elevated (e.g., root) privileges, but in the container, it runs as a non-root user (e.g., `appuser`) without sufficient permissions.
- **File Access**: The application or user lacks executable permissions for files or directories.

### Troubleshooting
1. Check the user in the Dockerfile (e.g., `USER appuser`).
2. Verify file permissions using `docker exec -it <container_id> /bin/bash` and `ls -l`.
3. Inspect logs for specific errors:
   ```bash
   docker logs <container_id>
   ```

### Fix
- **Add User in Dockerfile**: Create a non-root user with appropriate permissions:
  ```dockerfile
  RUN useradd -ms /bin/bash appuser
  USER appuser
  ```
- **Set File Permissions**: Ensure the application or scripts have executable permissions:
  ```dockerfile
  RUN chmod +x /app/app.py
  ```
- **Adjust Ownership**: If needed, change ownership of directories:
  ```dockerfile
  RUN chown -R appuser:appuser /app
  ```

## Docker Host Running Out of Disk Space

### Issue
The Docker host is running out of disk space due to accumulated images, containers, or volumes.

### Troubleshooting
1. Check disk usage:
   ```bash
   docker system df
   ```
2. Identify unused images, containers, or volumes.

### Fix
- **Remove Dangling Images**: Delete images not used by any containers:
  ```bash
  docker system prune
  ```
- **Remove All Unused Resources**: Include unused images (not just dangling ones):
  ```bash
  docker system prune -a
  ```
- **Remove Unused Volumes**: Free up space used by unattached volumes:
  ```bash
  docker volume prune
  ```
- **Verify Cleanup**: Recheck disk usage:
  ```bash
  docker system df
  ```

### Note
Volumes often consume significant space, especially for databases. Always ensure volumes are not critical before pruning.

## Debugging a Live Container

### Correct Approach
To debug a running container, access its shell:
```bash
docker exec -it <container_id> /bin/bash
```
or, if `/bin/bash` is unavailable:
```bash
docker exec -it <container_id> /bin/sh
```

### Common Mistake
Using `docker run -it` is incorrect here, as it starts a new container instead of accessing the running one.

## Container Registry Used in Organizations

- **Preferred Registries**: Enterprises typically avoid public Docker Hub due to security concerns. Common registries include:
  - Amazon Elastic Container Registry (ECR)
  - Azure Container Registry (ACR)
  - Quay.io (managed by Red Hat)
  - GitHub Container Registry (ghcr.io)
  - JFrog Artifactory
- **Reason**: These provide better security, access control, and integration with CI/CD pipelines.

## Difference Between CMD and ENTRYPOINT in Dockerfile

- **CMD**: Specifies the default command to run when a container starts. It can be overridden by arguments in `docker run`.
- **ENTRYPOINT**: Defines the main executable, which is harder to override unless explicitly specified with `--entrypoint`.

### Example
Dockerfile with `ENTRYPOINT`:
```dockerfile
ENTRYPOINT ["echo", "Hello"]
```
Dockerfile with `CMD`:
```dockerfile
CMD ["echo", "Hello"]
```

- Build and run:
  ```bash
  docker build -t demo-ep ep/
  docker run demo-ep
  ```
  Both print `Hello`.

- Override behavior:
  ```bash
  docker run demo-ep world
  ```
  - **ENTRYPOINT**: Outputs `Hello world` (appends `world` to `echo Hello`).
  - **CMD**: Fails, as it tries to run `world` as a command (not executable).

- Correct CMD override:
  ```bash
  docker run demo-cmd echo world
  ```
  Outputs `world`.

- Override ENTRYPOINT:
  ```bash
  docker run --entrypoint /bin/echo demo-ep world
  ```
  Outputs `world`.

### Key Difference
- `CMD` is fully replaced by `docker run` arguments.
- `ENTRYPOINT` appends arguments unless overridden with `--entrypoint`.

## Common Docker Commands Used Daily

- **Build Image**: Create an image from a Dockerfile:
  ```bash
  docker build -t <image_name> .
  ```
- **Run Container**: Start a container from an image:
  ```bash
  docker run <image_name>
  ```
- **List Containers**: View running containers:
  ```bash
  docker ps
  ```
  Include stopped containers:
  ```bash
  docker ps -a
  ```
- **List Images**: View available images on the host:
  ```bash
  docker images
  ```
- **View Logs**: Check container logs:
  ```bash
  docker logs <container_id>
  ```
- **Clean Up**: Remove unused resources:
  ```bash
  docker system prune
  ```

## Forcefully Removing a Container

### When to Force Remove
- Container is stuck, unresponsive, or restarting unexpectedly (e.g., during CI/CD pipelines).
- Need to clear a container that’s causing issues.

### How to Force Remove
1. Find the container ID:
   ```bash
   docker ps -a
   ```
2. Force remove the container:
   ```bash
   docker rm -f <container_id>
   ```

## Conclusion
Docker simplifies application deployment through containerization, but understanding its behavior is key to resolving issues like container exits, port mapping failures, data loss, permission errors, and disk space management. By using proper commands, Dockerfile instructions, and secure registries, you can ensure reliable and efficient container workflows.
