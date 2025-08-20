# How to Install and Use Docker on Ubuntu 24.04 LTS for Production

This guide provides a comprehensive, step-by-step tutorial for installing Docker Community Edition (CE) on Ubuntu 24.04 LTS, from initial setup to production-ready best practices.

## Prerequisites

- An Ubuntu 24.04 LTS server (64-bit).
- A non-root user with `sudo` privileges.
- An account on [Docker Hub](https://hub.docker.com/) if you want to push your own images.

> **Note for Ubuntu 24.04 LTS**: This version includes enhanced security features such as restricted unprivileged user namespaces by default. Docker installation and operation remain unaffected by these changes when following this guide.

## Step 1: Uninstall Old Docker Versions

Before installing, it's best to remove any older or unofficial Docker packages.

```bash
sudo apt-get remove docker docker-engine docker.io containerd runc
```

It's okay if `apt-get` reports that none of these packages are installed.

## Step 2: Set Up Docker's APT Repository

We will install Docker from the official Docker repository to ensure we get the latest version.

### 1. Update the `apt` package index and install prerequisites

```bash
sudo apt-get update
sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```

### 2. Add Docker's official GPG key

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

### 3. Set up the repository

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

## Step 3: Install Docker Engine

Now we can install the latest version of Docker Engine and containerd.

### 1. Update the `apt` package index

```bash
sudo apt-get update
```

### 2. Install Docker Engine, CLI, containerd, and plugins

```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### 3. Verify the installation

Check that the Docker service is running and enabled to start at boot.

```bash
sudo systemctl status docker
```

You should see that the service is `active (running)`.

Then, verify that the installation is successful by running the `hello-world` image.

```bash
sudo docker run hello-world
```

This command downloads a test image and runs it in a container. If it runs correctly, it will print a confirmation message.

## Step 4: Post-installation Steps for a Production Environment

These steps are crucial for a production setup.

### 1. Manage Docker as a non-root user

To run `docker` commands without `sudo`, you need to add your user to the `docker` group.

```bash
sudo groupadd docker
sudo usermod -aG docker ${USER}
```

You will need to log out and log back in for this to take effect, or you can activate the changes for the group immediately with:

```bash
newgrp docker
```

### 2. Configure Docker to start on boot

The Docker service should already be configured to start on boot, but you can ensure this is the case with `systemd`.

```bash
sudo systemctl enable docker.service
sudo systemctl enable containerd.service
```

## Step 5: Using Docker

Here are some basic commands to get you started.

-   **List images**: `docker images`
-   **List running containers**: `docker ps`
-   **List all containers (running and stopped)**: `docker ps -a`
-   **Run a container**: `docker run <image_name>`
-   **Stop a container**: `docker stop <container_id_or_name>`
-   **Remove a container**: `docker rm <container_id_or_name>`
-   **Pull an image from Docker Hub**: `docker pull <image_name>`

## Step 6: Using Docker Compose

Docker Compose is a tool for defining and running multi-container Docker applications. It's included in the `docker-compose-plugin`.

You can create a `docker-compose.yml` file to configure your application's services and then use `docker compose up` to start them.

## Step 7: Production Best Practices & Security

For a production environment, consider the following:

-   **Keep Docker Updated**: Always use the latest version of Docker Engine.

### Use a Non-Root User Inside Containers

By default, processes inside a Docker container run as the `root` user. This poses a security risk because if an attacker compromises your application, they could potentially gain `root` access on the container. This could allow them to perform malicious actions, and potentially find a way to escalate their privileges to the host machine.

To mitigate this, you should always run your container processes as a non-root user. This is known as the principle of least privilege.

Here's how you can do it in your `Dockerfile`:

1.  **Create a dedicated user and group**: You can use `RUN` commands to add a user and a group. It's a good practice to use static UID (User ID) and GID (Group ID).
2.  **Set ownership of application files**: When you `COPY` your application files into the image, ensure the new user owns them. The `--chown` flag for the `COPY` instruction is perfect for this.
3.  **Switch to the new user**: The `USER` instruction sets the user name (or UID) to use when running the image.

**Example `Dockerfile` implementation:**

```Dockerfile
# ... (your base image, e.g., FROM ubuntu:24.04)

# Create a user and group
RUN groupadd -g 1000 myapp && \
    useradd -u 1000 -g myapp -m -s /bin/bash myapp

# Copy application files with correct ownership
COPY --chown=myapp:myapp . /app

# Switch to the non-root user
USER myapp

# Now, any subsequent commands (like CMD or ENTRYPOINT)
# will be run as the 'myapp' user.
CMD ["./start-app.sh"]
```

### Use Named Volumes Instead of Bind Mounts in Production

While bind mounts are convenient for development (allowing you to edit code on the host and see changes live), they are not recommended for production. They create a tight coupling between the container and the host's filesystem, which harms portability and can introduce security risks.

**The best practice for production is to use Docker-named volumes.**

-   **For Application Code**: The code should be baked into the image using a `COPY` instruction in your `Dockerfile`. This makes your deployment a self-contained, versioned artifact. If multiple containers need access to the same code (like an Nginx container needing access to static files from a PHP application), they can share a named volume.
-   **For Persistent Data**: Data that needs to persist beyond a container's lifecycle (like databases, user uploads, or logs) should always be stored in a named volume.

**Example `docker-compose.yml` for a Laravel App:**

This example shows how an Nginx and PHP-FPM service can share application code and persistent storage using named volumes.

```yaml
version: '3.8'

services:
  app:
    build:
      context: . # Assumes Dockerfile is in the same directory
      dockerfile: Dockerfile # Your PHP Dockerfile
    image: my-laravel-app
    volumes:
      # Mounts the entire app code to a named volume
      - app-code:/var/www/html
      # Mounts the persistent storage directory to another named volume
      - laravel-storage:/var/www/html/storage
    # ... other app configurations

  nginx:
    image: nginx:stable-alpine
    ports:
      - "80:80"
    volumes:
      # Mounts the shared code volume (read-only is safer)
      - app-code:/var/www/html:ro
      # Mount a custom Nginx configuration
      - ./nginx.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - app

volumes:
  app-code:
  laravel-storage:
```

In this setup:
- The `app` service builds the image, `COPY`ing the code in. On its first run, it populates the `app-code` and `laravel-storage` volumes.
- The `nginx` service gets read-only access to the application code to serve static files. This is the key purpose of the `app-code` volume: to share code between the `app` and `nginx` containers so Nginx can serve static files (like CSS and JS) directly and efficiently.
- The application's storage is safely persisted in the `laravel-storage` volume.

> **Note on Nested Volumes**: You might wonder why `laravel-storage` is defined separately when `app-code` already covers its children directory. This is a powerful Docker feature. By defining a more specific mount path, the `laravel-storage` volume "masks" the `storage/` directory inside the `app-code` volume. This separates your persistent data (user uploads, cache, logs) from your stateless application code, which is critical for safe deployments and easy backups.

### A Pragmatic Choice: When to Use Bind Mounts in Production

The advice to bake code into an image is the ideal best practice for scalable, portable applications. However, for a lone developer or a small team running multiple, similar sites on a single server, this can be inefficient.

In this scenario, a hybrid approach is often the most practical choice:

1.  **Build a Generic Base Image**: Create one clean, secure base image with your required dependencies (like PHP and its extensions).
2.  **Use Bind Mounts for Code**: For each website, use the generic base image but bind mount its unique source code from the host (e.g., from `/var/www/site1.com`).

**Why is this a good trade-off?**
- **Resource Efficiency**: You have only one base image shared by all your sites, saving significant disk space.
- **Deployment Speed**: To update a site, you just `git pull` the changes on the host and restart the container. There is no need to rebuild the image.

With this method, you make a conscious decision to trade portability for efficiency, which is a perfectly valid engineering choice for this specific context.

**Crucially, you should still use named volumes for any truly persistent data, like database files or Laravel's `storage` directory.** The compromise is only on the stateless application code.

-   **Limit Resources**: Configure memory and CPU limits for your containers to prevent them from consuming too many resources on the host.
-   **Configure Logging**: By default, Docker uses the `json-file` logging driver, which can consume a lot of disk space. For production, configure a log rotation or use a different logging driver like `syslog` or send logs to a centralized logging solution.
-   **Use Docker Content Trust (DCT)**: DCT provides cryptographic signing and verification of Docker images.
-   **Scan Images for Vulnerabilities**: Use tools like Docker Scout or other third-party scanners to check your images for security vulnerabilities.
-   **Manage Secrets Securely**: Use Docker secrets to manage sensitive data like passwords and API keys.

## Conclusion

You now have a production-ready Docker environment on your Ubuntu 24.04 LTS server. You can build, ship, and run your applications using containers. For more advanced topics, refer to the official Docker documentation.
