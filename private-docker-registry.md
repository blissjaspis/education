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
- [6. Consolidating on a Single Host](#6-consolidating-on-a-single-host)
- [Conclusion](#conclusion)

## Introduction

A private Docker registry is a storage system for your Docker images, hosted on your own infrastructure. It's essential when you want to keep your images private and not publish them to the public Docker Hub. Using a private registry gives you control over where your images are stored, reduces reliance on third-party services, and can improve security and performance.

This guide will use a DigitalOcean Droplet as the host, but the principles apply to any VPS provider or on-premises server.

## Prerequisites

- A DigitalOcean account.
- A registered domain name (e.g., `your-domain.com`). This is highly recommended for setting up a secure registry with TLS.
- A DigitalOcean Droplet (VPS) running a modern Linux distribution like Ubuntu 22.04.
- Docker and Docker Compose installed on your Droplet.
- DNS `A` record pointing a subdomain (e.g., `registry.your-domain.com`) to your Droplet's IP address. You will also need another subdomain for Komodo (e.g., `komodo.your-domain.com`) and domains for each website you plan to host.

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
    Create a file named `nginx/conf.d/registry.conf` with the following content. Replace `registry.your-domain.com`, `komodo.your-domain.com`, and your website domains with your actual domains.

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
      listen 80;
      server_name komodo.your-domain.com;

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

    server {
      listen 443 ssl http2;
      server_name komodo.your-domain.com;

      ssl_certificate /etc/letsencrypt/live/komodo.your-domain.com/fullchain.pem;
      ssl_certificate_key /etc/letsencrypt/live/komodo.your-domain.com/privkey.pem;

      # SSL settings
      ssl_protocols TLSv1.2 TLSv1.3;
      ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384';

      location / {
        proxy_pass http://komodo:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto "https";
        
        # Required for WebSocket support
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
      }
    }

    # --- Add server blocks for each of your websites below ---
    
    upstream website1 {
      server website1-container:8080; # Points to the service name and port in docker-compose
    }

    server {
      listen 80;
      server_name your-website-1.com;
      location / {
        return 301 https://$host$request_uri;
      }
    }

    server {
      listen 443 ssl http2;
      server_name your-website-1.com;

      ssl_certificate /etc/letsencrypt/live/your-website-1.com/fullchain.pem;
      ssl_certificate_key /etc/letsencrypt/live/your-website-1.com/privkey.pem;
      
      # SSL settings...

      location / {
        proxy_pass http://website1;
        proxy_set_header Host $host;
        # ... other proxy headers
      }
    }

    # ... Add similar blocks for website2, website3, etc.
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
    Install Certbot and get a certificate. You will need to do this for all the domains you are hosting.
    ```bash
    sudo certbot certonly --standalone -d registry.your-domain.com
    sudo certbot certonly --standalone -d komodo.your-domain.com
    sudo certbot certonly --standalone -d your-website-1.com
    # ... and so on for your other websites
    ```
    This will create certificates in `/etc/letsencrypt/live/`.

6.  **Create the `docker-compose.yml` file:**
    Create a `docker-compose.yml` in your `docker-registry` directory. This will now include the registry, our Nginx proxy, and the Komodo service.

    ```yaml
    version: '3.7'

    services:
      registry:
        image: registry:2
        container_name: private-registry
        restart: always
        volumes:
          - registry-data:/var/lib/registry
        networks:
          - registry-net

      komodo:
        image: ghcr.io/moghtech/komodo:latest
        container_name: komodo
        restart: always
        volumes:
          - komodo-data:/app/data
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
          - ./nginx/conf.d:/etc/nginx/conf.d:/etc/nginx/conf.d:ro
          - /etc/letsencrypt:/etc/letsencrypt:ro
        networks:
          - registry-net
        depends_on:
          - registry
          - komodo
          - website1 # Nginx should depend on the websites it proxies
          # - website2
          # ...

      # --- Add your website services below ---

      website1:
        image: registry.your-domain.com/my-website-1:latest
        container_name: website1-container
        restart: always
        networks:
          - registry-net
        # environment:
        #   - DB_HOST=...
        #   - API_URL=...

      # website2:
      #   image: registry.your-domain.com/my-website-2:latest
      #   container_name: website2-container
      #   restart: always
      #   networks:
      #     - registry-net

    networks:
      registry-net:

    volumes:
      registry-data:
      komodo-data:
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

## 3. Adding a Web UI: Komodo

A Web UI for your build and deployment system, referencing [Komodo](https://github.com/moghtech/komodo). Komodo is a powerful open-source tool to build and deploy software across many servers, which fits perfectly with our private Docker registry.

We have already integrated Komodo into our `docker-compose.yml` and exposed it securely via Nginx at `https://komodo.your-domain.com`.

### Initial Komodo Setup

1.  **Access the UI:** Navigate to `https://komodo.your-domain.com` in your browser.
2.  **Configuration:** Komodo's initial setup and configuration (like connecting to your servers, defining applications, and setting up deployment pipelines) are done through its web interface. You can consult the official [Komodo documentation](https://komo.do/) for detailed guidance on its features.
3.  **Connecting to the Private Registry:** Within Komodo, when you define a new application or service, you will specify the image source. Here, you will use the full path to your private registry, e.g., `registry.your-domain.com/my-app:latest`. Since Komodo is running on the same Docker network, it can pull images from the registry. You will need to configure credentials for your private registry within Komodo's settings.

## 4. CI/CD Integration with Komodo

Integrating Komodo provides a more advanced CI/CD workflow. The process now becomes:
1.  **CI (Continuous Integration):** Your CI pipeline (e.g., GitHub Actions) builds the Docker image and pushes it to your private registry.
2.  **CD (Continuous Deployment):** The CI pipeline then triggers Komodo to start the deployment.

Here's an updated GitHub Actions workflow that demonstrates this concept.

1.  **Add Komodo Secrets to GitHub:**
    In addition to your Docker registry credentials, add Komodo secrets:
    *   `KOMODO_URL`: `https://komodo.your-domain.com`
    *   `KOMODO_API_KEY`: An API key generated from the Komodo UI.

2.  **Updated GitHub Actions Workflow:**
    The workflow now includes a final step to call the Komodo API and trigger a deployment.

    ```yaml
    name: Build, Push, and Deploy

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
            id: build_and_push
            uses: docker/build-push-action@v4
            with:
              context: .
              push: true
              tags: ${{ secrets.DOCKER_REGISTRY }}/my-app:${{ github.sha }}

          - name: Trigger Komodo Deployment
            run: |
              curl -X POST \
                '${{ secrets.KOMODO_URL }}/api/v1/deploy' \
                -H 'Authorization: Bearer ${{ secrets.KOMODO_API_KEY }}' \
                -H 'Content-Type: application/json' \
                -d '{
                  "application": "my-app",
                  "image": "${{ secrets.DOCKER_REGISTRY }}/my-app:${{ github.sha }}"
                }'
    ```
    **Note:** The API endpoint and payload (`/api/v1/deploy`) are illustrative. You'll need to refer to the Komodo documentation for the exact API details for triggering a deployment.

## 5. Security Considerations on DigitalOcean

-   **Firewall:** Use DigitalOcean Cloud Firewalls to restrict access to your services.
    -   Allow port `443` (for the registry and Komodo) and `80` (for HTTP to HTTPS redirection) from anywhere.
    -   Consider restricting access to the Komodo UI (`443` on `komodo.your-domain.com`) to your company's IP addresses if it's for internal use only.
    -   Allow port `22` (SSH) only from your trusted IP addresses.

-   **Backups:** The Docker volume `registry-data` contains all your images, and `komodo-data` contains your Komodo configuration. Regularly back up these volumes to prevent data loss. You can use DigitalOcean Snapshots or other backup solutions.

-   **Monitoring:** Monitor your Droplet's resource usage (CPU, memory, disk space) to ensure your registry and deployment system remain performant.

## 6. Consolidating on a Single Host

Running your production websites on the same host as your Docker registry and Komodo is a very efficient setup. It centralizes your infrastructure, simplifies networking, and speeds up deployments since images are pulled locally without traversing the internet.

### Key Considerations

*   **Server Resources**: This is the most important factor. You are now running the registry, Komodo, a reverse proxy, and multiple websites. Monitor your Droplet's CPU, RAM, and Disk Space usage closely. Start with a general-purpose Droplet with at least 4GB or 8GB of RAM and scale up as needed based on your websites' traffic and resource consumption.
*   **Nginx as a Router**: As shown in the updated `nginx/conf.d/registry.conf` file, Nginx acts as the main router. It uses the `server_name` directive to direct traffic for each domain to the appropriate backend service defined in your `docker-compose.yml` file.
*   **Docker Compose Expansion**: The updated `docker-compose.yml` file includes a template for adding your websites as services. Each service will pull its image from your private registry.
*   **Security and Isolation**: While Docker provides process isolation, all services share the same host kernel. For enhanced security, you can create separate Docker networks for different groups of applications to prevent containers from communicating unless explicitly allowed. For instance, your websites might be on one network, and your admin tools (Komodo) on another.

### Deployment Workflow

The workflow remains the same, but it's now even more powerful:
1.  **Develop**: Make changes to one of your website's codebases.
2.  **CI**: Push the code, which triggers a GitHub Action. The action builds the new Docker image and pushes it to `registry.your-domain.com/my-website-1:latest`.
3.  **CD**: The GitHub Action then makes an API call to Komodo.
4.  **Deploy**: Komodo, running on the same server, pulls the new image from the local registry (which is very fast) and redeploys the `website1` service with the new image.

## Conclusion

You now have a fully functional, secure, and private Docker Registry on DigitalOcean, paired with Komodo as a powerful web UI for continuous deployment. By extending this setup, you can efficiently manage and deploy multiple production websites from a single, consolidated server. This setup gives you full control over your Docker images and automates your deployment workflow. Remember to keep your server and Docker images updated to protect against vulnerabilities.
