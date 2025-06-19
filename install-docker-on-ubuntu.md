# How to Install and Use Docker on Ubuntu 22.04 for Production

This guide provides a comprehensive, step-by-by-step tutorial for installing Docker Community Edition (CE) on Ubuntu 22.04, from initial setup to production-ready best practices.

## Prerequisites

- An Ubuntu 22.04 server (64-bit).
- A non-root user with `sudo` privileges.
- An account on [Docker Hub](https://hub.docker.com/) if you want to push your own images.

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
# ... (your base image, e.g., FROM ubuntu:22.04)

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

-   **Limit Resources**: Configure memory and CPU limits for your containers to prevent them from consuming too many resources on the host.
-   **Configure Logging**: By default, Docker uses the `json-file` logging driver, which can consume a lot of disk space. For production, configure a log rotation or use a different logging driver like `syslog` or send logs to a centralized logging solution.
-   **Use Docker Content Trust (DCT)**: DCT provides cryptographic signing and verification of Docker images.
-   **Scan Images for Vulnerabilities**: Use tools like Docker Scout or other third-party scanners to check your images for security vulnerabilities.
-   **Manage Secrets Securely**: Use Docker secrets to manage sensitive data like passwords and API keys.

## Conclusion

You now have a production-ready Docker environment on your Ubuntu 22.04 server. You can build, ship, and run your applications using containers. For more advanced topics, refer to the official Docker documentation.
