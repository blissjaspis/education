# Private Docker Registry on DigitalOcean

This guide provides a comprehensive walkthrough for setting up a secure, private Docker registry on a DigitalOcean VPS. We'll cover security, authentication, and deployment options using either Nginx or Caddy as reverse proxies. Includes optional CI/CD integration with Komodo.

## Table of Contents
- [Introduction](#introduction)
- [Registry-Only Setup (Without Komodo)](#registry-only-setup-without-komodo)
- [Alternative: Using Caddy Instead of Nginx](#alternative-using-caddy-instead-of-nginx)
- [Prerequisites](#prerequisites)
- [1. Setting up the Core Infrastructure](#1-setting-up-the-core-infrastructure)
- [2. Pushing Images from Local Development](#2-pushing-images-from-local-development)
- [3. Deploying Your First Website](#3-deploying-your-first-website)
- [4. Managing and Deploying More Websites](#4-managing-and-deploying-more-websites)
- [5. The Multi-Compose Strategy Explained](#5-the-multi-compose-strategy-explained)
- [6. CI/CD Integration with Komodo](#6-ci-cd-integration-with-komodo)
- [7. Security Considerations on DigitalOcean](#7-security-considerations-on-digitalocean)
- [Conclusion](#conclusion)

## Introduction

A private Docker registry is a storage system for your Docker images, hosted on your own infrastructure. It's essential when you want to keep your images private and not publish them to the public Docker Hub. Using a private registry gives you control over where your images are stored, reduces reliance on third-party services, and can improve security and performance.

This guide will use a DigitalOcean Droplet as the host, but the principles apply to any VPS provider or on-premises server.

## Registry-Only Setup (Without Komodo)

**Komodo is optional.** If you only want a simple private registry for storing and retrieving Docker images (like Docker Hub), you can skip Komodo entirely.

### What You Get With Registry Only:
- ✅ Private image storage (unlimited repositories)
- ✅ Push/pull from local development
- ✅ Authentication and security
- ✅ HTTPS with TLS certificates

### What You Lose Without Komodo:
- ❌ Web UI for deployment management
- ❌ Automated CI/CD deployment triggers
- ❌ Advanced deployment features

### Steps to Skip for Registry-Only Setup:
1. **Skip Komodo DNS**: Don't create `komodo.your-domain.com` DNS record
2. **Skip Komodo in nginx**: Remove Komodo upstream and server blocks from `core.conf`
3. **Skip Komodo service**: Remove the `komodo` service from `docker-compose-core.yml`
4. **Skip Komodo volume**: Remove `komodo-data` volume from compose file
5. **Skip Komodo SSL**: Don't get SSL certificate for `komodo.your-domain.com`
6. **Skip Section 6**: "CI/CD Integration with Komodo" (not needed for basic registry)

The registry will work identically for local development - you can still push, pull, and manage images as described in Section 2.

## Alternative: Using Caddy Instead of Nginx

If you prefer Caddy over Nginx, here's the configuration. Caddy handles SSL certificates automatically with Let's Encrypt.

### Caddy Configuration (Caddyfile)

```caddyfile
registry.your-domain.com {
    # Automatic HTTPS with Let's Encrypt
    tls your-email@example.com

    # Basic auth for registry
    basicauth {
        # Hash generated with: caddy hash-password
        your_user $2a$14$XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
    }

    # Proxy to registry
    reverse_proxy registry:5000
}

# Add Komodo config if using it
komodo.your-domain.com {
    tls your-email@example.com
    reverse_proxy komodo:8080
}
```

### Docker Setup for Caddy

1. **Create password hash:**
   ```bash
   docker run --rm -it caddy:2 caddy hash-password
   # Enter your password, copy the hash
   ```

2. **Create Caddyfile** at `~/projects/core/Caddyfile`

3. **Create password file** at `~/projects/core/registry.password`:
   ```bash
   echo "your_user:$2a$14$YOUR_HASH_HERE" > registry.password
   ```

4. **Docker Compose for Caddy:**
   ```yaml
   services:
     caddy:
       image: caddy:2
       ports:
         - "80:80"
         - "443:443"
       volumes:
         - ./Caddyfile:/etc/caddy/Caddyfile:ro
         - ./registry.password:/etc/caddy/registry.password:ro
         - caddy-data:/data
         - caddy-config:/config
       networks:
         - proxy-net

   networks:
     proxy-net:
       external: true
       name: nginx-proxy-net  # Same network as registry

   volumes:
     caddy-data:
     caddy-config:
   ```

**Note:** Caddy automatically handles SSL certificates - no need for certbot. The `basicauth` directive replaces nginx's `auth_basic` and `auth_basic_user_file`.

## Prerequisites

- A DigitalOcean account.
- A registered domain name (e.g., `your-domain.com`). This is highly recommended for setting up a secure registry with TLS.
- A DigitalOcean Droplet (VPS) running Ubuntu 24.04 LTS.
- Docker and Docker Compose V2 installed on your Droplet.
- DNS `A` record pointing a subdomain (e.g., `registry.your-domain.com`) to your Droplet's IP address.

**Additional for full setup with Komodo:**
- Another subdomain for Komodo (e.g., `komodo.your-domain.com`) if using the deployment UI.
- Additional domains for each website you plan to host if deploying websites.

## 1. Setting up the Core Infrastructure

This section covers setting up the shared, stable components of your system: the Nginx reverse proxy and the private Docker Registry. (Komodo is optional - see [Registry-Only Setup](#registry-only-setup-without-komodo) if you don't need the deployment UI.)

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
    Create a file at `~/projects/core/nginx/conf.d/core.conf`. This file will contain the routing for your registry (and Komodo if you're using it).

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

    **Note:** If you're not using Komodo, remove the Komodo upstream and server blocks from this file.

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
         --email your-email@example.com --agree-tos --no-eff-email -n
    # Add -d komodo.your-domain.com if using Komodo
    ```

7.  **Create the Core Infrastructure Compose File:**
    Create a file at `~/projects/core/docker-compose-core.yml`. This manages your foundational services. (If not using Komodo, remove the `komodo` service and `komodo-data` volume from this file.)

    ```yaml
    services:
      registry:
        image: registry:3
        container_name: private-registry
        restart: always
        environment:
          - REGISTRY_HTTP_ADDR=0.0.0.0:5000
          - REGISTRY_HTTP_SECRET=${REGISTRY_HTTP_SECRET}
          - REGISTRY_STORAGE_DELETE_ENABLED=true
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

8.  **Set Registry Environment Variable:**
    Generate a random secret for the registry:
    ```bash
    export REGISTRY_HTTP_SECRET=$(openssl rand -hex 32)
    echo "REGISTRY_HTTP_SECRET=$REGISTRY_HTTP_SECRET" >> ~/.bashrc
    ```

9.  **Start the Core Infrastructure:**
    ```bash
    docker-compose -f docker-compose-core.yml up -d
    ```

## 2. Pushing Images from Local Development

Before deploying websites, you need to push your Docker images to the private registry from your local machine.

1.  **Authenticate with Your Registry:**
    From your local development machine:
    ```bash
    docker login registry.your-domain.com
    # Enter the username and password you created with htpasswd
    ```

2.  **Tag and Push Your Images:**
    ```bash
    # Tag your local image
    docker tag my-app:latest registry.your-domain.com/my-app:latest

    # Push to your private registry
    docker push registry.your-domain.com/my-app:latest
    ```

3.  **Pull Images from Registry:**
    ```bash
    # Pull from your private registry
    docker pull registry.your-domain.com/my-app:latest
    ```

4.  **Manage Registry Content:**
    ```bash
    # List repositories via API
    curl -u your_user https://registry.your-domain.com/v2/_catalog

    # List tags for a repository
    curl -u your_user https://registry.your-domain.com/v2/my-app/tags/list

    # Delete an image (requires REGISTRY_STORAGE_DELETE_ENABLED=true)
    curl -u your_user -X DELETE https://registry.your-domain.com/v2/my-app/manifests/latest
    ```

## 3. Deploying Your First Website

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

## 4. Managing and Deploying More Websites

To deploy `website2`, `website3`, and so on up to 20+, you simply repeat the steps from Section 3:
1.  Create `~/projects/website2`.
2.  Create `~/projects/core/nginx/conf.d/website2.conf`.
3.  Get the certificate for `your-website-2.com`.
4.  Reload Nginx.
5.  Create `~/projects/website2/docker-compose-website2.yml`.
6.  `docker-compose up` the new website.

This file-based approach is robust and easy to automate with simple scripts, and as discussed, will scale perfectly well for 20+ websites on a single host, provided the server has enough resources.

## 5. The Multi-Compose Strategy Explained

The architecture in this guide uses the best practice for this scenario: **multiple, separate `docker-compose` files linked by a single, shared Docker network.**

*   **Isolation:** Core infrastructure (`core.yml`) is separate from applications (`website1.yml`). You can update one without affecting the others.
*   **Communication:** The manually created `nginx-proxy-net` allows the Nginx container to communicate with and route traffic to all website containers, regardless of which compose file they are in.
*   **Clarity:** This structure makes it obvious where the configuration for each piece of your system lives, which is critical for long-term maintenance.

## 6. CI/CD Integration with Komodo

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
            uses: actions/checkout@v4

          - name: Log in to the Container registry
            uses: docker/login-action@v3
            with:
              registry: ${{ secrets.DOCKER_REGISTRY }}
              username: ${{ secrets.DOCKER_USERNAME }}
              password: ${{ secrets.DOCKER_PASSWORD }}

          - name: Build and push Docker image
            id: build_and_push
            uses: docker/build-push-action@v6
            with:
              context: .
              push: true
              tags: ${{ secrets.DOCKER_REGISTRY }}/my-app:${{ github.sha }}

          - name: Trigger Komodo Deployment
            run: |
              curl -X POST \
                '${{ secrets.KOMODO_URL }}/api/execute' \
                -H 'Authorization: Bearer ${{ secrets.KOMODO_API_KEY }}' \
                -H 'Content-Type: application/json' \
                -d '{
                  "type": "RunBuild",
                  "params": { "build": "my-app" }
                }'
    ```
    **Note:** Refer to [Komodo API docs](https://komo.do/docs/api) for exact endpoint details. The `/api/execute` endpoint triggers builds/deployments.

## 7. Security Considerations on DigitalOcean

-   **Firewall:** Use DigitalOcean Cloud Firewalls to restrict access to your services.
    -   Allow port `443` (for the registry and Komodo) and `80` (for HTTP to HTTPS redirection) from anywhere.
    -   Consider restricting access to the Komodo UI (`443` on `komodo.your-domain.com`) to your company's IP addresses if it's for internal use only.
    -   Allow port `22` (SSH) only from your trusted IP addresses.

-   **Backups:** The Docker volume `registry-data` contains all your images, and `komodo-data` contains your Komodo configuration. Regularly back up these volumes. **Crucially, also back up your `~/projects` directory**, as it now contains all your vital configuration-as-code.

-   **Monitoring:** Monitor your Droplet's resource usage (CPU, memory, disk space) to ensure your registry and deployment system remain performant.

## Conclusion

You now have a fully functional, secure, and private Docker Registry on DigitalOcean. For basic registry usage (storing and retrieving images like Docker Hub), this is complete - you can skip Komodo entirely as described in the [Registry-Only Setup](#registry-only-setup-without-komodo) section.

If you choose to add Komodo, you'll get a powerful web UI for continuous deployment, allowing you to efficiently and safely manage and deploy 20+ production websites from a single, consolidated server. This setup gives you full control over your Docker images and automates your deployment workflow. Remember to keep your server and Docker images updated to protect against vulnerabilities.
