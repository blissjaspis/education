# Caddy Web Server: Beginner to Expert Guide

## Table of Contents
1. [Introduction](#introduction)
2. [Installation](#installation)
3. [Basic Concepts](#basic-concepts)
4. [Caddyfile Basics](#caddyfile-basics)
5. [Serving Static Files](#serving-static-files)
6. [Reverse Proxy](#reverse-proxy)
7. [Automatic HTTPS](#automatic-https)
8. [Advanced Routing](#advanced-routing)
9. [Load Balancing](#load-balancing)
10. [Authentication & Security](#authentication--security)
11. [Templates & Dynamic Content](#templates--dynamic-content)
12. [JSON Configuration](#json-configuration)
13. [Caddy API](#caddy-api)
14. [Plugins & Modules](#plugins--modules)
15. [Performance Optimization](#performance-optimization)
16. [Production Deployment](#production-deployment)
17. [Docker Integration](#docker-integration)
18. [Monitoring & Logging](#monitoring--logging)
19. [Troubleshooting](#troubleshooting)
20. [Expert Topics](#expert-topics)

---

## Introduction

### What is Caddy?

Caddy is a powerful, enterprise-ready, open-source web server with automatic HTTPS written in Go. It's designed to be easy to use and configure while providing advanced features.

**Key Features:**
- Automatic HTTPS with Let's Encrypt and ZeroSSL
- HTTP/1.1, HTTP/2, and HTTP/3 support
- Reverse proxy with load balancing
- Simple, intuitive configuration
- Dynamic configuration via API
- Extensible through modules
- Built-in security features

**Why Choose Caddy?**
- Zero-configuration HTTPS
- Simpler than Nginx/Apache
- Modern HTTP standards
- Great for microservices
- Active community and development

---

## Installation

### Linux

**Using Official Script:**
```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy
```

**Using Binary:**
```bash
curl -O https://caddyserver.com/api/download?os=linux&arch=amd64
chmod +x caddy
sudo mv caddy /usr/local/bin/
```

### macOS

**Using Homebrew:**
```bash
brew install caddy
```

### Docker

```bash
docker pull caddy:latest
```

### Windows

**Using Chocolatey:**
```powershell
choco install caddy
```

Or download from [caddyserver.com/download](https://caddyserver.com/download)

### Verify Installation

```bash
caddy version
```

---

## Basic Concepts

### Architecture

Caddy follows a modular architecture:
- **Core**: HTTP server and configuration management
- **Modules**: Extend functionality (handlers, matchers, encoders, etc.)
- **Adapters**: Convert different config formats to Caddy's native JSON

### Configuration Methods

1. **Caddyfile**: Human-readable configuration format
2. **JSON**: Native configuration format
3. **API**: Dynamic runtime configuration

### Key Terminology

- **Site Block**: Configuration for a specific domain/site
- **Directive**: Configuration instruction (e.g., `reverse_proxy`)
- **Matcher**: Condition for applying directives
- **Handler**: Module that processes requests
- **Upstream**: Backend server in reverse proxy setup

---

## Caddyfile Basics

### File Location

Default locations:
- Linux: `/etc/caddy/Caddyfile`
- macOS: `/usr/local/etc/Caddyfile`
- Windows: `C:\caddy\Caddyfile`

### Basic Syntax

```caddyfile
# Comment

# Global options
{
    # Global settings here
}

# Site block
example.com {
    # Directives for this site
    respond "Hello, World!"
}
```

### Hello World Example

```caddyfile
localhost:8080 {
    respond "Hello, Caddy!"
}
```

**Run:**
```bash
caddy run
# Or with specific file
caddy run --config Caddyfile
```

### Multiple Sites

```caddyfile
site1.com {
    respond "Site 1"
}

site2.com {
    respond "Site 2"
}

# Multiple domains, same config
site3.com, www.site3.com {
    respond "Site 3"
}
```

### Common Directives

```caddyfile
example.com {
    root * /var/www/html           # Document root
    file_server                     # Serve static files
    encode gzip                     # Compression
    log                            # Access logging
    tls email@example.com          # TLS settings
}
```

---

## Serving Static Files

### Basic Static Site

```caddyfile
example.com {
    root * /var/www/html
    file_server
}
```

### With Directory Browsing

```caddyfile
example.com {
    root * /var/www/html
    file_server browse
}
```

### Custom Error Pages

```caddyfile
example.com {
    root * /var/www/html
    file_server
    
    handle_errors {
        @404 {
            expression {http.error.status_code} == 404
        }
        rewrite @404 /404.html
        file_server
    }
}
```

### SPA (Single Page Application)

```caddyfile
example.com {
    root * /var/www/dist
    encode gzip
    
    # Try files, fallback to index.html (for Vue/React/Angular)
    try_files {path} /index.html
    file_server
}
```

### Multiple Directories

```caddyfile
example.com {
    root * /var/www
    
    handle /static/* {
        root * /var/www/static
        file_server
    }
    
    handle /media/* {
        root * /var/www/media
        file_server
    }
    
    handle {
        file_server
    }
}
```

---

## Reverse Proxy

### Basic Reverse Proxy

```caddyfile
example.com {
    reverse_proxy localhost:8080
}
```

### Multiple Backends (Load Balancing)

```caddyfile
example.com {
    reverse_proxy localhost:8080 localhost:8081 localhost:8082
}
```

### With Custom Headers

```caddyfile
example.com {
    reverse_proxy localhost:8080 {
        header_up Host {host}
        header_up X-Real-IP {remote_host}
        header_up X-Forwarded-For {remote_host}
        header_up X-Forwarded-Proto {scheme}
    }
}
```

### Path-Based Routing

```caddyfile
example.com {
    # API requests to backend
    reverse_proxy /api/* localhost:3000
    
    # Admin to different backend
    reverse_proxy /admin/* localhost:4000
    
    # Everything else to frontend
    reverse_proxy localhost:8080
}
```

### WebSocket Support

```caddyfile
example.com {
    reverse_proxy /ws localhost:8080 {
        # WebSocket settings
        header_up Connection Upgrade
        header_up Upgrade websocket
    }
    
    # Regular HTTP
    reverse_proxy localhost:8080
}
```

### Custom Transport Options

```caddyfile
example.com {
    reverse_proxy localhost:8080 {
        transport http {
            dial_timeout 5s
            response_header_timeout 10s
            read_timeout 30s
            write_timeout 30s
        }
    }
}
```

### Health Checks

```caddyfile
example.com {
    reverse_proxy localhost:8080 localhost:8081 {
        health_uri /health
        health_interval 10s
        health_timeout 5s
        health_status 200
    }
}
```

---

## Automatic HTTPS

### Default Behavior

Caddy automatically obtains and renews TLS certificates for all sites:

```caddyfile
example.com {
    respond "Automatic HTTPS!"
}
```

### Custom Email

```caddyfile
example.com {
    tls your-email@example.com
}
```

### Multiple Domains

```caddyfile
example.com, www.example.com {
    # Single certificate for both domains
    respond "Hello!"
}
```

### Wildcard Certificates

Requires DNS challenge:

```caddyfile
*.example.com {
    tls {
        dns cloudflare {env.CLOUDFLARE_API_TOKEN}
    }
    respond "Wildcard!"
}
```

### Self-Signed Certificates (Development)

```caddyfile
localhost {
    tls internal
}
```

### Custom Certificates

```caddyfile
example.com {
    tls /path/to/cert.pem /path/to/key.pem
}
```

### Disable HTTPS (Not Recommended)

```caddyfile
http://example.com {
    respond "HTTP only"
}
```

### Certificate Staging (Testing)

```caddyfile
{
    acme_ca https://acme-staging-v02.api.letsencrypt.org/directory
}

example.com {
    respond "Using staging certificates"
}
```

### Custom ACME Provider

```caddyfile
{
    acme_ca https://acme.zerossl.com/v2/DV90
}

example.com {
    respond "Using ZeroSSL"
}
```

---

## Advanced Routing

### Matchers

**Path Matchers:**
```caddyfile
example.com {
    @api {
        path /api/*
    }
    reverse_proxy @api localhost:3000
    
    @admin {
        path /admin/*
    }
    reverse_proxy @admin localhost:4000
}
```

**Header Matchers:**
```caddyfile
example.com {
    @mobile {
        header User-Agent *Mobile*
    }
    reverse_proxy @mobile localhost:8080
    
    reverse_proxy localhost:8081
}
```

**Method Matchers:**
```caddyfile
example.com {
    @postOnly {
        method POST
    }
    reverse_proxy @postOnly localhost:3000
}
```

**Complex Matchers:**
```caddyfile
example.com {
    @complex {
        path /api/*
        method POST PUT
        header Content-Type application/json
    }
    reverse_proxy @complex localhost:3000
}
```

### Rewrites

```caddyfile
example.com {
    # Simple rewrite
    rewrite /old /new
    
    # Pattern rewrite
    rewrite /posts/* /blog{uri}
    
    file_server
}
```

### Redirects

```caddyfile
example.com {
    # Permanent redirect (301)
    redir /old /new permanent
    
    # Temporary redirect (302)
    redir /temp /new
    
    # Redirect to another domain
    redir https://newdomain.com{uri} permanent
}

# Redirect www to non-www
www.example.com {
    redir https://example.com{uri} permanent
}
```

### Try Files

```caddyfile
example.com {
    root * /var/www/html
    
    # Try files in order, fallback to index.php
    try_files {path} {path}/ /index.php?{query}
    
    file_server
}
```

---

## Load Balancing

### Round Robin (Default)

```caddyfile
example.com {
    reverse_proxy localhost:8080 localhost:8081 localhost:8082
}
```

### Least Connections

```caddyfile
example.com {
    reverse_proxy localhost:8080 localhost:8081 {
        lb_policy least_conn
    }
}
```

### IP Hash (Sticky Sessions)

```caddyfile
example.com {
    reverse_proxy localhost:8080 localhost:8081 {
        lb_policy ip_hash
    }
}
```

### Cookie-Based Sticky Sessions

```caddyfile
example.com {
    reverse_proxy localhost:8080 localhost:8081 {
        lb_policy cookie {
            name sticky_session
            ttl 1h
        }
    }
}
```

### Weighted Load Balancing

```caddyfile
example.com {
    reverse_proxy localhost:8080 localhost:8081 {
        lb_policy weighted {
            weight 3  # 8080 gets 75% traffic
            weight 1  # 8081 gets 25% traffic
        }
    }
}
```

### Active Health Checks

```caddyfile
example.com {
    reverse_proxy localhost:8080 localhost:8081 {
        health_uri /health
        health_interval 10s
        health_timeout 5s
        health_status 200
        
        # Remove unhealthy upstreams
        fail_duration 30s
        max_fails 3
        unhealthy_status 500 502 503
    }
}
```

### Passive Health Checks

```caddyfile
example.com {
    reverse_proxy localhost:8080 localhost:8081 {
        max_fails 3
        fail_duration 30s
        unhealthy_status 500 502 503 504
    }
}
```

---

## Authentication & Security

### Basic Authentication

**Generate password hash:**
```bash
caddy hash-password
```

**Caddyfile:**
```caddyfile
example.com {
    basicauth /admin/* {
        admin JDJhJDE0JDRXaVF4SjdBQnQ3TmxkL2FYajZmUE9BMkhHaWMvN3JsUy9uZjREMXR3UUdLUjNCL0RTaUFF
    }
    
    reverse_proxy localhost:8080
}
```

### Forward Auth

```caddyfile
example.com {
    forward_auth localhost:9091 {
        uri /verify
        copy_headers X-User-Id X-User-Email
    }
    
    reverse_proxy localhost:8080
}
```

### IP Restrictions

```caddyfile
example.com {
    @blocked {
        remote_ip 192.168.1.100 10.0.0.0/8
    }
    respond @blocked "Access denied" 403
    
    reverse_proxy localhost:8080
}
```

### Rate Limiting

Using rate limit module (needs to be built with):
```caddyfile
example.com {
    rate_limit {
        zone dynamic {
            key {remote_host}
            events 100
            window 1m
        }
    }
    
    reverse_proxy localhost:8080
}
```

### Security Headers

```caddyfile
example.com {
    header {
        # Enable HSTS
        Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
        
        # Prevent MIME sniffing
        X-Content-Type-Options "nosniff"
        
        # Prevent clickjacking
        X-Frame-Options "SAMEORIGIN"
        
        # Enable XSS filter
        X-XSS-Protection "1; mode=block"
        
        # CSP
        Content-Security-Policy "default-src 'self'"
        
        # Remove server header
        -Server
    }
    
    reverse_proxy localhost:8080
}
```

### CORS

```caddyfile
example.com {
    @cors_preflight {
        method OPTIONS
    }
    
    handle @cors_preflight {
        header {
            Access-Control-Allow-Origin "*"
            Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS"
            Access-Control-Allow-Headers "Content-Type, Authorization"
            Access-Control-Max-Age "3600"
        }
        respond 204
    }
    
    header {
        Access-Control-Allow-Origin "*"
    }
    
    reverse_proxy localhost:8080
}
```

---

## Templates & Dynamic Content

### Basic Template

```caddyfile
example.com {
    root * /var/www/html
    templates
    file_server
}
```

**HTML file:**
```html
<!DOCTYPE html>
<html>
<head>
    <title>{{.Host}}</title>
</head>
<body>
    <h1>Welcome to {{.Host}}</h1>
    <p>Your IP: {{.RemoteIP}}</p>
    <p>Current time: {{now | date "2006-01-02 15:04:05"}}</p>
</body>
</html>
```

### Available Template Variables

```html
<!-- Request info -->
{{.Host}}              <!-- Hostname -->
{{.Method}}            <!-- HTTP method -->
{{.Path}}              <!-- Request path -->
{{.RemoteIP}}          <!-- Client IP -->
{{.Scheme}}            <!-- http or https -->

<!-- Headers -->
{{.Req.Header.Get "User-Agent"}}

<!-- Query parameters -->
{{.Req.URL.Query.Get "name"}}

<!-- Environment variables -->
{{env "HOME"}}

<!-- File operations -->
{{listFiles "/path/to/dir"}}
{{fileExists "/path/to/file"}}
{{readFile "/path/to/file"}}
```

### Template Functions

```html
<!-- String functions -->
{{.Host | upper}}
{{.Host | lower}}
{{.Path | trimPrefix "/api"}}
{{.Path | trimSuffix "/"}}
{{splitFrontMatter .OrigReq.URL.Path}}

<!-- Date/Time -->
{{now | date "Monday, Jan 2, 2006"}}
{{now.Unix}}

<!-- Math -->
{{add 1 2}}
{{sub 5 3}}
{{mul 2 3}}
{{div 10 2}}

<!-- Control flow -->
{{if eq .Method "POST"}}
    POST request
{{else}}
    GET request
{{end}}

{{range $file := listFiles "/path"}}
    <li>{{$file}}</li>
{{end}}
```

### Including Files

```caddyfile
example.com {
    root * /var/www/html
    templates {
        mime text/html
    }
    file_server
}
```

**Main file:**
```html
<!DOCTYPE html>
<html>
<head>
    <title>My Site</title>
</head>
<body>
    {{include "header.html"}}
    
    <main>
        Content here
    </main>
    
    {{include "footer.html"}}
</body>
</html>
```

---

## JSON Configuration

### Convert Caddyfile to JSON

```bash
caddy adapt --config Caddyfile
```

### Basic JSON Config

```json
{
  "apps": {
    "http": {
      "servers": {
        "srv0": {
          "listen": [":443"],
          "routes": [
            {
              "match": [
                {
                  "host": ["example.com"]
                }
              ],
              "handle": [
                {
                  "handler": "reverse_proxy",
                  "upstreams": [
                    {
                      "dial": "localhost:8080"
                    }
                  ]
                }
              ],
              "terminal": true
            }
          ]
        }
      }
    }
  }
}
```

### Static File Serving (JSON)

```json
{
  "apps": {
    "http": {
      "servers": {
        "srv0": {
          "listen": [":443"],
          "routes": [
            {
              "match": [
                {
                  "host": ["example.com"]
                }
              ],
              "handle": [
                {
                  "handler": "file_server",
                  "root": "/var/www/html"
                }
              ]
            }
          ]
        }
      }
    }
  }
}
```

### Run with JSON Config

```bash
caddy run --config config.json
```

---

## Caddy API

### Enable API

```caddyfile
{
    admin localhost:2019
}
```

### API Endpoints

**Get current config:**
```bash
curl http://localhost:2019/config/
```

**Load config:**
```bash
curl -X POST http://localhost:2019/load \
  -H "Content-Type: application/json" \
  -d @config.json
```

**Update specific path:**
```bash
curl -X PATCH http://localhost:2019/config/apps/http/servers/srv0/routes \
  -H "Content-Type: application/json" \
  -d @route.json
```

**Stop Caddy:**
```bash
curl -X POST http://localhost:2019/stop
```

### Dynamic Configuration Example

**Add a route dynamically:**
```bash
curl -X POST http://localhost:2019/config/apps/http/servers/srv0/routes \
  -H "Content-Type: application/json" \
  -d '{
    "match": [{"host": ["newsite.com"]}],
    "handle": [{
      "handler": "reverse_proxy",
      "upstreams": [{"dial": "localhost:9000"}]
    }]
  }'
```

**Remove a route:**
```bash
curl -X DELETE http://localhost:2019/config/apps/http/servers/srv0/routes/0
```

### API Authentication

```caddyfile
{
    admin localhost:2019 {
        origins localhost
    }
}
```

Or disable remote access:
```caddyfile
{
    admin off
}
```

---

## Plugins & Modules

### Built-in Modules

- **http.handlers**: `file_server`, `reverse_proxy`, `rewrite`, `templates`, etc.
- **http.matchers**: `path`, `header`, `method`, `expression`, etc.
- **tls.dns**: DNS providers for DNS-01 challenge
- **caddy.logging**: Logging configuration

### Popular Third-Party Modules

- **caddy-dns/cloudflare**: Cloudflare DNS for wildcard certs
- **caddy-dns/route53**: AWS Route53 DNS
- **caddy-rate-limit**: Rate limiting
- **caddy-auth-portal**: Authentication portal
- **caddy-git**: Git integration
- **caddy-s3-proxy**: S3 reverse proxy

### Building Caddy with Plugins

**Using xcaddy:**
```bash
# Install xcaddy
go install github.com/caddyserver/xcaddy/cmd/xcaddy@latest

# Build with plugins
xcaddy build \
  --with github.com/caddy-dns/cloudflare \
  --with github.com/mholt/caddy-ratelimit
```

**Using Docker:**
```dockerfile
FROM caddy:builder AS builder

RUN xcaddy build \
    --with github.com/caddy-dns/cloudflare \
    --with github.com/mholt/caddy-ratelimit

FROM caddy:latest

COPY --from=builder /usr/bin/caddy /usr/bin/caddy
```

### Using DNS Modules

**Cloudflare:**
```caddyfile
{
    email your@email.com
}

*.example.com {
    tls {
        dns cloudflare {env.CLOUDFLARE_API_TOKEN}
    }
    
    reverse_proxy localhost:8080
}
```

**AWS Route53:**
```caddyfile
*.example.com {
    tls {
        dns route53 {
            access_key_id {env.AWS_ACCESS_KEY_ID}
            secret_access_key {env.AWS_SECRET_ACCESS_KEY}
        }
    }
    
    reverse_proxy localhost:8080
}
```

---

## Performance Optimization

### Compression

```caddyfile
example.com {
    encode gzip zstd
    
    reverse_proxy localhost:8080
}
```

### Caching

**Static assets:**
```caddyfile
example.com {
    @static {
        path *.css *.js *.jpg *.png *.gif *.svg *.woff *.woff2
    }
    
    header @static {
        Cache-Control "public, max-age=31536000, immutable"
    }
    
    file_server
}
```

### HTTP/2 Server Push

```caddyfile
example.com {
    header Link "</style.css>; rel=preload; as=style"
    header Link "</app.js>; rel=preload; as=script"
    
    file_server
}
```

### Connection Limits

```caddyfile
{
    servers {
        max_connections 10000
    }
}

example.com {
    reverse_proxy localhost:8080
}
```

### Buffer Settings

```caddyfile
example.com {
    reverse_proxy localhost:8080 {
        transport http {
            read_buffer 8192
            write_buffer 8192
        }
    }
}
```

### Keep-Alive

```caddyfile
{
    servers {
        timeouts {
            idle 5m
        }
    }
}
```

---

## Production Deployment

### Systemd Service

**Create service file: `/etc/systemd/system/caddy.service`**
```ini
[Unit]
Description=Caddy
Documentation=https://caddyserver.com/docs/
After=network.target network-online.target
Requires=network-online.target

[Service]
Type=notify
User=caddy
Group=caddy
ExecStart=/usr/bin/caddy run --environ --config /etc/caddy/Caddyfile
ExecReload=/usr/bin/caddy reload --config /etc/caddy/Caddyfile --force
TimeoutStopSec=5s
LimitNOFILE=1048576
LimitNPROC=512
PrivateTmp=true
ProtectSystem=full
AmbientCapabilities=CAP_NET_BIND_SERVICE

[Install]
WantedBy=multi-user.target
```

**Enable and start:**
```bash
sudo systemctl daemon-reload
sudo systemctl enable caddy
sudo systemctl start caddy
sudo systemctl status caddy
```

### Environment Variables

```caddyfile
{
    admin {env.CADDY_ADMIN}
    email {env.CADDY_EMAIL}
}

example.com {
    reverse_proxy {env.BACKEND_HOST}:{env.BACKEND_PORT}
}
```

**Set in systemd:**
```ini
[Service]
Environment="CADDY_ADMIN=localhost:2019"
Environment="CADDY_EMAIL=admin@example.com"
Environment="BACKEND_HOST=localhost"
Environment="BACKEND_PORT=8080"
```

### Graceful Reloads

```bash
# Reload configuration without downtime
caddy reload
# Or via systemd
sudo systemctl reload caddy
```

### File Permissions

```bash
# Create caddy user
sudo useradd -r -s /bin/false caddy

# Set ownership
sudo chown -R caddy:caddy /etc/caddy
sudo chown -R caddy:caddy /var/www
```

### Directory Structure

```
/etc/caddy/
  ├── Caddyfile           # Main config
  ├── conf.d/             # Include additional configs
  │   ├── site1.caddy
  │   └── site2.caddy
  └── certs/              # Custom certificates

/var/www/
  ├── site1.com/
  ├── site2.com/
  └── default/

/var/log/caddy/
  ├── access.log
  └── error.log
```

### Including Config Files

```caddyfile
# Main Caddyfile
{
    admin localhost:2019
    email admin@example.com
}

import conf.d/*.caddy
```

**conf.d/site1.caddy:**
```caddyfile
site1.com {
    reverse_proxy localhost:8080
}
```

---

## Docker Integration

### Basic Dockerfile

```dockerfile
FROM caddy:latest

COPY Caddyfile /etc/caddy/Caddyfile
COPY site /usr/share/caddy
```

### Docker Run

```bash
docker run -d \
  --name caddy \
  -p 80:80 \
  -p 443:443 \
  -v $PWD/Caddyfile:/etc/caddy/Caddyfile \
  -v $PWD/site:/usr/share/caddy \
  -v caddy_data:/data \
  -v caddy_config:/config \
  caddy:latest
```

### Docker Compose

```yaml
version: '3.8'

services:
  caddy:
    image: caddy:latest
    container_name: caddy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
      - "443:443/udp"  # HTTP/3
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - ./site:/usr/share/caddy
      - caddy_data:/data
      - caddy_config:/config
    environment:
      - ACME_AGREE=true

volumes:
  caddy_data:
  caddy_config:
```

### Multi-Service Setup

```yaml
version: '3.8'

services:
  caddy:
    image: caddy:latest
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data
      - caddy_config:/config
    networks:
      - webnet

  backend:
    image: node:18
    working_dir: /app
    command: npm start
    volumes:
      - ./backend:/app
    networks:
      - webnet

  frontend:
    image: nginx:alpine
    volumes:
      - ./frontend/dist:/usr/share/nginx/html
    networks:
      - webnet

networks:
  webnet:

volumes:
  caddy_data:
  caddy_config:
```

**Caddyfile:**
```caddyfile
example.com {
    reverse_proxy /api/* backend:3000
    reverse_proxy frontend:80
}
```

### Custom Build with Modules

```dockerfile
FROM caddy:builder AS builder

RUN xcaddy build \
    --with github.com/caddy-dns/cloudflare \
    --with github.com/mholt/caddy-ratelimit

FROM caddy:latest

COPY --from=builder /usr/bin/caddy /usr/bin/caddy
COPY Caddyfile /etc/caddy/Caddyfile
```

---

## Monitoring & Logging

### Access Logs

```caddyfile
example.com {
    log {
        output file /var/log/caddy/access.log
        format json
    }
    
    reverse_proxy localhost:8080
}
```

### Structured Logging

```caddyfile
example.com {
    log {
        output file /var/log/caddy/access.log {
            roll_size 100mb
            roll_keep 10
            roll_keep_for 720h
        }
        format json {
            time_format iso8601
        }
        level INFO
    }
    
    reverse_proxy localhost:8080
}
```

### Filter Logs

```caddyfile
example.com {
    log {
        output file /var/log/caddy/access.log
        format json
        
        # Exclude health checks
        skip {
            path /health /metrics
        }
    }
    
    reverse_proxy localhost:8080
}
```

### Multiple Log Files

```caddyfile
example.com {
    # API logs
    log {
        output file /var/log/caddy/api.log
        include /api/*
    }
    
    # Admin logs
    log {
        output file /var/log/caddy/admin.log
        include /admin/*
    }
    
    # General logs
    log {
        output file /var/log/caddy/access.log
    }
    
    reverse_proxy localhost:8080
}
```

### Metrics Endpoint

```caddyfile
{
    servers {
        metrics
    }
}

example.com {
    reverse_proxy localhost:8080
}

# Metrics endpoint (Prometheus format)
:2019 {
    metrics /metrics
}
```

### Prometheus Integration

**Prometheus config:**
```yaml
scrape_configs:
  - job_name: 'caddy'
    static_configs:
      - targets: ['localhost:2019']
    metrics_path: '/metrics'
```

### Error Logging

```caddyfile
{
    log {
        output file /var/log/caddy/error.log
        level ERROR
    }
}

example.com {
    reverse_proxy localhost:8080
}
```

---

## Troubleshooting

### Check Configuration

```bash
# Validate Caddyfile
caddy validate --config Caddyfile

# Adapt and print JSON
caddy adapt --config Caddyfile --pretty
```

### Debug Mode

```bash
# Run with debug logging
caddy run --config Caddyfile --debug
```

### Common Issues

**1. Certificate Issues**

Check certificate status:
```bash
# Via API
curl http://localhost:2019/pki/ca/local | jq
```

Force certificate renewal:
```caddyfile
example.com {
    tls {
        # Use staging for testing
        ca https://acme-staging-v02.api.letsencrypt.org/directory
    }
}
```

**2. Port Already in Use**

```bash
# Check what's using port 80/443
sudo lsof -i :80
sudo lsof -i :443

# Kill the process
sudo kill <PID>
```

**3. Permission Denied**

```bash
# Allow binding to privileged ports
sudo setcap 'cap_net_bind_service=+ep' /usr/bin/caddy
```

**4. DNS Issues**

Test DNS resolution:
```bash
dig example.com
nslookup example.com
```

**5. Reverse Proxy Not Working**

Check backend is running:
```bash
curl localhost:8080
```

Enable debug headers:
```caddyfile
example.com {
    reverse_proxy localhost:8080 {
        header_up X-Forwarded-For {remote}
        header_up X-Forwarded-Proto {scheme}
    }
    
    header {
        X-Debug-Backend localhost:8080
    }
}
```

### Viewing Logs

```bash
# Systemd logs
sudo journalctl -u caddy -f

# File logs
tail -f /var/log/caddy/access.log
tail -f /var/log/caddy/error.log
```

### Testing Configuration

```bash
# Dry run (doesn't start server)
caddy adapt --config Caddyfile --validate

# Test specific directives
caddy file-server --listen :8080 --root /var/www/html
caddy reverse-proxy --from :8080 --to localhost:3000
```

---

## Expert Topics

### Layer 4 Proxying (TCP/UDP)

```caddyfile
:3306 {
    reverse_proxy mysql:3306
}

:6379 {
    reverse_proxy redis:6379
}
```

### Advanced Matchers with CEL

```caddyfile
example.com {
    @complex {
        expression {
            (path('/api/*') && method('POST')) || 
            (header_regexp('User-Agent', 'bot.*'))
        }
    }
    
    respond @complex "Matched complex condition" 200
}
```

### Custom TLS Configuration

```caddyfile
example.com {
    tls {
        protocols tls1.2 tls1.3
        ciphers TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384 TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
        curves x25519 secp384r1
        
        # OCSP stapling
        ocsp_stapling on
        
        # Must staple
        must_staple
    }
}
```

### Request Body Manipulation

```caddyfile
example.com {
    route {
        # Read request body
        reverse_proxy localhost:8080 {
            @hasBody {
                not header Content-Length ""
            }
            
            # Limit body size
            request_buffers 10MB
        }
    }
}
```

### Custom Error Handling

```caddyfile
example.com {
    reverse_proxy localhost:8080
    
    handle_errors {
        @5xx expression `{http.error.status_code} >= 500`
        
        rewrite @5xx /errors/5xx.html
        file_server
    }
}
```

### On-Demand TLS

```caddyfile
{
    on_demand_tls {
        ask http://localhost:8080/check-domain
        interval 2m
        burst 5
    }
}

https:// {
    tls {
        on_demand
    }
    
    reverse_proxy localhost:8080
}
```

### Storage Backend

**S3 Storage:**
```json
{
  "storage": {
    "module": "s3",
    "host": "s3.amazonaws.com",
    "bucket": "caddy-storage",
    "access_key": "YOUR_ACCESS_KEY",
    "secret_key": "YOUR_SECRET_KEY"
  }
}
```

**Redis Storage:**
```json
{
  "storage": {
    "module": "redis",
    "address": "localhost:6379",
    "db": 0
  }
}
```

### Multi-Region Setup

```caddyfile
{
    storage redis {
        address redis:6379
        db 0
    }
    
    cluster {
        members ["caddy1:2019", "caddy2:2019", "caddy3:2019"]
    }
}

example.com {
    reverse_proxy backend1:8080 backend2:8080 backend3:8080
}
```

### Zero-Downtime Deployments

```bash
# Blue-Green Deployment Script
#!/bin/bash

# Deploy new version
docker-compose -f docker-compose.blue.yml up -d

# Health check
until $(curl --output /dev/null --silent --head --fail http://localhost:8081/health); do
    sleep 5
done

# Update Caddy config
cat > Caddyfile <<EOF
example.com {
    reverse_proxy localhost:8081
}
EOF

# Reload Caddy
caddy reload

# Stop old version
docker-compose -f docker-compose.green.yml down
```

### Circuit Breaker Pattern

```caddyfile
example.com {
    reverse_proxy localhost:8080 localhost:8081 {
        # Health checks
        health_uri /health
        health_interval 10s
        
        # Circuit breaker behavior
        max_fails 3
        fail_duration 30s
        
        # Timeouts
        transport http {
            dial_timeout 5s
            response_header_timeout 10s
        }
    }
}
```

### Advanced Logging with Filters

```caddyfile
example.com {
    log {
        output file /var/log/caddy/access.log {
            roll_size 100mb
            roll_keep 10
        }
        
        format transform "{common_log}" {
            request>headers>User-Agent regex "bot" ""
            request>headers>Cookie regex ".*" "[REDACTED]"
        }
        
        # Include/exclude
        include /api/*
        exclude /health /metrics
    }
    
    reverse_proxy localhost:8080
}
```

### Request Tracing

```caddyfile
example.com {
    # Add trace ID
    header {
        X-Request-ID {uuid}
    }
    
    log {
        output file /var/log/caddy/access.log
        format json {
            request_id {header.X-Request-ID}
        }
    }
    
    reverse_proxy localhost:8080 {
        header_up X-Request-ID {header.X-Request-ID}
    }
}
```

### Dynamic Backend Discovery

Using Caddy API with service discovery:

```python
import requests
import consul

# Connect to Consul
c = consul.Consul()

# Get healthy services
services = c.health.service('backend', passing=True)[1]

# Build upstreams
upstreams = [
    {"dial": f"{s['Service']['Address']}:{s['Service']['Port']}"}
    for s in services
]

# Update Caddy config
config = {
    "match": [{"host": ["example.com"]}],
    "handle": [{
        "handler": "reverse_proxy",
        "upstreams": upstreams
    }]
}

requests.post(
    "http://localhost:2019/config/apps/http/servers/srv0/routes",
    json=config
)
```

### Advanced Security: ModSecurity

Using Caddy with ModSecurity WAF:

```yaml
# docker-compose.yml
services:
  modsecurity:
    image: owasp/modsecurity-crs:nginx-alpine
    volumes:
      - ./modsec-config:/etc/nginx/conf.d
    
  caddy:
    image: caddy:latest
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
    depends_on:
      - modsecurity
```

**Caddyfile:**
```caddyfile
example.com {
    # Route through ModSecurity
    reverse_proxy modsecurity:80 {
        header_up Host {host}
    }
}
```

### Performance Testing

```bash
# Load testing with Apache Bench
ab -n 10000 -c 100 https://example.com/

# Load testing with wrk
wrk -t12 -c400 -d30s https://example.com/

# Monitor metrics
watch -n 1 'curl -s http://localhost:2019/metrics | grep caddy_http_requests_total'
```

---

## Additional Resources

### Official Documentation
- [Caddy Docs](https://caddyserver.com/docs/)
- [Caddy Community](https://caddy.community/)
- [GitHub Repository](https://github.com/caddyserver/caddy)

### Useful Tools
- **xcaddy**: Build custom Caddy with plugins
- **caddy-docker**: Official Docker images
- **Caddy API**: Dynamic configuration

### Best Practices Checklist

✅ **Security:**
- [ ] Use automatic HTTPS for all sites
- [ ] Implement security headers
- [ ] Enable rate limiting for APIs
- [ ] Use basic auth or forward auth for admin panels
- [ ] Keep Caddy updated

✅ **Performance:**
- [ ] Enable compression (gzip/zstd)
- [ ] Configure caching headers
- [ ] Use HTTP/2 and HTTP/3
- [ ] Implement load balancing
- [ ] Monitor metrics

✅ **Reliability:**
- [ ] Configure health checks
- [ ] Set appropriate timeouts
- [ ] Implement graceful reloads
- [ ] Use circuit breakers
- [ ] Set up monitoring and alerts

✅ **Operations:**
- [ ] Use systemd for service management
- [ ] Configure structured logging
- [ ] Implement log rotation
- [ ] Use environment variables for secrets
- [ ] Document your configuration

✅ **High Availability:**
- [ ] Use shared storage (Redis/S3) for certificates
- [ ] Implement multiple backend servers
- [ ] Configure health checks
- [ ] Use DNS-based failover
- [ ] Test disaster recovery procedures

---

## Conclusion

Caddy is a powerful, modern web server that simplifies many complex tasks like HTTPS automation, reverse proxying, and load balancing. This guide covered everything from basic static file serving to advanced production deployments.

**Key Takeaways:**
- Caddy automatically handles HTTPS certificates
- The Caddyfile syntax is intuitive and powerful
- Dynamic configuration via API enables automation
- Built-in modules cover most use cases
- Production-ready with minimal configuration

**Next Steps:**
1. Set up a test environment
2. Deploy a simple site with automatic HTTPS
3. Experiment with reverse proxying
4. Explore advanced features like load balancing
5. Join the Caddy community for support

Happy Caddying! 🎉

