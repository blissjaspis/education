# Private Docker Registry on DigitalOcean

This guide provides a comprehensive walkthrough for setting up a secure, private Docker registry on a DigitalOcean VPS. We'll cover security, adding a web UI, and integrating with a CI/CD pipeline.

## Table of Contents
- [Introduction](#introduction)
- [Prerequisites](#prerequisites)
- [1. Setting up the Docker Registry](#1-setting-up-the-docker-registry)
- [2. Securing the Registry](#2-securing-the-registry)
- [3. Adding a Web UI](#3-adding-a-web-ui)
- [4. CI/CD Integration](#4-ci-cd-integration)
- [5. Security Considerations on DigitalOcean](#5-security-considerations-on-digitalocean)
- [Conclusion](#conclusion)

## Introduction

A private Docker registry is a storage system for your Docker images, hosted on your own infrastructure. It's essential when you want to keep your images private and not publish them to the public Docker Hub. Using a private registry gives you control over where your images are stored, reduces reliance on third-party services, and can improve security and performance.

This guide will use a DigitalOcean Droplet as the host, but the principles apply to any VPS provider or on-premises server.

## Prerequisites

- A DigitalOcean account.
- A registered domain name (e.g., `your-domain.com`). This is highly recommended for setting up a secure registry with TLS.
- A DigitalOcean Droplet (VPS) running a modern Linux distribution like Ubuntu 22.04.
- Docker and Docker Compose installed on your Droplet.
- DNS `A` record pointing a subdomain (e.g., `registry.your-domain.com`) to your Droplet's IP address.

## 1. Setting up the Docker Registry

First, let's get a basic registry running. The official Docker Registry is available as a Docker image.

1.  **SSH into your Droplet:**
    ```bash
    ssh root@your_droplet_ip
    ```

2.  **Run the registry container:**
    ```bash
    docker run -d -p 5000:5000 --restart=always --name registry registry:2
    ```
    This command starts a registry container named `registry` listening on port 5000.

At this point, you have a *local, insecure* registry. You can push and pull images from the Droplet itself, but you won't be able to connect from other machines without configuring Docker to trust this insecure registry, which is not recommended for production.

## 2. Securing the Registry

A production-ready registry must be secured. We'll use a reverse proxy (Nginx) to handle HTTPS (TLS) and basic authentication.

We will use `docker-compose` to manage our services.

1.  **Create a directory for your registry setup:**
    ```bash
    mkdir docker-registry && cd docker-registry
    ```

2.  **Create a directory for Nginx configuration:**
    ```bash
    mkdir -p nginx/conf.d
    ```

3.  **Create an Nginx configuration file:**
    Create a file named `nginx/conf.d/registry.conf` with the following content. Replace `registry.your-domain.com` with your actual subdomain.

    ```nginx
    upstream docker-registry {
      server registry:5000;
    }

    server {
      listen 80;
      server_name registry.your-domain.com;

      location / {
        return 301 https://$host$request_uri;
      }
    }

    server {
      listen 443 ssl http2;
      server_name registry.your-domain.com;

      ssl_certificate /etc/letsencrypt/live/registry.your-domain.com/fullchain.pem;
      ssl_certificate_key /etc/letsencrypt/live/registry.your-domain.com/privkey.pem;

      # SSL settings
      ssl_protocols TLSv1.2 TLSv1.3;
      ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384';
      ssl_prefer_server_ciphers off;

      client_max_body_size 0; # Allow large image uploads

      location / {
        # Required for authentication
        auth_basic "Docker Registry";
        auth_basic_user_file /etc/nginx/conf.d/registry.password;

        proxy_pass http://docker-registry;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto "https";
        proxy_read_timeout 900;
      }
    }
    ```

4.  **Set up Basic Authentication:**
    We need `htpasswd` to create the password file.
    ```bash
    sudo apt-get update && sudo apt-get install -y apache2-utils
    ```
    Create a user and password. Replace `your_user` with your desired username.
    ```bash
    htpasswd -c nginx/conf.d/registry.password your_user
    ```

5.  **Use Certbot to get a Let's Encrypt Certificate:**
    Install Certbot and get a certificate. We use the `--standalone` mode on a temporary server. Make sure port 80 is not in use.
    ```bash
    sudo certbot certonly --standalone -d registry.your-domain.com
    ```
    This will create certificates in `/etc/letsencrypt/live/registry.your-domain.com/`.

6.  **Create the `docker-compose.yml` file:**
    Create a `docker-compose.yml` in your `docker-registry` directory.

    ```yaml
    version: '3'

    services:
      registry:
        image: registry:2
        container_name: private-registry
        restart: always
        volumes:
          - registry-data:/var/lib/registry
        networks:
          - registry-net

      nginx-proxy:
        image: nginx:latest
        container_name: registry-nginx-proxy
        restart: always
        ports:
          - "80:80"
          - "443:443"
        volumes:
          - ./nginx/conf.d:/etc/nginx/conf.d
          - /etc/letsencrypt:/etc/letsencrypt
        networks:
          - registry-net
        depends_on:
          - registry

    networks:
      registry-net:

    volumes:
      registry-data:
    ```

7.  **Start the services:**
    ```bash
    docker-compose up -d
    ```

Now you have a secure, private Docker registry accessible at `https://registry.your-domain.com`.

**To use it:**

```bash
# Login
docker login registry.your-domain.com

# Tag an image
docker tag my-image:latest registry.your-domain.com/my-image:latest

# Push the image
docker push registry.your-domain.com/my-image:latest

# Pull the image from another machine
docker pull registry.your-domain.com/my-image:latest
```

## 3. Adding a Web UI

While the registry is functional, a web UI makes it easier to browse and manage images. The user mentioned `komo.do`, which seems to be less known. A popular and simple choice is `joxit/docker-registry-ui`.

Let's update our `docker-compose.yml` to include the UI.

1.  **Modify `docker-compose.yml`:**

    ```yaml
    version: '3.7'

    services:
      registry:
        image: registry:2
        container_name: private-registry
        restart: always
        volumes:
          - registry-data:/var/lib/registry
        environment:
          - REGISTRY_DELETE_ENABLED=true # Enable image deletion in the UI
        networks:
          - registry-net

      registry-ui:
        image: joxit/docker-registry-ui:latest
        container_name: registry-ui
        restart: always
        ports:
          - "8080:80"
        environment:
          - REGISTRY_URL=http://registry:5000
          - REGISTRY_TITLE=My Private Registry
          - DELETE_IMAGES=true
          - REGISTRY_SECURED=true
          - REGISTRY_USER=your_user # Your htpasswd username
          - REGISTRY_PASSWORD=your_password # Your htpasswd password
        networks:
          - registry-net
        depends_on:
          - registry

      nginx-proxy:
        image: nginx:latest
        container_name: registry-nginx-proxy
        restart: always
        ports:
          - "80:80"
          - "443:443"
        volumes:
          - ./nginx/conf.d:/etc/nginx/conf.d
          - /etc/letsencrypt:/etc/letsencrypt
        networks:
          - registry-net
        depends_on:
          - registry

    networks:
      registry-net:

    volumes:
      registry-data:
    ```
    **Note:** The UI now connects directly to the registry container over the internal Docker network. We also expose the UI on port 8080. You can access it via `http://your_droplet_ip:8080`. For better security, you could configure Nginx to proxy the UI as well, perhaps on a different subdomain.

2.  **Restart the services:**
    ```bash
    docker-compose up -d
    ```

You can now visit `http://your_droplet_ip:8080` to see your registry's UI.

## 4. CI/CD Integration

You can integrate your private registry into your CI/CD pipeline. Here's an example using **GitHub Actions**.

1.  **Add your registry credentials to GitHub Secrets:**
    In your GitHub repository, go to `Settings` > `Secrets and variables` > `Actions` and add the following secrets:
    *   `DOCKER_REGISTRY`: `registry.your-domain.com`
    *   `DOCKER_USERNAME`: `your_user`
    *   `DOCKER_PASSWORD`: The password you created with `htpasswd`.

2.  **Create a GitHub Actions workflow:**
    Create a file `.github/workflows/docker-publish.yml` in your project:

    ```yaml
    name: Publish Docker Image

    on:
      push:
        branches: [ main ]

    jobs:
      build-and-push:
        runs-on: ubuntu-latest
        steps:
          - name: Checkout repository
            uses: actions/checkout@v3

          - name: Log in to the Container registry
            uses: docker/login-action@v2
            with:
              registry: ${{ secrets.DOCKER_REGISTRY }}
              username: ${{ secrets.DOCKER_USERNAME }}
              password: ${{ secrets.DOCKER_PASSWORD }}

          - name: Build and push Docker image
            uses: docker/build-push-action@v4
            with:
              context: .
              push: true
              tags: ${{ secrets.DOCKER_REGISTRY }}/my-app:${{ github.sha }}
    ```
    This workflow will trigger on every push to the `main` branch, log in to your private registry, build the Docker image from your `Dockerfile`, and push it, tagged with the Git commit SHA.

## 5. Security Considerations on DigitalOcean

-   **Firewall:** Use DigitalOcean Cloud Firewalls to restrict access to your registry.
    -   Allow port `443` (for the registry) and `80` (for HTTP to HTTPS redirection) from anywhere.
    -   If you expose the UI directly, you might want to restrict port `8080` to your company's IP addresses.
    -   Allow port `22` (SSH) only from your trusted IP addresses.

-   **Backups:** The Docker volume `registry-data` contains all your images. Regularly back up this volume to prevent data loss. You can use DigitalOcean Snapshots or other backup solutions.

-   **Monitoring:** Monitor your Droplet's resource usage (CPU, memory, disk space) to ensure your registry remains performant.

## Conclusion

You now have a fully functional, secure, and private Docker Registry on DigitalOcean, complete with a web UI and CI/CD integration. This setup gives you full control over your Docker images, enhancing security and streamlining your development workflow. Remember to keep your server and Docker images updated to protect against vulnerabilities.
