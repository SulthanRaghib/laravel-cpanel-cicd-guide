# CI/CD Setup untuk Laravel di cPanel

Dokumentasi lengkap untuk mengatur Continuous Integration/Continuous Deployment (CI/CD) otomatis untuk aplikasi Laravel di shared hosting cPanel menggunakan GitHub Webhooks.

---

## 📋 Table of Contents

1. [Prerequisites](#prerequisites)
2. [Persiapan Server cPanel](#persiapan-server-cpanel)
3. [Setup SSH Key untuk GitHub](#setup-ssh-key-untuk-github)
4. [Clone Repository ke cPanel](#clone-repository-ke-cpanel)
5. [Setup Laravel Environment](#setup-laravel-environment)
6. [Buat Deploy Script](#buat-deploy-script)
7. [Buat Webhook Handler](#buat-webhook-handler)
8. [Setup GitHub Webhook](#setup-github-webhook)
9. [Security Configuration](#security-configuration)
10. [Testing & Monitoring](#testing--monitoring)
11. [Troubleshooting](#troubleshooting)

---

## Prerequisites

Sebelum memulai, pastikan Anda memiliki:

- ✅ Akses SSH ke cPanel
- ✅ Repository GitHub dengan project Laravel
- ✅ Database MySQL sudah dibuat di cPanel
- ✅ Domain/subdomain sudah dikonfigurasi
- ✅ PHP 8.1 atau lebih tinggi (untuk Laravel 10+)

---

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

---

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

---

## Buat Deploy Script

### 1. Generate Script

```bash
cd ~/public_html
nano deploy.sh
```

### 2. Paste Script Berikut

```bash
#!/bin/bash

# =========================================
# Laravel Auto Deploy Script for cPanel
# Compatible with proc_open disabled
# =========================================

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m'

# =========================================
# CONFIGURATION - EDIT THESE VALUES
# =========================================

PROJECT_PATH="/home/YOUR_USERNAME/public_html"
BRANCH="main"  # Change to 'master' if needed

# =========================================
# DO NOT EDIT BELOW THIS LINE
# =========================================

LOG_FILE="$PROJECT_PATH/deployment.log"
COMPOSER="php $PROJECT_PATH/composer.phar"

# Functions
log_message() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" >> "$LOG_FILE"
    echo -e "${2}$1${NC}"
}

# Start deployment
echo -e "${BLUE}========================================${NC}"
log_message "🚀 Starting Deployment..." "$GREEN"
echo -e "${BLUE}========================================${NC}"

# Go to project directory
cd "$PROJECT_PATH" || exit 1

# Put application in maintenance mode
log_message "📝 Enabling maintenance mode..." "$YELLOW"
php artisan down --retry=60 2>&1 || log_message "⚠️  Maintenance mode skipped" "$YELLOW"

# Pull latest code from GitHub
log_message "📥 Pulling latest code from GitHub..." "$YELLOW"
git fetch origin 2>&1 | tee -a "$LOG_FILE"
git reset --hard origin/$BRANCH 2>&1 | tee -a "$LOG_FILE"

if [ ${PIPESTATUS[0]} -ne 0 ]; then
    log_message "❌ Git pull failed!" "$RED"
    php artisan up 2>&1 || true
    exit 1
fi

# Install/Update Composer dependencies
log_message "📦 Installing Composer dependencies..." "$YELLOW"
$COMPOSER install --no-dev --no-interaction --prefer-dist --optimize-autoloader --no-scripts 2>&1 | tee -a "$LOG_FILE"

if [ ${PIPESTATUS[0]} -ne 0 ]; then
    log_message "❌ Composer install failed!" "$RED"
    php artisan up 2>&1 || true
    exit 1
fi

# Package discovery (optional, may fail due to proc_open)
log_message "🔍 Discovering packages..." "$YELLOW"
php artisan package:discover --ansi 2>&1 | tee -a "$LOG_FILE" || log_message "⚠️  Package discovery skipped" "$YELLOW"

# Run database migrations
log_message "🗄️  Running database migrations..." "$YELLOW"
php artisan migrate --force 2>&1 | tee -a "$LOG_FILE"

# Clear all caches
log_message "🧹 Clearing caches..." "$YELLOW"
php artisan config:clear 2>&1 | tee -a "$LOG_FILE" || true
php artisan cache:clear 2>&1 | tee -a "$LOG_FILE" || true
php artisan route:clear 2>&1 | tee -a "$LOG_FILE" || true
php artisan view:clear 2>&1 | tee -a "$LOG_FILE" || true

# Optimize application
log_message "⚡ Optimizing application..." "$YELLOW"
php artisan config:cache 2>&1 | tee -a "$LOG_FILE" || true
php artisan route:cache 2>&1 | tee -a "$LOG_FILE" || true
php artisan view:cache 2>&1 | tee -a "$LOG_FILE" || true

# Set correct permissions
log_message "🔐 Setting permissions..." "$YELLOW"
chmod -R 755 storage bootstrap/cache 2>&1 | tee -a "$LOG_FILE"
chmod -R 775 storage 2>&1 | tee -a "$LOG_FILE"

# Bring application back online
log_message "✅ Disabling maintenance mode..." "$YELLOW"
php artisan up 2>&1 | tee -a "$LOG_FILE" || true

# Finish
echo -e "${BLUE}========================================${NC}"
log_message "✅ Deployment completed successfully!" "$GREEN"
echo -e "${BLUE}========================================${NC}"
echo ""
log_message "📊 Deployment Summary:" "$BLUE"
log_message "   Branch: $BRANCH" "$NC"
log_message "   Composer: $COMPOSER" "$NC"
log_message "   Time: $(date '+%Y-%m-%d %H:%M:%S')" "$NC"
log_message "   Path: $PROJECT_PATH" "$NC"
echo -e "${BLUE}========================================${NC}"
```

⚠️ **PENTING: Edit baris berikut:**

- `PROJECT_PATH="/home/YOUR_USERNAME/public_html"` - Ganti dengan path Anda
- `BRANCH="main"` - Ganti ke `"master"` jika branch utama Anda master

**Save:** Ctrl+X, Y, Enter

### 3. Beri Permission Eksekusi

```bash
chmod +x deploy.sh
```

### 4. Test Deploy Script

```bash
./deploy.sh
```

**Output yang diharapkan:**

```
========================================
🚀 Starting Deployment...
========================================
...
✅ Deployment completed successfully!
========================================
```

---

## Buat Webhook Handler

### 1. Generate Secret Token

```bash
openssl rand -hex 32
```

**Copy output token** - Anda akan membutuhkannya!

Contoh output:

```
a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7r8s9t0u1v2w3x4y5z6a7b8c9d0e1f2
```

### 2. Buat File deploy.php

```bash
cd ~/public_html/public
nano deploy.php
```

⚠️ **PENTING:** File harus di folder `public/` bukan root project!

### 3. Paste Script Berikut

```php
<?php
/**
 * GitHub Webhook Auto Deploy Handler
 * For Laravel on cPanel
 *
 * Security: Only GitHub IPs can access this endpoint
 * Token authentication required
 */

// =========================================
// CONFIGURATION - EDIT THESE VALUES
// =========================================

// Paste your token from: openssl rand -hex 32
define('SECRET_TOKEN', 'PASTE_YOUR_TOKEN_HERE');

define('PROJECT_PATH', '/home/YOUR_USERNAME/public_html');
define('BRANCH_TO_DEPLOY', 'main'); // Change to 'master' if needed

// =========================================
// DO NOT EDIT BELOW THIS LINE
// =========================================

define('DEPLOY_SCRIPT', PROJECT_PATH . '/deploy.sh');
define('LOG_FILE', PROJECT_PATH . '/webhook.log');
define('MAX_LOG_SIZE', 5 * 1024 * 1024); // 5MB

// =========================================
// FUNCTIONS
// =========================================

function writeLog($message, $data = null) {
    $timestamp = date('Y-m-d H:i:s');
    $logMessage = "[$timestamp] $message";

    if ($data !== null) {
        $logMessage .= "\n" . json_encode($data, JSON_PRETTY_PRINT | JSON_UNESCAPED_SLASHES);
    }

    $logMessage .= "\n" . str_repeat('-', 80) . "\n";

    // Rotate log if too large
    if (file_exists(LOG_FILE) && filesize(LOG_FILE) > MAX_LOG_SIZE) {
        rename(LOG_FILE, LOG_FILE . '.' . date('Ymd-His') . '.old');
    }

    file_put_contents(LOG_FILE, $logMessage, FILE_APPEND);
}

function sendResponse($status, $message, $data = null, $code = 200) {
    http_response_code($code);
    header('Content-Type: application/json');

    $response = [
        'status' => $status,
        'message' => $message,
        'timestamp' => date('Y-m-d H:i:s'),
    ];

    if ($data !== null) {
        $response['data'] = $data;
    }

    echo json_encode($response, JSON_PRETTY_PRINT);
    exit;
}

// =========================================
// SECURITY CHECK
// =========================================

// Check token exists
if (!isset($_GET['token'])) {
    writeLog('UNAUTHORIZED: No token provided', [
        'ip' => $_SERVER['REMOTE_ADDR'] ?? 'Unknown',
        'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? 'Unknown'
    ]);
    sendResponse('error', 'Unauthorized: Token required', null, 403);
}

// Verify token
if ($_GET['token'] !== SECRET_TOKEN) {
    writeLog('UNAUTHORIZED: Invalid token', [
        'ip' => $_SERVER['REMOTE_ADDR'] ?? 'Unknown',
        'provided' => substr($_GET['token'], 0, 10) . '...'
    ]);
    sendResponse('error', 'Unauthorized: Invalid token', null, 403);
}

// =========================================
// WEBHOOK PROCESSING
// =========================================

writeLog('✅ Webhook received - Token verified');

// Get payload
$payload = file_get_contents('php://input');
$data = json_decode($payload, true);

// Get GitHub headers
$event = $_SERVER['HTTP_X_GITHUB_EVENT'] ?? 'unknown';
$delivery = $_SERVER['HTTP_X_GITHUB_DELIVERY'] ?? 'unknown';

writeLog('📨 GitHub Event Received', [
    'event' => $event,
    'delivery_id' => $delivery
]);

// Only process push events
if ($event !== 'push') {
    writeLog("⏭️  Skipped: Not a push event", ['event' => $event]);
    sendResponse('skipped', "Event '$event' ignored. Only 'push' events trigger deployment.");
}

// Check branch
$ref = $data['ref'] ?? '';
$branch = str_replace('refs/heads/', '', $ref);

if ($branch !== BRANCH_TO_DEPLOY) {
    writeLog("⏭️  Skipped: Wrong branch", [
        'expected' => BRANCH_TO_DEPLOY,
        'received' => $branch
    ]);
    sendResponse('skipped', "Push to '$branch' ignored. Only '" . BRANCH_TO_DEPLOY . "' triggers deployment.");
}

// Get commit info
$commit = $data['head_commit'] ?? [];
$commitInfo = [
    'id' => substr($commit['id'] ?? 'unknown', 0, 7),
    'message' => $commit['message'] ?? 'No message',
    'author' => $commit['author']['name'] ?? 'Unknown',
    'timestamp' => $commit['timestamp'] ?? date('c'),
    'url' => $commit['url'] ?? ''
];

writeLog('🚀 Starting deployment', [
    'branch' => $branch,
    'commit' => $commitInfo
]);

// =========================================
// DEPLOYMENT
// =========================================

// Check deploy script exists
if (!file_exists(DEPLOY_SCRIPT)) {
    writeLog('❌ ERROR: Deploy script not found', ['path' => DEPLOY_SCRIPT]);
    sendResponse('error', 'Deploy script not found', ['path' => DEPLOY_SCRIPT], 500);
}

// Execute deployment
$startTime = microtime(true);
$output = shell_exec(DEPLOY_SCRIPT . ' 2>&1');
$executionTime = round(microtime(true) - $startTime, 2);

// Check success
$success = (strpos($output, '✅ Deployment completed successfully!') !== false);

if ($success) {
    writeLog('✅ Deployment SUCCESS', [
        'execution_time' => $executionTime . 's',
        'commit_id' => $commitInfo['id'],
        'author' => $commitInfo['author']
    ]);

    sendResponse('success', 'Deployment completed successfully!', [
        'execution_time' => $executionTime . 's',
        'commit' => $commitInfo,
        'branch' => $branch,
        'deployed_at' => date('Y-m-d H:i:s')
    ]);
} else {
    writeLog('⚠️  Deployment completed with warnings', [
        'execution_time' => $executionTime . 's',
        'output_preview' => substr($output, -500)
    ]);

    sendResponse('warning', 'Deployment executed, check logs for details', [
        'execution_time' => $executionTime . 's'
    ], 200);
}
```

⚠️ **PENTING: Edit nilai berikut:**

- `SECRET_TOKEN` - Paste token dari `openssl rand -hex 32`
- `PROJECT_PATH` - Path ke project Anda
- `BRANCH_TO_DEPLOY` - `'main'` atau `'master'`

**Save:** Ctrl+X, Y, Enter

### 4. Test Webhook Handler

```bash
# Ganti YOUR_TOKEN dengan token Anda
curl "https://yourdomain.com/deploy.php?token=YOUR_TOKEN" \
  -H "X-GitHub-Event: push" \
  -H "Content-Type: application/json" \
  -d '{"ref":"refs/heads/main","head_commit":{"id":"test","message":"Test","author":{"name":"Test"}}}'
```

**Expected response:**

```json
{
    "status": "success",
    "message": "Deployment completed successfully!",
    ...
}
```

---

## Setup GitHub Webhook

### 1. Buka Repository GitHub

Navigate ke repository Anda di GitHub.

### 2. Pergi ke Settings

Klik tab **Settings** di bagian atas repository.

### 3. Pilih Webhooks

Di sidebar kiri, klik **Webhooks** → **Add webhook**

### 4. Konfigurasi Webhook

**Payload URL:**

```
https://yourdomain.com/deploy.php?token=YOUR_SECRET_TOKEN
```

⚠️ Ganti `YOUR_SECRET_TOKEN` dengan token dari `openssl rand -hex 32`

**Content type:**

```
application/json
```

**Secret:**

```
(kosongkan)
```

**SSL verification:**

```
☑️ Enable SSL verification
```

**Which events would you like to trigger this webhook?**

```
☑️ Just the push event
```

**Active:**

```
☑️ Active
```

### 5. Add Webhook

Klik **Add webhook**

### 6. Verify Webhook

GitHub akan otomatis mengirim test ping.

1. Scroll ke bawah ke **Recent Deliveries**
2. Klik request pertama (ping)
3. Check **Response** tab
4. Harus ada response 200 (boleh "skipped" karena bukan push event)

---

## Security Configuration

### 1. Protect deploy.php dengan IP Whitelist

Edit `.htaccess` di folder `public/`:

```bash
nano ~/public_html/public/.htaccess
```

**Tambahkan di PALING ATAS** (sebelum semua rules):

```apache
# ============================================
# GitHub Webhook Security
# ============================================

# Only allow GitHub IPs to access deploy.php
<Files "deploy.php">
    # GitHub Webhook IP Ranges (update regularly)
    Require ip 140.82.112.0/20
    Require ip 143.55.64.0/20
    Require ip 185.199.108.0/22
    Require ip 192.30.252.0/22
    Require ip 2a0a:a440::/29
    Require ip 2606:50c0::/32
</Files>

# ============================================
```

⚠️ **Note:** GitHub IP ranges dapat berubah. Cek update di:
https://api.github.com/meta

### 2. Protect Sensitive Files

Tambahkan juga di `.htaccess`:

```apache
# ============================================
# Protect Sensitive Files
# ============================================

# Deny access to log files
<FilesMatch "\.(log)$">
    Require all denied
</FilesMatch>

# Deny access to shell scripts
<FilesMatch "\.sh$">
    Require all denied
</FilesMatch>

# Deny access to composer files
<FilesMatch "^composer\.(json|lock|phar)$">
    Require all denied
</FilesMatch>

# Deny access to git files
<FilesMatch "^\.git">
    Require all denied
</FilesMatch>

# Deny access to environment file
<Files ".env">
    Require all denied
</Files>

# ============================================
```

### 3. Protect deploy.sh

Pastikan `deploy.sh` tidak accessible dari web:

```bash
# deploy.sh ada di root project, bukan di public/
ls ~/public_html/deploy.sh

# Jika ada di public/, pindahkan!
mv ~/public_html/public/deploy.sh ~/public_html/
```

### 4. Set File Permissions

```bash
# Deploy script
chmod 700 ~/public_html/deploy.sh

# Webhook handler
chmod 644 ~/public_html/public/deploy.php

# Log files
chmod 644 ~/public_html/*.log
```

---

## Testing & Monitoring

### 1. Test Deployment

Di **local computer** Anda:

```bash
# Make a change
echo "# Test CI/CD" >> README.md

# Commit and push
git add .
git commit -m "test: CI/CD auto deployment"
git push origin main  # or master
```

### 2. Monitor Logs (Real-time)

Via SSH di cPanel:

```bash
# Watch webhook log
tail -f ~/public_html/webhook.log

# Watch deployment log
tail -f ~/public_html/deployment.log
```

Press `Ctrl+C` to stop monitoring.

### 3. Check Webhook Deliveries

Di GitHub:

1. **Settings** → **Webhooks** → Click your webhook
2. **Recent Deliveries** tab
3. Click latest request
4. Check **Response** tab (should be 200 with "success")

### 4. View Log History

```bash
# Last 50 lines
tail -50 ~/public_html/webhook.log

# Search for errors
grep -i "error" ~/public_html/deployment.log

# Search for success
grep -i "success" ~/public_html/webhook.log
```

---

## Troubleshooting

### ❌ Problem: 404 Not Found saat akses deploy.php

**Penyebab:** File deploy.php tidak di folder `public/`

**Solusi:**

```bash
# Pastikan file di folder public
ls -la ~/public_html/public/deploy.php

# Jika tidak ada, pindahkan
mv ~/public_html/deploy.php ~/public_html/public/
```

---

### ❌ Problem: 403 Forbidden saat test manual

**Penyebab:** IP Anda tidak di whitelist (ini normal!)

**Solusi:**

Untuk testing, sementara comment IP restriction:

```bash
nano ~/public_html/public/.htaccess
```

Comment baris ini:

```apache
# <Files "deploy.php">
#     Require ip ...
# </Files>
```

Setelah test, **uncomment kembali** untuk keamanan!

---

### ❌ Problem: Git pull failed

**Penyebab:** Git credentials atau SSH key bermasalah

**Solusi:**

```bash
# Test SSH connection
ssh -T git@github.com

# Should output: Hi username! You've successfully authenticated...

# Check git remote
cd ~/public_html
git remote -v

# Should use git@github.com, not https://
```

---

### ❌ Problem: Composer install failed

**Penyebab:** Memory limit atau composer.phar tidak ada

**Solusi:**

```bash
# Check composer exists
ls -la ~/public_html/composer.phar

# If not exists, download
cd ~/public_html
curl -sS https://getcomposer.org/installer | php

# Try with memory limit
php -d memory_limit=512M composer.phar install --no-dev --optimize-autoloader --no-scripts
```

---

### ❌ Problem: proc_open error

**Penyebab:** Shared hosting disable proc_open

**Solusi:**

Sudah di-handle di `deploy.sh` dengan flag `--no-scripts`.

Jika masih error:

```bash
# Skip package discovery
php artisan package:discover --ansi || true
```

---

### ❌ Problem: Permission denied errors

**Penyebab:** File permissions tidak benar

**Solusi:**

```bash
cd ~/public_html

# Fix storage permissions
chmod -R 755 storage bootstrap/cache
chmod -R 775 storage

# Fix deploy script
chmod 700 deploy.sh

# Fix ownership (if needed)
chown -R username:username storage
```

---

### ❌ Problem: Database migration failed

**Penyebab:** Database credentials salah atau database tidak ada

**Solusi:**

```bash
# Check .env
cat ~/public_html/.env | grep DB_

# Test database connection
php artisan tinker
>>> DB::connection()->getPdo();
```

---

### ❌ Problem: Website shows 500 error after deploy

**Penyebab:** Cache atau permission issue

**Solusi:**

```bash
cd ~/public_html

# Clear all caches
php artisan cache:clear
php artisan config:clear
php artisan route:clear
php artisan view:clear

# Rebuild caches
php artisan config:cache
php artisan route:cache
php artisan view:cache

# Check permissions
chmod -R 755 storage bootstrap/cache
chmod -R 775 storage

# Check logs
tail -50 storage/logs/laravel.log
```

---

### 📊 Webhook Not Triggering

**Check:**

1. **GitHub webhook status:**

   - Settings → Webhooks → Check for red X or green checkmark
   - Recent Deliveries → Click request → Check response

2. **Token correct:**

   ```bash
   # In deploy.php
   grep "SECRET_TOKEN" ~/public_html/public/deploy.php
   ```

3. **Branch correct:**

   ```bash
   # Check branch name in deploy.sh
   grep "BRANCH=" ~/public_html/deploy.sh

   # Check your actual branch
   cd ~/public_html
   git branch
   ```

4. **Webhook log:**
   ```bash
   tail -100 ~/public_html/webhook.log
   ```

---

## Best Practices

### ✅ Security

- ✅ Always use HTTPS, never HTTP
- ✅ Keep `SECRET_TOKEN` private
- ✅ Restrict `deploy.php` to GitHub IPs only
- ✅ Never commit `.env` to repository
- ✅ Use `APP_DEBUG=false` in production
- ✅ Regularly update GitHub IP whitelist

### ✅ Maintenance

- ✅ Monitor logs regularly
- ✅ Rotate large log files
- ✅ Backup database before migrations
- ✅ Test deployments in staging first
- ✅ Keep Laravel and dependencies updated

### ✅ Performance

- ✅ Use `--no-dev` for production dependencies
- ✅ Always cache config, routes, and views
- ✅ Enable OPcache if available
- ✅ Use queue workers for heavy tasks
- ✅ Optimize images and assets

---

## Deployment Flow Diagram

```
Developer (Local Machine)
        ↓
    git commit
        ↓
    git push origin main/master
        ↓
GitHub Repository
        ↓
Webhook Triggered
        ↓
POST → https://yourdomain.com/deploy.php?token=xxx
        ↓
    Verify Token ✓
        ↓
    Execute ./deploy.sh
        ↓
┌─────────────────────────┐
│  Deployment Process     │
├─────────────────────────┤
│ 1. Enable Maintenance   │
│ 2. Git Pull Latest      │
│ 3. Composer Install     │
│ 4. Run Migrations       │
│ 5. Clear Cache          │
│ 6. Optimize App         │
│ 7. Set Permissions      │
│ 8. Disable Maintenance  │
└─────────────────────────┘
        ↓
   Website Updated! ✅
```

---

## Advanced Configuration

### Multiple Environments

Jika Anda memiliki staging dan production:

**1. Buat branch terpisah:**

- `staging` branch → staging.yourdomain.com
- `main/master` branch → yourdomain.com

**2. Buat deploy script terpisah:**

```bash
# deploy-staging.sh untuk staging
BRANCH="staging"
PROJECT_PATH="/home/username/staging"

# deploy-production.sh untuk production
BRANCH="main"
PROJECT_PATH="/home/username/public_html"
```

**3. Buat webhook terpisah:**

```php
// deploy-staging.php
define('BRANCH_TO_DEPLOY', 'staging');
define('DEPLOY_SCRIPT', '/home/username/staging/deploy-staging.sh');

// deploy-production.php (deploy.php)
define('BRANCH_TO_DEPLOY', 'main');
define('DEPLOY_SCRIPT', '/home/username/public_html/deploy.sh');
```

**4. Setup 2 webhooks di GitHub:**

- Staging: `https://staging.yourdomain.com/deploy-staging.php?token=xxx`
- Production: `https://yourdomain.com/deploy.php?token=xxx`

---

### Database Backup Before Migration

Tambahkan di `deploy.sh` sebelum migration:

```bash
# Backup database before migration
log_message "💾 Backing up database..." "$YELLOW"
BACKUP_FILE="$PROJECT_PATH/storage/backups/db-$(date +%Y%m%d-%H%M%S).sql"
mkdir -p "$PROJECT_PATH/storage/backups"

php artisan db:backup 2>&1 | tee -a "$LOG_FILE" || \
mysqldump -u$DB_USER -p$DB_PASS $DB_NAME > $BACKUP_FILE 2>&1

if [ -f "$BACKUP_FILE" ]; then
    log_message "✅ Database backed up: $BACKUP_FILE" "$GREEN"

    # Keep only last 7 backups
    ls -t $PROJECT_PATH/storage/backups/db-*.sql | tail -n +8 | xargs rm -f
else
    log_message "⚠️  Database backup failed, continuing..." "$YELLOW"
fi
```

---

### Slack Notifications

Tambahkan notifikasi Slack saat deployment berhasil/gagal:

**1. Dapatkan Slack Webhook URL** dari Slack settings

**2. Tambahkan di `deploy.sh`:**

```bash
# Configuration
SLACK_WEBHOOK="https://hooks.slack.com/services/YOUR/WEBHOOK/URL"

# Function untuk send notification
send_slack_notification() {
    local status=$1
    local message=$2
    local color=$3

    if [ -n "$SLACK_WEBHOOK" ]; then
        curl -X POST "$SLACK_WEBHOOK" \
        -H "Content-Type: application/json" \
        -d "{
            \"attachments\": [{
                \"color\": \"$color\",
                \"title\": \"Deployment $status\",
                \"text\": \"$message\",
                \"fields\": [{
                    \"title\": \"Branch\",
                    \"value\": \"$BRANCH\",
                    \"short\": true
                },{
                    \"title\": \"Server\",
                    \"value\": \"$PROJECT_PATH\",
                    \"short\": true
                }],
                \"footer\": \"Auto Deploy\",
                \"ts\": $(date +%s)
            }]
        }" 2>&1 > /dev/null
    fi
}

# Panggil saat success
send_slack_notification "Success" "✅ Deployment completed successfully!" "good"

# Panggil saat error
send_slack_notification "Failed" "❌ Deployment failed!" "danger"
```

---

### Email Notifications

Kirim email saat deployment:

```bash
# Function untuk send email
send_email_notification() {
    local status=$1
    local message=$2
    local email="admin@yourdomain.com"

    echo "$message" | mail -s "Deployment $status - $(date)" "$email"
}

# Panggil saat deployment
send_email_notification "Success" "Deployment completed at $(date)"
```

---

### Auto-restart Queue Workers

Jika menggunakan Laravel Queue:

```bash
# Restart queue workers after deployment
log_message "🔄 Restarting queue workers..." "$YELLOW"
php artisan queue:restart 2>&1 | tee -a "$LOG_FILE"

# Or restart supervisor
# supervisorctl restart laravel-worker:*
```

---

### Git Commit Info in Response

Tampilkan info commit di webhook response:

```php
// In deploy.php, add to success response
sendResponse('success', 'Deployment completed successfully!', [
    'execution_time' => $executionTime . 's',
    'commit' => [
        'id' => $commitInfo['id'],
        'message' => $commitInfo['message'],
        'author' => $commitInfo['author'],
        'url' => $commitInfo['url'],
        'timestamp' => $commitInfo['timestamp']
    ],
    'branch' => $branch,
    'deployed_at' => date('Y-m-d H:i:s'),
    'server' => gethostname()
]);
```

---

### Deployment Lock (Prevent Concurrent Deploys)

Cegah deployment bersamaan:

```bash
# Add at start of deploy.sh
LOCK_FILE="$PROJECT_PATH/deployment.lock"

if [ -f "$LOCK_FILE" ]; then
    log_message "⚠️  Deployment already in progress!" "$YELLOW"
    exit 1
fi

# Create lock
touch "$LOCK_FILE"

# Add trap to remove lock on exit
trap "rm -f $LOCK_FILE" EXIT

# Rest of deployment script...
```

---

### Rollback Script

Buat script untuk rollback jika deployment gagal:

```bash
nano rollback.sh
```

```bash
#!/bin/bash

# Rollback Script
PROJECT_PATH="/home/username/public_html"
cd "$PROJECT_PATH"

echo "🔙 Rolling back to previous commit..."

# Get previous commit
PREVIOUS_COMMIT=$(git rev-parse HEAD~1)

# Rollback
git reset --hard $PREVIOUS_COMMIT

# Reinstall dependencies
php composer.phar install --no-dev --optimize-autoloader --no-scripts

# Rollback database
php artisan migrate:rollback --step=1 --force

# Clear caches
php artisan cache:clear
php artisan config:clear

echo "✅ Rollback completed!"
```

```bash
chmod +x rollback.sh
```

---

## Maintenance Commands

### Manual Deployment

```bash
cd ~/public_html
./deploy.sh
```

### Check Deployment Status

```bash
# Current branch
git branch

# Last commit
git log -1

# Current version/tag
git describe --tags

# Check if there are updates
git fetch origin
git status
```

### Clear All Caches

```bash
cd ~/public_html

php artisan cache:clear
php artisan config:clear
php artisan route:clear
php artisan view:clear
php artisan optimize:clear
```

### View Logs

```bash
# Webhook log
tail -f ~/public_html/webhook.log

# Deployment log
tail -f ~/public_html/deployment.log

# Laravel log
tail -f ~/public_html/storage/logs/laravel.log

# All logs
tail -f ~/public_html/*.log
```

### Check Disk Space

```bash
# Check project size
du -sh ~/public_html

# Check storage size
du -sh ~/public_html/storage

# Check log files size
du -sh ~/public_html/*.log

# Clear old logs (older than 30 days)
find ~/public_html/storage/logs -name "*.log" -mtime +30 -delete
```

### Database Maintenance

```bash
# Optimize tables
php artisan db:optimize

# Clear expired sessions
php artisan session:gc

# Clear expired cache
php artisan cache:prune-stale-tags
```

---

## Monitoring Scripts

### Create Health Check Endpoint

```bash
nano ~/public_html/public/health.php
```

```php
<?php
// Simple health check
$checks = [
    'database' => false,
    'storage_writable' => is_writable(__DIR__ . '/../storage'),
    'cache_writable' => is_writable(__DIR__ . '/../bootstrap/cache'),
];

// Check database
try {
    require __DIR__ . '/../vendor/autoload.php';
    $app = require_once __DIR__ . '/../bootstrap/app.php';
    $app->make('Illuminate\Contracts\Console\Kernel')->bootstrap();

    DB::connection()->getPdo();
    $checks['database'] = true;
} catch (Exception $e) {
    $checks['database'] = false;
}

$healthy = !in_array(false, $checks, true);

http_response_code($healthy ? 200 : 503);
header('Content-Type: application/json');

echo json_encode([
    'status' => $healthy ? 'healthy' : 'unhealthy',
    'checks' => $checks,
    'timestamp' => date('Y-m-d H:i:s')
]);
```

### Monitor Script

```bash
nano ~/monitor-deploy.sh
```

```bash
#!/bin/bash

# Monitor deployment status
WEBHOOK_LOG="/home/username/public_html/webhook.log"
DEPLOY_LOG="/home/username/public_html/deployment.log"

echo "=== Last 5 Deployments ==="
grep "Starting deployment" "$WEBHOOK_LOG" | tail -5

echo ""
echo "=== Last Deployment Status ==="
grep "Deployment SUCCESS\|Deployment FAILED" "$WEBHOOK_LOG" | tail -1

echo ""
echo "=== Recent Errors ==="
grep -i "error\|failed" "$DEPLOY_LOG" | tail -5

echo ""
echo "=== Disk Usage ==="
df -h /home/username
```

```bash
chmod +x ~/monitor-deploy.sh
```

---

## FAQ

### Q: Apakah aman menggunakan CI/CD di shared hosting?

**A:** Ya, jika dikonfigurasi dengan benar:

- ✅ Gunakan token authentication
- ✅ Whitelist GitHub IPs
- ✅ Protect semua deployment files
- ✅ Never
