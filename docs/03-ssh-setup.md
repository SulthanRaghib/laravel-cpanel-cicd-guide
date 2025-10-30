# 03 - SSH Setup

## Setup SSH Key untuk GitHub

### 1. Generate SSH Key Baru

Generate SSH key khusus untuk deployment (tanpa passphrase untuk automation):

```bash
ssh-keygen -t rsa -b 4096 -C "cpanel-deploy" -f ~/.ssh/id_rsa_deploy -N ""
```

**Parameter:**

- `-t rsa`: Tipe key RSA
- `-b 4096`: 4096 bit (lebih aman)
- `-C "cpanel-deploy"`: Label/comment
- `-f ~/.ssh/id_rsa_deploy`: Nama file custom
- `-N ""`: Tanpa passphrase (penting untuk automation)

### 2. Lihat Public Key

```bash
cat ~/.ssh/id_rsa_deploy.pub
```

Copy seluruh output (dimulai dengan `ssh-rsa AAAA...`)

### 3. Tambahkan ke GitHub

1. Buka GitHub → **Settings** (pojok kanan atas)
2. **SSH and GPG keys** (sidebar kiri)
3. **New SSH key**
4. **Title**: `cPanel Deploy Key`
5. **Key**: Paste public key dari langkah sebelumnya
6. **Add SSH key**

### 4. Konfigurasi SSH Config

Buat atau edit file SSH config:

```bash
nano ~/.ssh/config
```

Tambahkan konfigurasi:

```
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_rsa_deploy
    IdentitiesOnly yes
```

**Save:** Ctrl+X, Y, Enter

### 5. Set Permission

```bash
chmod 600 ~/.ssh/id_rsa_deploy
chmod 644 ~/.ssh/id_rsa_deploy.pub
chmod 600 ~/.ssh/config
```

### 6. Test Koneksi

```bash
ssh -T git@github.com
```

**Output yang diharapkan:**

```
Hi username! You've successfully authenticated, but GitHub does not provide shell access.
```

---

## Clone Repository ke cPanel

### 1. Tentukan Lokasi Project

```bash
# Untuk domain utama
cd ~/public_html

# Atau untuk subdomain/addon domain
cd ~/yourdomain.com
```

### 2. Clone Repository

```bash
# Clone dengan SSH (recommended)
git clone git@github.com:username/repository-name.git .
```

⚠️ **Perhatikan titik (.) di akhir** - ini penting untuk clone ke current directory

### 3. Verify Clone

```bash
ls -la
```

Pastikan ada file Laravel seperti: `artisan`, `composer.json`, `.env.example`, dll.

---

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
