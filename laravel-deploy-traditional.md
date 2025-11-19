# Deploying Laravel to a Traditional Server

This guide documents the process of deploying a Laravel application to a traditional Linux server (e.g., Ubuntu/Debian), ensuring a secure and standard setup.

## 1. User Management

You can use your default non-root user (e.g., `ubuntu`, `forge`, or your own user) to manage the deployment process. We will configure the permissions so that the web server user (`www-data`) owns the application code.

Ensure your user has `sudo` privileges.

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
# Create directory and assign temporary ownership to your current user for cloning
sudo mkdir -p my-app
sudo chown $USER:www-data my-app

cd /var/www/my-app

# Clone your repository (ensure the directory is empty or clone into it)
git clone git@github.com:username/repo.git .
```

## 4. Installing Dependencies

Install PHP dependencies using Composer. Since we haven't changed ownership to `www-data` yet, you can run this as your current user.

```bash
sudo -u www-data php composer.phar install --optimize-autoloader --no-dev
```

## 5. Permissions & Ownership

We will set the entire application to be owned by `www-data:www-data`. This is the standard approach for PHP-FPM setups where the web server acts as the owner of the files.

### Apply Ownership

```bash
# Set ownership for the entire project to www-data:www-data
sudo chown -R www-data:www-data /var/www/my-app
```

### Apply Permissions

Ensure standard permissions are applied.

```bash
# Standard files: 644 (Owner read/write, Group read, Others read)
find /var/www/my-app -type f -exec chmod 644 {} \;

# Standard directories: 755 (Owner r/w/x, Group r/x, Others r/x)
find /var/www/my-app -type d -exec chmod 755 {} \;
```

## 6. Environment Configuration

Set up your `.env` file.

```bash
cp .env.example .env
nano .env
```

Update your database credentials and other settings. Then generate the application key. Since ownership is now `www-data`, we run artisan commands as that user.

```bash
sudo -u www-data php artisan key:generate
```

## 7. Finalizing Deployment

Run database migrations and cache configurations.

**Important:** Always run Artisan commands that generate files (like `optimize` or `migrate`) as `www-data` to ensure the generated files have the correct ownership.

```bash
# Run migrations
sudo -u www-data php artisan migrate --force

# Cache configurations (production only)
sudo -u www-data php artisan optimize
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
| `/var/www/my-app` | `www-data` | `www-data` | `755` | Application code |
| `/var/www/my-app/storage` | `www-data` | `www-data` | `775` | Logs, sessions, compiled views |
| `/var/www/my-app/bootstrap/cache` | `www-data` | `www-data` | `775` | Framework cache |
