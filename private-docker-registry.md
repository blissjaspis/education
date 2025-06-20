# Private Docker Registry on DigitalOcean

This guide provides a comprehensive walkthrough for setting up a secure, private Docker registry on a DigitalOcean VPS. We'll cover security, adding a web UI, and integrating with a CI/CD pipeline.

## Table of Contents
- [Introduction](#introduction)
- [Prerequisites](#prerequisites)
- [1. Setting up the Core Infrastructure](#1-setting-up-the-core-infrastructure)
- [2. Deploying Your First Website](#2-deploying-your-first-website)
- [3. Managing and Deploying More Websites](#3-managing-and-deploying-more-websites)
- [4. The Multi-Compose Strategy Explained](#4-the-multi-compose-strategy-explained)
- [5. CI/CD Integration with Komodo](#5-ci-cd-integration-with-komodo)
- [6. Security Considerations on DigitalOcean](#6-security-considerations-on-digitalocean)
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

## 1. Setting up the Core Infrastructure

This section covers setting up the shared, stable components of your system: the Nginx reverse proxy, the private Docker Registry, and the Komodo deployment tool.

1.  **Create Project Directories:**
    On your DigitalOcean droplet, create a main directory for your projects. We'll create separate subdirectories for the core infrastructure and each website to keep things organized.
    ```bash
    # Run from your home directory, e.g., /root/
    mkdir -p ~/projects/core
    cd ~/projects/core
    ```

2.  **Create a Shared Docker Network:**
    This is the key to allowing containers from different compose files to communicate. We create one external network that all our services will connect to.
    ```bash
    docker network create nginx-proxy-net
    ```

3.  **Create Directories for Nginx and Certbot:**
    Inside `~/projects/core`, create the necessary config directories.
    ```bash
    mkdir -p nginx/conf.d
    mkdir -p certbot/www
    ```

4.  **Create Nginx Configuration for Core Services:**
    Create a file at `~/projects/core/nginx/conf.d/core.conf`. This file will contain the routing for your registry and Komodo.

    ```nginx
    upstream docker-registry {
      server registry:5000;
    }

    server {
      listen 80;
      server_name registry.your-domain.com;
      location /.well-known/acme-challenge/ { root /var/www/certbot; }
      location / { return 301 https://$host$request_uri; }
    }

    server {
      listen 443 ssl http2;
      server_name registry.your-domain.com;

      # SSL Certs
      ssl_certificate /etc/letsencrypt/live/registry.your-domain.com/fullchain.pem;
      ssl_certificate_key /etc/letsencrypt/live/registry.your-domain.com/privkey.pem;
      
      # Auth
      auth_basic "Docker Registry";
      auth_basic_user_file /etc/nginx/registry.password;
      
      location / {
        proxy_pass http://docker-registry;
        # ... other proxy headers
      }
    }

    # Add Komodo config here as well...
    ```

5.  **Set up Registry Authentication:**
    Create the password file inside the `nginx` directory.
    ```bash
    sudo apt-get update && sudo apt-get install -y apache2-utils
    htpasswd -c ./nginx/registry.password your_user
    ```

6.  **Obtain SSL Certificates for Core Services:**
    ```bash
    sudo apt-get update && sudo apt-get install -y certbot
    sudo certbot certonly --webroot -w ./certbot/www \
         -d registry.your-domain.com \
         -d komodo.your-domain.com \
         --email your-email@example.com --agree-tos --no-eff-email -n
    ```

7.  **Create the Core Infrastructure Compose File:**
    Create a file at `~/projects/core/docker-compose-core.yml`. This manages your foundational services.

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
          - proxy-net

      komodo:
        image: ghcr.io/moghtech/komodo:latest
        container_name: komodo
        restart: always
        volumes:
          - komodo-data:/app/data
        networks:
          - proxy-net

      nginx-proxy:
        image: nginx:latest
        container_name: nginx-proxy
        restart: always
        ports:
          - "80:80"
          - "443:443"
        volumes:
          - ./nginx/conf.d:/etc/nginx/conf.d:ro
          - ./nginx/registry.password:/etc/nginx/registry.password:ro
          - /etc/letsencrypt:/etc/letsencrypt:ro
          - ./certbot/www:/var/www/certbot:ro
        networks:
          - proxy-net

    networks:
      proxy-net:
        external: true
        name: nginx-proxy-net

    volumes:
      registry-data:
      komodo-data:
    ```

8.  **Start the Core Infrastructure:**
    ```bash
    docker-compose -f docker-compose-core.yml up -d
    ```

## 2. Deploying Your First Website

Now, let's deploy `your-website-1.com`. This process is completely separate and won't affect the core services.

1.  **Create the Project Directory:**
    ```bash
    mkdir -p ~/projects/website1
    cd ~/projects/website1
    ```

2.  **Add Nginx Configuration for the Website:**
    Create a *new* config file for this website at `~/projects/core/nginx/conf.d/website1.conf`:

    ```nginx
    upstream website1 {
      # This name must match the service name in docker-compose-website1.yml
      server website1:8080; 
    }

    server {
      listen 80;
      server_name your-website-1.com;
      location /.well-known/acme-challenge/ { root /var/www/certbot; }
      location / { return 301 https://$host$request_uri; }
    }

    server {
      listen 443 ssl http2;
      server_name your-website-1.com;

      ssl_certificate /etc/letsencrypt/live/your-website-1.com/fullchain.pem;
      ssl_certificate_key /etc/letsencrypt/live/your-website-1.com/privkey.pem;
      
      location / {
        proxy_pass http://website1;
        proxy_set_header Host $host;
        # ... other proxy headers
      }
    }
    ```

3.  **Obtain the Website's SSL Certificate:**
    Note the webroot path is absolute now.
    ```bash
    sudo certbot certonly --webroot -w ~/projects/core/certbot/www -d your-website-1.com -n
    ```

4.  **Reload Nginx:**
    This crucial step makes Nginx aware of the new website without restarting anything.
    ```bash
    docker container exec nginx-proxy nginx -s reload
    ```

5.  **Create the Website's Compose File:**
    Create this file at `~/projects/website1/docker-compose-website1.yml`. Notice how minimal it is.

    ```yaml
    version: '3.7'

    services:
      website1:
        # Pull the image from your private registry
        image: registry.your-domain.com/my-website-1:latest
        container_name: website1-container
        restart: always
        networks:
          - proxy-net # Connect to the shared network
        # environment:
        #   - DATABASE_URL=...

    networks:
      proxy-net:
        external: true
        name: nginx-proxy-net # Specify the existing network
    ```

6.  **Deploy the Website:**
    From inside `~/projects/website1`, run:
    ```bash
    docker-compose -f docker-compose-website1.yml up -d
    ```

Your website is now live, running independently of the core stack.

## 3. Managing and Deploying More Websites

To deploy `website2`, `website3`, and so on up to 20+, you simply repeat the steps from Section 2:
1.  Create `~/projects/website2`.
2.  Create `~/projects/core/nginx/conf.d/website2.conf`.
3.  Get the certificate for `your-website-2.com`.
4.  Reload Nginx.
5.  Create `~/projects/website2/docker-compose-website2.yml`.
6.  `docker-compose up` the new website.

This file-based approach is robust and easy to automate with simple scripts, and as discussed, will scale perfectly well for 20+ websites on a single host, provided the server has enough resources.

## 4. The Multi-Compose Strategy Explained

The architecture in this guide uses the best practice for this scenario: **multiple, separate `docker-compose` files linked by a single, shared Docker network.**

*   **Isolation:** Core infrastructure (`core.yml`) is separate from applications (`website1.yml`). You can update one without affecting the others.
*   **Communication:** The manually created `nginx-proxy-net` allows the Nginx container to communicate with and route traffic to all website containers, regardless of which compose file they are in.
*   **Clarity:** This structure makes it obvious where the configuration for each piece of your system lives, which is critical for long-term maintenance.

## 5. CI/CD Integration with Komodo

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

## 6. Security Considerations on DigitalOcean

-   **Firewall:** Use DigitalOcean Cloud Firewalls to restrict access to your services.
    -   Allow port `443` (for the registry and Komodo) and `80` (for HTTP to HTTPS redirection) from anywhere.
    -   Consider restricting access to the Komodo UI (`443` on `komodo.your-domain.com`) to your company's IP addresses if it's for internal use only.
    -   Allow port `22` (SSH) only from your trusted IP addresses.

-   **Backups:** The Docker volume `registry-data` contains all your images, and `komodo-data` contains your Komodo configuration. Regularly back up these volumes. **Crucially, also back up your `~/projects` directory**, as it now contains all your vital configuration-as-code.

-   **Monitoring:** Monitor your Droplet's resource usage (CPU, memory, disk space) to ensure your registry and deployment system remain performant.

## Conclusion

You now have a fully functional, secure, and private Docker Registry on DigitalOcean, paired with Komodo as a powerful web UI for continuous deployment. By using a multi-compose file architecture, you can efficiently and safely manage and deploy 20+ production websites from a single, consolidated server. This setup gives you full control over your Docker images and automates your deployment workflow. Remember to keep your server and Docker images updated to protect against vulnerabilities.
