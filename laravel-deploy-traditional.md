# Deploying Laravel to a Traditional Server

This guide documents the process of deploying a Laravel application to a traditional Linux server (e.g., Ubuntu/Debian). It focuses on a secure permission setup using a specific non-root user.

## 1. User Management

We will use a dedicated non-root user named `deployer` (or your preferred username) for deployment and management. This user will share group membership with the web server user (`www-data`).

### Create the User

```bash
# Create user 'deployer'
sudo adduser deployer

# Add 'deployer' to the 'www-data' group
sudo usermod -aG www-data deployer
```

> **Note:** Ensure you are logged in as `deployer` or have switched to this user for the subsequent steps involving code management.

## 2. Server Prerequisites

Ensure your server has the necessary software installed:

- **Web Server**: Nginx or Apache
- **PHP**: PHP 8.x (matching your Laravel version) + extensions (bcmath, ctype, fileinfo, json, mbstring, openssl, pdo, tokenizer, xml)
- **Database**: MySQL or MariaDB
- **Composer**: Dependency manager
- **Git**: Version control

## 3. Application Setup

Navigate to your web root (usually `/var/www/html` or `/var/www`) and clone your repository.

```bash
cd /var/www
sudo mkdir -p my-app
sudo chown deployer:www-data my-app

# Switch to user deployer if not already
su - deployer
cd /var/www/my-app

# Clone your repository (ensure the directory is empty or clone into it)
git clone git@github.com:username/repo.git .
```

## 4. Installing Dependencies

Install PHP dependencies using Composer.

```bash
composer install --optimize-autoloader --no-dev
```

## 5. Permissions & Ownership

This is a critical security step. We implement a split ownership model:
- **Codebase**: Owned by `deployer:www-data`. This allows the deploy user to modify files, while the web server (group `www-data`) can read them.
- **Writable Directories**: Owned by `www-data:www-data`. This allows the web server (PHP-FPM) to write to logs and caches.

### Apply Ownership

```bash
# 1. Set base ownership for the entire project to deployer:www-data
sudo chown -R deployer:www-data /var/www/my-app

# 2. Set specific ownership for writable directories to www-data:www-data
sudo chown -R www-data:www-data /var/www/my-app/storage
sudo chown -R www-data:www-data /var/www/my-app/bootstrap/cache
```

### Apply Permissions

Ensure the directories are writable by the group (`www-data`).

```bash
# Standard files: 644 (Owner read/write, Group read, Others read)
find /var/www/my-app -type f -exec chmod 644 {} \;

# Standard directories: 755 (Owner r/w/x, Group r/x, Others r/x)
find /var/www/my-app -type d -exec chmod 755 {} \;

# Writable directories: Ensure group has write access
# storage and bootstrap/cache need to be writable by the web server
sudo chmod -R 775 /var/www/my-app/storage
sudo chmod -R 775 /var/www/my-app/bootstrap/cache
```

### Set Group ID (SGID)

It is highly recommended to set the SGID bit on writable directories. This ensures that any new files created within these directories (e.g., by the `deployer` user running artisan commands) inherit the `www-data` group, keeping them writable by the web server.

```bash
sudo chmod -R g+s /var/www/my-app/storage
sudo chmod -R g+s /var/www/my-app/bootstrap/cache
```

## 6. Environment Configuration

Set up your `.env` file.

```bash
cp .env.example .env
nano .env
```

Update your database credentials and other settings. Then generate the application key:

```bash
php artisan key:generate
```

## 7. Finalizing Deployment

Run database migrations and cache configurations.

```bash
# Run migrations
php artisan migrate --force

# Cache configurations (production only)
php artisan config:cache
php artisan event:cache
php artisan route:cache
php artisan view:cache
```

## 8. Web Server Configuration (Nginx Example)

Create a new Nginx server block: `/etc/nginx/sites-available/my-app`

```nginx
server {
    listen 80;
    server_name example.com;
    root /var/www/my-app/public;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";

    index index.php;

    charset utf-8;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    error_page 404 /index.php;

    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.2-fpm.sock; # Adjust PHP version
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

Enable the site and restart Nginx:

```bash
sudo ln -s /etc/nginx/sites-available/my-app /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

## Summary of Permissions

| Path | Owner | Group | Permissions | Purpose |
|------|-------|-------|-------------|---------|
| `/var/www/my-app` | `deployer` | `www-data` | `755` | Application code (read-only for web server) |
| `/var/www/my-app/storage` | `www-data` | `www-data` | `775` + `g+s` | Logs, sessions, compiled views (writable by web server) |
| `/var/www/my-app/bootstrap/cache` | `www-data` | `www-data` | `775` + `g+s` | Framework cache (writable by web server) |
