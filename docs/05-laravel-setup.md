## Setup Laravel Environment

### 1. Install Dependencies

```bash
php composer.phar install --no-dev --optimize-autoloader --no-scripts
```

### 2. Setup Environment File

```bash
# Copy .env.example
cp .env.example .env

# Edit .env
nano .env
```

**Konfigurasi minimal:**

```env
APP_NAME="Your App Name"
APP_ENV=production
APP_DEBUG=false
APP_URL=https://yourdomain.com

DB_CONNECTION=mysql
DB_HOST=localhost
DB_PORT=3306
DB_DATABASE=your_database_name
DB_USERNAME=your_database_user
DB_PASSWORD=your_database_password
```

⚠️ **PENTING:**

- `APP_DEBUG=false` untuk production
- `APP_ENV=production`
- Database credentials dari cPanel MySQL

**Save:** Ctrl+X, Y, Enter

### 3. Generate Application Key

```bash
php artisan key:generate
```

### 4. Setup Storage & Permissions

```bash
# Create symbolic link
php artisan storage:link

# Set permissions
chmod -R 755 storage bootstrap/cache
chmod -R 775 storage
```

### 5. Run Migrations

```bash
php artisan migrate --force
```

### 6. Optimize Application

```bash
php artisan config:cache
php artisan route:cache
php artisan view:cache
```

### 7. Setup Document Root

**Untuk domain utama:**

Laravel harus diakses dari folder `public`, bukan root. Di cPanel:

1. **Domains** → Klik domain Anda
2. **Document Root**: Ubah ke `/home/username/public_html/public`
3. **Save**

**Atau gunakan .htaccess redirect** (jika tidak bisa edit document root):

Edit `~/public_html/.htaccess`:

```apache
RewriteEngine on
RewriteBase /

RewriteCond %{HTTP_HOST} ^yourdomain\.com$ [NC]
RewriteRule ^(.*)$ public/$1 [L]
```
