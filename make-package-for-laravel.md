# Creating Production-Ready Laravel Packages: Complete Guide

## Table of Contents
1. [Introduction](#introduction)
2. [Prerequisites](#prerequisites)
3. [Package Structure](#package-structure)
4. [Initial Setup](#initial-setup)
5. [Composer Configuration](#composer-configuration)
6. [Service Provider](#service-provider)
7. [Workbench Setup](#workbench-setup)
8. [PHPUnit Testing](#phpunit-testing)
9. [Configuration Files](#configuration-files)
10. [Package Features](#package-features)
11. [Publishing Assets](#publishing-assets)
12. [Documentation](#documentation)
13. [Production Readiness](#production-readiness)
14. [Best Practices](#best-practices)
15. [Publishing to Packagist](#publishing-to-packagist)

## Introduction

Laravel packages are reusable pieces of functionality that can be shared across multiple Laravel applications. Creating a package allows you to:

- Share code between projects
- Contribute to the Laravel ecosystem
- Build commercial products
- Maintain clean, modular applications

This guide covers creating production-ready packages for Laravel 11+ with modern tooling.

## Prerequisites

- PHP 8.1+
- Composer
- Laravel 11+ knowledge
- Git
- Basic understanding of PSR-4 autoloading

## Package Structure

A well-structured Laravel package follows these conventions:

```
your-package/
├── .github/
│   └── workflows/
│       └── tests.yml
├── config/
│   └── your-package.php
├── database/
│   ├── migrations/
│   └── factories/
├── resources/
│   ├── views/
│   └── assets/
├── routes/
│   ├── web.php
│   └── api.php
├── src/
│   ├── Console/
│   │   └── Commands/
│   ├── Http/
│   │   ├── Controllers/
│   │   ├── Middleware/
│   │   └── Requests/
│   ├── Models/
│   ├── Providers/
│   │   └── YourPackageServiceProvider.php
│   ├── Services/
│   └── YourPackageClass.php
├── tests/
│   ├── Feature/
│   ├── Unit/
│   └── TestCase.php
├── workbench/
├── .gitignore
├── CHANGELOG.md
├── composer.json
├── LICENSE
├── phpunit.xml
└── README.md
```

## Initial Setup

### 1. Create Package Directory

```bash
mkdir your-vendor-name-package-name
cd your-vendor-name-package-name
```

### 2. Initialize Git Repository

```bash
git init
echo "vendor/
.phpunit.result.cache
.DS_Store
.idea/
.vscode/
*.log" > .gitignore
```

### 3. Create Basic Directory Structure

```bash
mkdir -p src/Providers
mkdir -p src/Http/Controllers
mkdir -p src/Http/Middleware
mkdir -p src/Http/Requests
mkdir -p src/Models
mkdir -p src/Services
mkdir -p src/Console/Commands
mkdir -p config
mkdir -p database/migrations
mkdir -p database/factories
mkdir -p resources/views
mkdir -p resources/assets
mkdir -p routes
mkdir -p tests/Feature
mkdir -p tests/Unit
```

## Composer Configuration

Create `composer.json`:

```json
{
    "name": "your-vendor/your-package",
    "description": "A comprehensive Laravel package",
    "type": "library",
    "license": "MIT",
    "authors": [
        {
            "name": "Your Name",
            "email": "your.email@example.com"
        }
    ],
    "require": {
        "php": "^8.1",
        "laravel/framework": "^11.0"
    },
    "require-dev": {
        "laravel/pint": "^1.0",
        "nunomaduro/collision": "^7.0",
        "orchestra/testbench": "^9.0",
        "pestphp/pest": "^2.0",
        "pestphp/pest-plugin-laravel": "^2.0",
        "phpunit/phpunit": "^10.0",
        "spatie/laravel-ray": "^1.26"
    },
    "autoload": {
        "psr-4": {
            "YourVendor\\YourPackage\\": "src/"
        }
    },
    "autoload-dev": {
        "psr-4": {
            "YourVendor\\YourPackage\\Tests\\": "tests/",
            "Workbench\\App\\": "workbench/app/",
            "Workbench\\Database\\Factories\\": "workbench/database/factories/",
            "Workbench\\Database\\Seeders\\": "workbench/database/seeders/"
        }
    },
    "scripts": {
        "post-autoload-dump": "@composer run prepare",
        "clear": "@php vendor/bin/testbench package:purge-skeleton --ansi",
        "prepare": "@php vendor/bin/testbench package:discover --ansi",
        "build": [
            "@composer run prepare",
            "@php vendor/bin/testbench workbench:build --ansi"
        ],
        "start": [
            "Composer\\Config::disableProcessTimeout",
            "@composer run build",
            "@php vendor/bin/testbench serve"
        ],
        "analyse": "vendor/bin/phpstan analyse",
        "test": "vendor/bin/pest",
        "test-coverage": "vendor/bin/pest --coverage",
        "format": "vendor/bin/pint"
    },
    "config": {
        "sort-packages": true,
        "allow-plugins": {
            "pestphp/pest-plugin": true,
            "phpstan/extension-installer": true
        }
    },
    "extra": {
        "laravel": {
            "providers": [
                "YourVendor\\YourPackage\\Providers\\YourPackageServiceProvider"
            ],
            "aliases": {
                "YourPackage": "YourVendor\\YourPackage\\Facades\\YourPackage"
            }
        }
    },
    "minimum-stability": "dev",
    "prefer-stable": true
}
```

## Service Provider

Create `src/Providers/YourPackageServiceProvider.php`:

```php
<?php

namespace YourVendor\YourPackage\Providers;

use Illuminate\Support\ServiceProvider;
use YourVendor\YourPackage\Console\Commands\YourPackageCommand;

class YourPackageServiceProvider extends ServiceProvider
{
    /**
     * Register any application services.
     */
    public function register(): void
    {
        $this->mergeConfigFrom(
            __DIR__.'/../../config/your-package.php',
            'your-package'
        );

        $this->app->singleton('your-package', function ($app) {
            return new \YourVendor\YourPackage\YourPackageClass();
        });
    }

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        $this->loadViewsFrom(__DIR__.'/../../resources/views', 'your-package');
        $this->loadMigrationsFrom(__DIR__.'/../../database/migrations');
        $this->loadRoutesFrom(__DIR__.'/../../routes/web.php');

        if ($this->app->runningInConsole()) {
            $this->publishes([
                __DIR__.'/../../config/your-package.php' => config_path('your-package.php'),
            ], 'your-package-config');

            $this->publishes([
                __DIR__.'/../../resources/views' => resource_path('views/vendor/your-package'),
            ], 'your-package-views');

            $this->publishes([
                __DIR__.'/../../database/migrations/' => database_path('migrations'),
            ], 'your-package-migrations');

            $this->commands([
                YourPackageCommand::class,
            ]);
        }
    }
}
```

## Workbench Setup

### 1. Install Testbench

```bash
composer require --dev orchestra/testbench
```

### 2. Create Workbench Configuration

Create `workbench/bootstrap/app.php`:

```php
<?php

use Illuminate\Foundation\Application;
use Illuminate\Foundation\Configuration\Exceptions;
use Illuminate\Foundation\Configuration\Middleware;

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        commands: __DIR__.'/../routes/console.php',
        health: '/up',
    )
    ->withMiddleware(function (Middleware $middleware) {
        //
    })
    ->withExceptions(function (Exceptions $exceptions) {
        //
    })->create();
```

### 3. Create Workbench Configuration File

Create `workbench/config/app.php`:

```php
<?php

return [
    'name' => 'Your Package Workbench',
    'env' => 'local',
    'debug' => true,
    'url' => env('APP_URL', 'http://localhost'),
    'timezone' => 'UTC',
    'locale' => 'en',
    'fallback_locale' => 'en',
    'faker_locale' => 'en_US',
    'key' => env('APP_KEY', 'base64:'.base64_encode(random_bytes(32))),
    'cipher' => 'AES-256-CBC',
    'providers' => [
        YourVendor\YourPackage\Providers\YourPackageServiceProvider::class,
    ],
];
```

### 4. Initialize Workbench

```bash
composer run prepare
```

## PHPUnit Testing

### 1. Create PHPUnit Configuration

Create `phpunit.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<phpunit xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="vendor/phpunit/phpunit/phpunit.xsd"
         bootstrap="vendor/autoload.php"
         colors="true">
    <testsuites>
        <testsuite name="Unit">
            <directory suffix="Test.php">./tests/Unit</directory>
        </testsuite>
        <testsuite name="Feature">
            <directory suffix="Test.php">./tests/Feature</directory>
        </testsuite>
    </testsuites>
    <coverage>
        <include>
            <directory suffix=".php">./src</directory>
        </include>
    </coverage>
    <php>
        <env name="APP_ENV" value="testing"/>
        <env name="BCRYPT_ROUNDS" value="4"/>
        <env name="CACHE_DRIVER" value="array"/>
        <env name="DB_CONNECTION" value="sqlite"/>
        <env name="DB_DATABASE" value=":memory:"/>
        <env name="MAIL_MAILER" value="array"/>
        <env name="QUEUE_CONNECTION" value="sync"/>
        <env name="SESSION_DRIVER" value="array"/>
        <env name="TELESCOPE_ENABLED" value="false"/>
    </php>
</phpunit>
```

### 2. Create Base Test Case

Create `tests/TestCase.php`:

```php
<?php

namespace YourVendor\YourPackage\Tests;

use Orchestra\Testbench\TestCase as Orchestra;
use YourVendor\YourPackage\Providers\YourPackageServiceProvider;

class TestCase extends Orchestra
{
    protected function setUp(): void
    {
        parent::setUp();

        // Additional setup if needed
    }

    protected function getPackageProviders($app): array
    {
        return [
            YourPackageServiceProvider::class,
        ];
    }

    protected function defineEnvironment($app): void
    {
        $app['config']->set('database.default', 'sqlite');
        $app['config']->set('database.connections.sqlite', [
            'driver' => 'sqlite',
            'database' => ':memory:',
            'prefix' => '',
        ]);
    }

    protected function defineDatabaseMigrations(): void
    {
        $this->loadLaravelMigrations();
        $this->loadMigrationsFrom(__DIR__.'/../database/migrations');
    }
}
```

### 3. Create Sample Tests

Create `tests/Unit/YourPackageTest.php`:

```php
<?php

namespace YourVendor\YourPackage\Tests\Unit;

use YourVendor\YourPackage\Tests\TestCase;
use YourVendor\YourPackage\YourPackageClass;

class YourPackageTest extends TestCase
{
    /** @test */
    public function it_can_be_instantiated(): void
    {
        $package = new YourPackageClass();
        
        $this->assertInstanceOf(YourPackageClass::class, $package);
    }

    /** @test */
    public function it_can_perform_basic_functionality(): void
    {
        $package = new YourPackageClass();
        
        // Add your actual tests here
        $this->assertTrue(true);
    }
}
```

Create `tests/Feature/ServiceProviderTest.php`:

```php
<?php

namespace YourVendor\YourPackage\Tests\Feature;

use YourVendor\YourPackage\Tests\TestCase;

class ServiceProviderTest extends TestCase
{
    /** @test */
    public function service_provider_is_loaded(): void
    {
        $this->assertTrue(
            $this->app->providerIsLoaded('YourVendor\YourPackage\Providers\YourPackageServiceProvider')
        );
    }

    /** @test */
    public function config_is_published(): void
    {
        $this->artisan('vendor:publish', [
            '--provider' => 'YourVendor\YourPackage\Providers\YourPackageServiceProvider',
            '--tag' => 'your-package-config',
        ])->assertExitCode(0);
    }
}
```

## Configuration Files

Create `config/your-package.php`:

```php
<?php

return [
    /*
    |--------------------------------------------------------------------------
    | Default Configuration
    |--------------------------------------------------------------------------
    |
    | This file contains the default configuration for your package.
    |
    */

    'enabled' => true,

    'cache' => [
        'enabled' => true,
        'ttl' => 3600,
    ],

    'api' => [
        'rate_limit' => 1000,
        'timeout' => 30,
    ],

    'features' => [
        'logging' => true,
        'metrics' => false,
    ],
];
```

## Package Features

### 1. Create Main Package Class

Create `src/YourPackageClass.php`:

```php
<?php

namespace YourVendor\YourPackage;

class YourPackageClass
{
    protected array $config;

    public function __construct()
    {
        $this->config = config('your-package', []);
    }

    public function isEnabled(): bool
    {
        return $this->config['enabled'] ?? true;
    }

    public function doSomething(): string
    {
        if (!$this->isEnabled()) {
            throw new \Exception('Package is disabled');
        }

        return 'Package is working!';
    }
}
```

### 2. Create Console Command

Create `src/Console/Commands/YourPackageCommand.php`:

```php
<?php

namespace YourVendor\YourPackage\Console\Commands;

use Illuminate\Console\Command;

class YourPackageCommand extends Command
{
    protected $signature = 'your-package:install';
    protected $description = 'Install and setup your package';

    public function handle(): void
    {
        $this->info('Installing Your Package...');

        $this->call('vendor:publish', [
            '--provider' => 'YourVendor\YourPackage\Providers\YourPackageServiceProvider',
            '--tag' => 'your-package-config',
        ]);

        $this->info('Package installed successfully!');
    }
}
```

### 3. Create Facade

Create `src/Facades/YourPackage.php`:

```php
<?php

namespace YourVendor\YourPackage\Facades;

use Illuminate\Support\Facades\Facade;

class YourPackage extends Facade
{
    protected static function getFacadeAccessor(): string
    {
        return 'your-package';
    }
}
```

### 4. Create Controllers

Create `src/Http/Controllers/YourPackageController.php`:

```php
<?php

namespace YourVendor\YourPackage\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Routing\Controller;
use YourVendor\YourPackage\YourPackageClass;

class YourPackageController extends Controller
{
    public function __construct(
        protected YourPackageClass $package
    ) {}

    public function index()
    {
        return view('your-package::index', [
            'status' => $this->package->doSomething(),
        ]);
    }
}
```

### 5. Create Views

Create `resources/views/index.blade.php`:

```blade
<!DOCTYPE html>
<html>
<head>
    <title>Your Package</title>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
</head>
<body>
    <div class="container">
        <h1>Your Package</h1>
        <p>Status: {{ $status }}</p>
    </div>
</body>
</html>
```

### 6. Create Routes

Create `routes/web.php`:

```php
<?php

use Illuminate\Support\Facades\Route;
use YourVendor\YourPackage\Http\Controllers\YourPackageController;

Route::prefix('your-package')
    ->name('your-package.')
    ->group(function () {
        Route::get('/', [YourPackageController::class, 'index'])->name('index');
    });
```

## Publishing Assets

Update your service provider to handle asset publishing:

```php
public function boot(): void
{
    // ... existing code ...

    if ($this->app->runningInConsole()) {
        // Publish assets
        $this->publishes([
            __DIR__.'/../../resources/assets' => public_path('vendor/your-package'),
        ], 'your-package-assets');

        // Publish language files
        $this->publishes([
            __DIR__.'/../../resources/lang' => $this->app->langPath('vendor/your-package'),
        ], 'your-package-lang');
    }
}
```

## Documentation

### 1. Create README.md

```markdown
# Your Package

A comprehensive Laravel package for [purpose].

## Installation

Install via Composer:

```bash
composer require your-vendor/your-package
```

## Configuration

Publish the configuration file:

```bash
php artisan vendor:publish --provider="YourVendor\YourPackage\Providers\YourPackageServiceProvider" --tag="your-package-config"
```

## Usage

### Basic Usage

```php
use YourVendor\YourPackage\Facades\YourPackage;

$result = YourPackage::doSomething();
```

### Advanced Usage

[Add more detailed examples]

## Testing

```bash
composer test
```

## Security

If you discover any security-related issues, please email your.email@example.com instead of using the issue tracker.

## License

The MIT License (MIT). Please see [License File](LICENSE.md) for more information.
```

### 2. Create CHANGELOG.md

```markdown
# Changelog

All notable changes to `your-package` will be documented in this file.

## [Unreleased]

### Added
- Initial package structure
- Basic functionality
- Comprehensive tests

### Changed

### Deprecated

### Removed

### Fixed

### Security
```

## Production Readiness

### 1. Create GitHub Actions Workflow

Create `.github/workflows/tests.yml`:

```yaml
name: Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: true
      matrix:
        os: [ubuntu-latest]
        php: [8.1, 8.2, 8.3]
        laravel: [11.*]
        stability: [prefer-lowest, prefer-stable]

    name: P${{ matrix.php }} - L${{ matrix.laravel }} - ${{ matrix.stability }} - ${{ matrix.os }}

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: ${{ matrix.php }}
          extensions: dom, curl, libxml, mbstring, zip, pcntl, pdo, sqlite, pdo_sqlite, bcmath, soap, intl, gd, exif, iconv, imagick, fileinfo
          coverage: none

      - name: Setup problem matchers
        run: |
          echo "::add-matcher::${{ runner.tool_cache }}/php.json"
          echo "::add-matcher::${{ runner.tool_cache }}/phpunit.json"

      - name: Install dependencies
        run: |
          composer require "laravel/framework:${{ matrix.laravel }}" --no-interaction --no-update
          composer update --${{ matrix.stability }} --prefer-dist --no-interaction

      - name: List Installed Dependencies
        run: composer show -D

      - name: Execute tests
        run: vendor/bin/pest

  coverage:
    runs-on: ubuntu-latest
    name: Coverage

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: 8.2
          extensions: dom, curl, libxml, mbstring, zip, pcntl, pdo, sqlite, pdo_sqlite, bcmath, soap, intl, gd, exif, iconv, imagick, fileinfo
          coverage: xdebug

      - name: Install dependencies
        run: composer update --prefer-dist --no-interaction

      - name: Execute tests
        run: vendor/bin/pest --coverage --min=80
```

### 2. Create Release Script

Create `scripts/release.sh`:

```bash
#!/bin/bash

# Get version argument
if [ -z "$1" ]; then
    echo "Usage: ./scripts/release.sh <version>"
    exit 1
fi

VERSION=$1

# Validate version format
if [[ ! $VERSION =~ ^[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
    echo "Version must be in format x.y.z"
    exit 1
fi

# Update changelog
echo "Updating CHANGELOG.md..."
sed -i "s/## \[Unreleased\]/## [$VERSION] - $(date +%Y-%m-%d)/" CHANGELOG.md

# Commit changes
git add CHANGELOG.md
git commit -m "Release v$VERSION"

# Create tag
git tag -a "v$VERSION" -m "Release v$VERSION"

# Push changes and tags
git push origin main
git push origin "v$VERSION"

echo "Released v$VERSION successfully!"
```

### 3. Security Considerations

Create `.github/workflows/security.yml`:

```yaml
name: Security

on:
  schedule:
    - cron: '0 0 * * 0'
  push:
    branches: [main]

jobs:
  security:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: 8.2

      - name: Install dependencies
        run: composer install --no-dev

      - name: Run security audit
        run: composer audit
```

## Best Practices

### 1. Code Quality Tools

Add to `composer.json`:

```json
{
    "require-dev": {
        "phpstan/phpstan": "^1.10",
        "laravel/pint": "^1.0",
        "rector/rector": "^0.18"
    },
    "scripts": {
        "analyse": "vendor/bin/phpstan analyse",
        "format": "vendor/bin/pint",
        "refactor": "vendor/bin/rector process"
    }
}
```

### 2. Create PHPStan Configuration

Create `phpstan.neon`:

```neon
parameters:
    level: 8
    paths:
        - src
    ignoreErrors:
        - '#Call to an undefined method Illuminate\\Foundation\\Application::#'
```

### 3. Create Pint Configuration

Create `pint.json`:

```json
{
    "preset": "laravel",
    "rules": {
        "simplified_null_return": true,
        "braces": false,
        "new_with_braces": {
            "anonymous_class": false,
            "named_class": false
        }
    }
}
```

### 4. Error Handling

Create `src/Exceptions/YourPackageException.php`:

```php
<?php

namespace YourVendor\YourPackage\Exceptions;

use Exception;

class YourPackageException extends Exception
{
    public static function configurationMissing(string $key): self
    {
        return new self("Configuration key '{$key}' is missing.");
    }

    public static function featureDisabled(string $feature): self
    {
        return new self("Feature '{$feature}' is disabled.");
    }
}
```

## Publishing to Packagist

### 1. Create Account

1. Visit [packagist.org](https://packagist.org/)
2. Create an account
3. Connect your GitHub account

### 2. Submit Package

1. Click "Submit"
2. Enter your GitHub repository URL
3. Click "Check"
4. Review and submit

### 3. Setup Auto-Update

1. Go to your package on Packagist
2. Click "Settings"
3. Enable "GitHub Hook" or "API Token"

### 4. Package Validation

Ensure your package meets Packagist requirements:

- Valid `composer.json`
- Proper PSR-4 autoloading
- Valid LICENSE file
- Tagged releases
- Good README

## Final Checklist

Before releasing your package:

- [ ] All tests pass
- [ ] Code coverage above 80%
- [ ] Documentation is complete
- [ ] CHANGELOG.md is updated
- [ ] Security audit passes
- [ ] Code style is consistent
- [ ] Version is tagged
- [ ] GitHub Actions workflows work
- [ ] Package is published to Packagist
- [ ] README has installation instructions
- [ ] Configuration is well documented

## Running the Development Environment

```bash
# Install dependencies
composer install

# Run tests
composer test

# Start workbench
composer start

# Format code
composer format

# Analyze code
composer analyse
```

Your package will be available at `http://localhost:8000/your-package` when using the workbench.

---

This guide provides a comprehensive foundation for creating production-ready Laravel packages. Customize the structure and configuration based on your specific needs while maintaining these best practices.
