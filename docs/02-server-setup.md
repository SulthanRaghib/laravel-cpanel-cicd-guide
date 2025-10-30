# 02 - Server Setup

## Persiapan Server cPanel

### 1. Install Composer Lokal

Karena shared hosting biasanya tidak memiliki composer global, kita install secara lokal:

```bash
# Login SSH ke cPanel
ssh username@your-cpanel-host.com

# Masuk ke direktori project
cd ~/public_html

# Download Composer
curl -sS https://getcomposer.org/installer | php

# Verify instalasi
php composer.phar --version
```

**Output yang diharapkan:**

```
Composer version 2.x.x
```

### 2. Handle proc_open Restriction

Shared hosting sering mendisable fungsi `proc_open`. Solusinya:

```bash
# Install dependencies dengan skip post-install scripts
php composer.phar install --no-dev --optimize-autoloader --no-scripts
```

Setelah instalasi, jalankan manual:

```bash
# Clear cache
php artisan config:clear
php artisan cache:clear

# Discover packages (optional, boleh gagal)
php artisan package:discover --ansi || true
```
