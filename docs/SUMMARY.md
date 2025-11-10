# CI/CD Setup untuk Laravel di cPanel

Dokumentasi lengkap untuk mengatur Continuous Integration/Continuous Deployment (CI/CD) otomatis untuk aplikasi Laravel di shared hosting cPanel menggunakan GitHub Webhooks dengan **Cron Job Flag System**.

> **⚠️ Catatan Penting:** Dokumentasi ini menggunakan **Cron Job Flag System** karena shared hosting cPanel umumnya mendisable execution functions (`exec`, `shell_exec`, `system`, `passthru`) untuk security.

---

## 📋 Table of Contents

1. [Prerequisites](#prerequisites)
2. [Persiapan Server cPanel](#persiapan-server-cpanel)
3. [Setup SSH Key untuk GitHub](#setup-ssh-key-untuk-github)
4. [Clone Repository ke cPanel](#clone-repository-ke-cpanel)
5. [Setup Laravel Environment](#setup-laravel-environment)
6. [Buat Deploy Script](#buat-deploy-script)
7. [Buat Webhook Handler](#buat-webhook-handler)
8. [Setup Cron Job](#setup-cron-job)
9. [Setup GitHub Webhook](#setup-github-webhook)
10. [Security Configuration](#security-configuration)
11. [Testing & Monitoring](#testing--monitoring)
12. [Troubleshooting](#troubleshooting)

---

## Prerequisites

Sebelum memulai, pastikan Anda memiliki:

- ✅ Akses SSH ke cPanel
- ✅ Repository GitHub dengan project Laravel
- ✅ Database MySQL sudah dibuat di cPanel
- ✅ Domain/subdomain sudah dikonfigurasi
- ✅ PHP 8.1 atau lebih tinggi (untuk Laravel 10+)
- ✅ Akses ke Cron Jobs di cPanel

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

### 3. Verifikasi PHP Execution Functions

Cek apakah execution functions disabled:

```bash
php -r "echo 'shell_exec: ' . (function_exists('shell_exec') ? 'enabled' : 'disabled') . PHP_EOL;"
php -r "echo 'exec: ' . (function_exists('exec') ? 'enabled' : 'disabled') . PHP_EOL;"
php -r "echo 'system: ' . (function_exists('system') ? 'enabled' : 'disabled') . PHP_EOL;"
php -r "echo 'passthru: ' . (function_exists('passthru') ? 'enabled' : 'disabled') . PHP_EOL;"
```

**Jika semua disabled**, dokumentasi ini cocok untuk Anda! ✅

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
 * For Laravel on cPanel with Disabled Execution Functions
 *
 * Uses Flag System + Cron Job for deployment
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
// DEPLOYMENT VIA FLAG SYSTEM
// =========================================

// Check deploy script exists
if (!file_exists(DEPLOY_SCRIPT)) {
    writeLog('❌ ERROR: Deploy script not found', ['path' => DEPLOY_SCRIPT]);
    sendResponse('error', 'Deploy script not found', ['path' => DEPLOY_SCRIPT], 500);
}

// Create deployment flag file (since exec/shell_exec disabled)
$flagFile = PROJECT_PATH . '/deploy.flag';
$flagData = [
    'timestamp' => time(),
    'branch' => $branch,
    'commit' => $commitInfo,
    'triggered_at' => date('Y-m-d H:i:s'),
    'delivery_id' => $delivery
];

// Write flag file
$flagWritten = @file_put_contents($flagFile, json_encode($flagData, JSON_PRETTY_PRINT));

if ($flagWritten === false) {
    writeLog('❌ ERROR: Cannot create deploy flag', [
        'flag_path' => $flagFile,
        'directory_writable' => is_writable(PROJECT_PATH)
    ]);
    sendResponse('error', 'Cannot create deployment flag', null, 500);
}

writeLog('✅ Deployment flag created', [
    'flag_file' => $flagFile,
    'commit' => $commitInfo['id'],
    'branch' => $branch,
    'note' => 'Waiting for cron job to execute deployment'
]);

sendResponse('success', 'Deployment queued successfully!', [
    'flag_created' => true,
    'commit' => $commitInfo,
    'branch' => $branch,
    'note' => 'Deployment will be executed by cron job within 1 minute',
    'queued_at' => date('Y-m-d H:i:s')
]);
?>
```

⚠️ **PENTING: Edit nilai berikut:**

- `SECRET_TOKEN` - Paste token dari `openssl rand -hex 32`
- `PROJECT_PATH` - Path ke project Anda (contoh: `/home/jasz5267/public_html`)
- `BRANCH_TO_DEPLOY` - `'main'` atau `'master'`

**Save:** Ctrl+X, Y, Enter

---

## Setup Cron Job

Karena execution functions disabled, kita butuh Cron Job untuk menjalankan deployment.

### 1. Buat Cron Deploy Script

```bash
cd ~/public_html
nano cron-deploy.sh
```

### 2. Paste Script Berikut

```bash
#!/bin/bash

# =========================================
# Cron Deploy Script
# Executes deployment if flag exists
# Runs every minute via cron job
# =========================================

PROJECT_PATH="/home/YOUR_USERNAME/public_html"
FLAG_FILE="$PROJECT_PATH/deploy.flag"
DEPLOY_SCRIPT="$PROJECT_PATH/deploy.sh"
LOG_FILE="$PROJECT_PATH/cron-deploy.log"

# Function to log
log_message() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" >> "$LOG_FILE"
}

# Check if flag file exists
if [ ! -f "$FLAG_FILE" ]; then
    # No deployment needed, exit silently
    exit 0
fi

log_message "=========================================="
log_message "📍 Deploy flag detected! Starting deployment..."

# Read flag data
if [ -f "$FLAG_FILE" ]; then
    FLAG_DATA=$(cat "$FLAG_FILE")
    log_message "Flag Data: $FLAG_DATA"
fi

# Remove flag file immediately to prevent duplicate execution
rm -f "$FLAG_FILE"
log_message "✅ Flag file removed"

# Execute deployment
log_message "🚀 Executing deployment script..."
cd "$PROJECT_PATH"

if [ -x "$DEPLOY_SCRIPT" ]; then
    # Execute deploy script and capture output
    "$DEPLOY_SCRIPT" >> "$LOG_FILE" 2>&1
    EXIT_CODE=$?

    if [ $EXIT_CODE -eq 0 ]; then
        log_message "✅ Deployment completed successfully!"
    else
        log_message "❌ Deployment failed with exit code: $EXIT_CODE"
    fi
else
    log_message "❌ Deploy script not found or not executable: $DEPLOY_SCRIPT"
    log_message "   Checking file: $(ls -la $DEPLOY_SCRIPT 2>&1)"
fi

log_message "=========================================="
```

⚠️ **PENTING: Edit** `PROJECT_PATH="/home/YOUR_USERNAME/public_html"`

**Save:** Ctrl+X, Y, Enter

### 3. Beri Permission Eksekusi

```bash
chmod +x cron-deploy.sh
```

### 4. Test Cron Script Manual

```bash
# Create test flag
echo '{"test": "manual"}' > ~/public_html/deploy.flag

# Run cron script
~/public_html/cron-deploy.sh

# Check logs
tail -50 ~/public_html/cron-deploy.log
```

### 5. Setup Cron Job via SSH

```bash
crontab -e
```

Jika muncul pilihan editor, pilih `nano` (option 1).

Tambahkan baris ini:

```bash
# Laravel Auto Deploy - Check for deploy flag every minute
* * * * * /home/YOUR_USERNAME/public_html/cron-deploy.sh >/dev/null 2>&1
```

⚠️ **Ganti** `YOUR_USERNAME` dengan username cPanel Anda!

**Save:** Ctrl+X, Y, Enter

### 6. Verify Cron Job

```bash
crontab -l
```

Output seharusnya menampilkan cron job yang baru ditambahkan.

### 7. Alternative: Setup via cPanel GUI

Jika tidak bisa via SSH:

1. Login cPanel
2. **Advanced** → **Cron Jobs**
3. **Common Settings**: Every Minute `(* * * * *)`
4. **Command**:
   ```bash
   /home/YOUR_USERNAME/public_html/cron-deploy.sh >/dev/null 2>&1
   ```
5. **Add New Cron Job**

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

**Secret:** (kosongkan)

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

⚠️ **Note:** GitHub IP ranges dapat berubah. Cek update di: https://api.github.com/meta

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

# Deny access to flag files
<FilesMatch "\.flag$">
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

### 3. Set File Permissions

```bash
# Deploy scripts
chmod 700 ~/public_html/deploy.sh
chmod 700 ~/public_html/cron-deploy.sh

# Webhook handler
chmod 644 ~/public_html/public/deploy.php

# Log files
chmod 644 ~/public_html/*.log
```

---

## Testing & Monitoring

### 1. Test Full Deployment Flow

**Step 1: Trigger webhook via curl**

```bash
curl -X POST "https://yourdomain.com/deploy.php?token=YOUR_TOKEN" \
  -H "X-GitHub-Event: push" \
  -H "Content-Type: application/json" \
  -d '{"ref":"refs/heads/master","head_commit":{"id":"test","message":"Test deploy","author":{"name":"Test"}}}'
```

**Expected Response:**

```json
{
    "status": "success",
    "message": "Deployment queued successfully!",
    "data": {
        "flag_created": true,
        "note": "Deployment will be executed by cron job within 1 minute",
        ...
    }
}
```

**Step 2: Check flag created**

```bash
ls -la ~/public_html/deploy.flag
```

**Step 3: Wait 60 seconds for cron**

```bash
sleep 65
```

**Step 4: Verify deployment**

```bash
# Flag should be removed
ls -la ~/public_html/deploy.flag  # Should not exist

# Check cron log
tail -30 ~/public_html/cron-deploy.log

# Check deployment log
tail -30 ~/public_html/deployment.log
```

### 2. Test dengan GitHub Push

Di **local computer**:

```bash
# Make a change
echo "# CI/CD Test" >> README.md

# Commit and push
git add .
git commit -m "test: CI/CD auto deployment"
git push origin master  # or main
```

### 3. Monitor Logs (Real-time)

Via SSH:

```bash
# Watch all logs
tail -f ~/public_html/webhook.log ~/public_html/cron-deploy.log ~/public_html/deployment.log
```

Press `Ctrl+C` to stop.

### 4. Check GitHub Webhook Status

1. **GitHub** → **Settings** → **Webhooks**
2. Click your webhook
3. **Recent Deliveries** tab
4. Latest request should show:
   - ✅ Status 200
   - Response body: `"status": "success"`

---

## Troubleshooting

### ❌ Error 500: No Response Body

**Penyebab:** PHP execution functions disabled, `deploy.php` mencoba execute script

**Solusi:** Update `deploy.php` untuk gunakan flag system (sudah dijelaskan di section [Buat Webhook Handler](#buat-webhook-handler))

```bash
# Verify functions disabled
php -r "echo 'exec: ' . (function_exists('exec') ? 'enabled' : 'disabled') . PHP_EOL;"
```

---

### ❌ Error 403: Unauthorized/Invalid Token

**Penyebab:** Token di GitHub webhook tidak match dengan token di `deploy.php`

**Solusi:**

```bash
# Check token in deploy.php
grep "SECRET_TOKEN" ~/public_html/public/deploy.php

# Update GitHub webhook dengan token yang benar
# GitHub → Settings → Webhooks → Edit webhook → Update Payload URL
```

---

### ❌ Flag File Tidak Terhapus

**Penyebab:** Cron job tidak berjalan atau script tidak executable

**Solusi:**

```bash
# Check cron job exists
crontab -l

# Check script executable
ls -la ~/public_html/cron-deploy.sh

# Make executable
chmod +x ~/public_html/cron-deploy.sh

# Test manual
~/public_html/cron-deploy.sh

# Check cron log
tail -50 ~/public_html/cron-deploy.log
```

---

### ❌ Deployment Tidak Jalan

**Penyebab:** Multiple kemungkinan

**Diagnostic Steps:**

```bash
# 1. Check webhook received
tail -20 ~/public_html/webhook.log

# 2. Check flag created
ls -la ~/public_html/deploy.flag

# 3. Check cron executed
grep "Deploy flag detected" ~/public_html/cron-deploy.log | tail -5

# 4. Check deployment ran
tail -50 ~/public_html/deployment.log

# 5. Check cron is running
ps aux | grep cron
```

---

### ❌ Git Pull Failed

**Penyebab:** SSH key atau git remote bermasalah

**Solusi:**

```bash
# Test SSH connection
ssh -T git@github.com

# Check git remote
cd ~/public_html
git remote -v

# Should show: git@github.com:username/repo.git

# Fix if using HTTPS
git remote set-url origin git@github.com:username/repo.git
```

---

### ❌ Composer Install Failed

**Penyebab:** composer.phar tidak ada atau memory limit

**Solusi:**

```bash
# Check composer exists
ls -la ~/public_html/composer.phar

# Re-download if missing
cd ~/public_html
curl -sS https://getcomposer.org/installer | php

# Try with higher memory limit
php -d memory_limit=512M composer.phar install --no-dev --optimize-autoloader --no-scripts
```

---

### ❌ Permission Denied Errors

**Solusi:**

```bash
cd ~/public_html

# Fix storage permissions
chmod -R 755 storage bootstrap/cache
chmod -R 775 storage

# Fix scripts
chmod +x deploy.sh
chmod +x cron-deploy.sh

# Check ownership
ls -la | grep -E "deploy|storage"
```

---

### 📊 Monitoring Commands

```bash
# Check deployment queue status
ls -la ~/public_html/deploy.flag

# View flag content
cat ~/public_html/deploy.flag

# Last 10 webhook triggers
grep "Deployment flag created" ~/public_html/webhook.log | tail -10

# Last 10 cron executions
grep "Deploy flag detected" ~/public_html/cron-deploy.log | tail -10

# Last deployment status
tail -30 ~/public_html/deployment.log

# Monitor all logs real-time
tail -f ~/public_html/*.log

# Check cron job
crontab -l

# Watch deployment process
watch -n 2 'echo "=== Flag Status ===" && ls -la ~/public_html/deploy.flag 2>&1 && echo "" && echo "=== Last Cron Execution ===" && tail -5 ~/public_html/cron-deploy.log'
```

---

## Deployment Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                     CI/CD FLOW DENGAN CRON                  │
└─────────────────────────────────────────────────────────────┘

Developer (Local)
       ↓
   git push origin master
       ↓
GitHub Repository
       ↓
Webhook Triggered
       ↓
POST → https://yourdomain.com/deploy.php?token=xxx
       ↓
✅ Verify Token
       ↓
📄 Create deploy.flag
       ↓
Return 200 OK "Deployment Queued"
       ↓
[Wait max 60 seconds...]
       ↓
Cron Job Runs (every minute)
       ↓
📍 Check deploy.flag exists?
       ↓ YES
🗑️ Remove flag immediately
       ↓
Execute deploy.sh
       ↓
┌─────────────────────────┐
│  Deployment Process     │
├─────────────────────────┤
│ 1. Maintenance Mode ON  │
│ 2. Git Pull Latest      │
│ 3. Composer Install     │
│ 4. Package Discovery    │
│ 5. Run Migrations       │
│ 6. Clear All Caches     │
│ 7. Optimize App         │
│ 8. Set Permissions      │
│ 9. Maintenance Mode OFF │
└─────────────────────────┘
       ↓
✅ Website Updated!
       ↓
Log Success to cron-deploy.log
```

---

## Keuntungan Flag System

### ✅ Bypass PHP Restrictions

```
PHP Web Context (deploy.php)
  ❌ shell_exec() disabled
  ❌ exec() disabled
  ❌ system() disabled
  ❌ passthru() disabled
  ✅ file_put_contents() allowed ← Create flag

SSH/Cron Context (cron-deploy.sh)
  ✅ All bash commands available
  ✅ Can execute deploy.sh
  ✅ Full system access
```

### ✅ Reliable & Safe

- **Async Execution:** Webhook return cepat, tidak timeout
- **No Concurrent Runs:** Flag dihapus immediately, prevent duplicate
- **Guaranteed Execution:** Cron runs setiap menit
- **Full Logging:** Terpisah webhook, cron, dan deployment logs
- **Auto Recovery:** Jika cron gagal, flag tetap ada untuk next run

### ✅ Simple & Maintainable

- No complex process handling
- Easy to debug (check flag file)
- Clear separation of concerns
- Standard cron job (built-in di cPanel)

---

## FAQ

### Q: Kenapa butuh Cron Job?

**A:** Karena shared hosting disable execution functions (`exec`, `shell_exec`, dll) untuk security. PHP tidak bisa langsung execute bash script, jadi kita pakai:

1. **deploy.php** (PHP) → Create file flag (simple file write)
2. **Cron job** (Bash/SSH) → Detect flag & execute deployment

### Q: Berapa lama delay deployment?

**A:** **Maksimal 60 detik** dari push ke live. Cron berjalan setiap menit, jadi:

- Push jam 10:00:30 → Cron detect jam 10:01:00 → Deploy 15 detik → Live jam 10:01:15

### Q: Apakah aman?

**A:** Ya, jika dikonfigurasi benar:

- ✅ Token authentication
- ✅ IP whitelist (hanya GitHub IPs)
- ✅ Flag system (tidak ada parameter injection)
- ✅ Proper file permissions
- ✅ Log semua aktivitas

### Q: Bagaimana jika ada 2 push bersamaan?

**A:** Aman! Flag dihapus immediately sebelum deployment. Push kedua akan create flag baru, yang akan diproses di cron run berikutnya (1 menit kemudian).

### Q: Bagaimana cara rollback?

**A:**

```bash
cd ~/public_html

# Rollback ke commit sebelumnya
git reset --hard HEAD~1

# Atau rollback ke commit tertentu
git reset --hard <commit-hash>

# Re-run deployment
./deploy.sh
```

### Q: Apakah bisa deploy dari branch lain?

**A:** Ya, edit `BRANCH` di `deploy.sh` dan `BRANCH_TO_DEPLOY` di `deploy.php`:

```bash
# deploy.sh
BRANCH="development"
```

```php
// deploy.php
define('BRANCH_TO_DEPLOY', 'development');
```

### Q: Bagaimana cara disable auto-deploy sementara?

**A:**

**Option 1:** Disable cron job

```bash
crontab -e
# Comment the line dengan #
# * * * * * /home/username/public_html/cron-deploy.sh >/dev/null 2>&1
```

**Option 2:** Disable GitHub webhook

- GitHub → Settings → Webhooks → Edit → Uncheck "Active"

### Q: Apakah flag file bisa membesar?

**A:** Tidak, ukuran flag file hanya ~200-300 bytes. Dan file dihapus setelah deployment. Tidak akan menumpuk.

### Q: Bagaimana cara test tanpa push ke GitHub?

**A:**

```bash
# Create flag manual
echo '{"test": true}' > ~/public_html/deploy.flag

# Wait for cron or run manual
~/public_html/cron-deploy.sh
```

---

## Best Practices

### ✅ Security

- ✅ Use strong random token (32+ characters)
- ✅ Always use HTTPS, never HTTP
- ✅ Restrict `deploy.php` to GitHub IPs only
- ✅ Never commit `.env` or tokens to repository
- ✅ Set proper file permissions (700 for scripts, 644 for PHP)
- ✅ Monitor logs regularly for unauthorized attempts
- ✅ Update GitHub IP whitelist regularly

### ✅ Maintenance

- ✅ Rotate log files when too large (>5MB)
- ✅ Backup database before major migrations
- ✅ Test deployments in staging environment first
- ✅ Keep Laravel and dependencies updated
- ✅ Monitor cron job execution
- ✅ Clean up old backup files

### ✅ Performance

- ✅ Use `--no-dev` for production dependencies
- ✅ Always cache config, routes, and views
- ✅ Enable OPcache if available
- ✅ Use queue workers for heavy tasks
- ✅ Optimize images before commit
- ✅ Minimize deployment downtime (<20 seconds)

### ✅ Logging

- ✅ Keep separate logs: webhook, cron, deployment
- ✅ Log all important events (push, flag created, deployment start/end)
- ✅ Include timestamps in all logs
- ✅ Implement log rotation (5MB limit)
- ✅ Monitor logs for errors regularly

---

## Advanced Configuration

### Multiple Environments (Staging + Production)

**1. Structure:**

```
~/staging/          → Staging environment
  ├── deploy.sh
  ├── cron-deploy.sh
  └── public/deploy-staging.php

~/public_html/      → Production environment
  ├── deploy.sh
  ├── cron-deploy.sh
  └── public/deploy.php
```

**2. Cron Jobs:**

```bash
# Staging - every minute
* * * * * /home/username/staging/cron-deploy.sh >/dev/null 2>&1

# Production - every minute
* * * * * /home/username/public_html/cron-deploy.sh >/dev/null 2>&1
```

**3. GitHub Webhooks:**

- `staging` branch → `https://staging.yourdomain.com/deploy-staging.php?token=xxx`
- `main` branch → `https://yourdomain.com/deploy.php?token=xxx`

---

### Slack Notifications

Tambahkan notifikasi Slack di `deploy.sh`:

```bash
# Configuration
SLACK_WEBHOOK="https://hooks.slack.com/services/YOUR/WEBHOOK/URL"

# Function
send_slack_notification() {
    local message=$1
    local color=$2

    if [ -n "$SLACK_WEBHOOK" ]; then
        curl -X POST "$SLACK_WEBHOOK" \
        -H "Content-Type: application/json" \
        -d "{
            \"attachments\": [{
                \"color\": \"$color\",
                \"text\": \"$message\",
                \"fields\": [{
                    \"title\": \"Branch\",
                    \"value\": \"$BRANCH\",
                    \"short\": true
                },{
                    \"title\": \"Time\",
                    \"value\": \"$(date '+%Y-%m-%d %H:%M:%S')\",
                    \"short\": true
                }]
            }]
        }" 2>&1 > /dev/null
    fi
}

# Call after deployment
send_slack_notification "✅ Deployment completed successfully!" "good"
```

---

### Database Backup Before Migration

Tambahkan di `deploy.sh` sebelum migration:

```bash
# Backup database
log_message "💾 Backing up database..." "$YELLOW"
BACKUP_DIR="$PROJECT_PATH/storage/backups"
mkdir -p "$BACKUP_DIR"
BACKUP_FILE="$BACKUP_DIR/db-$(date +%Y%m%d-%H%M%S).sql"

# Get DB credentials from .env
DB_NAME=$(grep DB_DATABASE .env | cut -d '=' -f2)
DB_USER=$(grep DB_USERNAME .env | cut -d '=' -f2)
DB_PASS=$(grep DB_PASSWORD .env | cut -d '=' -f2)

mysqldump -u"$DB_USER" -p"$DB_PASS" "$DB_NAME" > "$BACKUP_FILE" 2>&1

if [ -f "$BACKUP_FILE" ]; then
    log_message "✅ Database backed up: $BACKUP_FILE" "$GREEN"

    # Keep only last 7 backups
    ls -t "$BACKUP_DIR"/db-*.sql | tail -n +8 | xargs rm -f
else
    log_message "⚠️  Database backup failed" "$YELLOW"
fi
```

---

### Deployment Lock

Prevent concurrent deployments:

```bash
# Add at start of deploy.sh
LOCK_FILE="$PROJECT_PATH/deployment.lock"

if [ -f "$LOCK_FILE" ]; then
    log_message "⚠️  Deployment already in progress!" "$YELLOW"
    exit 1
fi

# Create lock
touch "$LOCK_FILE"

# Remove lock on exit (success or failure)
trap "rm -f $LOCK_FILE" EXIT
```

---

### Health Check Endpoint

Buat file `public/health.php`:

```php
<?php
header('Content-Type: application/json');

$checks = [
    'status' => 'healthy',
    'timestamp' => date('Y-m-d H:i:s'),
    'checks' => [
        'storage_writable' => is_writable(__DIR__ . '/../storage'),
        'cache_writable' => is_writable(__DIR__ . '/../bootstrap/cache'),
    ]
];

// Check database
try {
    require __DIR__ . '/../vendor/autoload.php';
    $app = require_once __DIR__ . '/../bootstrap/app.php';
    $app->make('Illuminate\Contracts\Console\Kernel')->bootstrap();

    DB::connection()->getPdo();
    $checks['checks']['database'] = true;
} catch (Exception $e) {
    $checks['checks']['database'] = false;
    $checks['status'] = 'unhealthy';
}

http_response_code($checks['status'] === 'healthy' ? 200 : 503);
echo json_encode($checks, JSON_PRETTY_PRINT);
```

Access: `https://yourdomain.com/health.php`

---

## Quick Reference Card

```bash
# ==========================================
# ESSENTIAL COMMANDS
# ==========================================

# Test deployment manually
cd ~/public_html && ./deploy.sh

# Create flag manually (trigger deploy)
echo '{"test":true}' > ~/public_html/deploy.flag

# Monitor logs real-time
tail -f ~/public_html/webhook.log
tail -f ~/public_html/cron-deploy.log
tail -f ~/public_html/deployment.log

# Check flag status
ls -la ~/public_html/deploy.flag

# View flag content
cat ~/public_html/deploy.flag

# Test webhook
curl -X POST "https://yourdomain.com/deploy.php?token=TOKEN" \
  -H "X-GitHub-Event: push" \
  -H "Content-Type: application/json" \
  -d '{"ref":"refs/heads/master","head_commit":{"id":"test","message":"Test","author":{"name":"Test"}}}'

# ==========================================
# CRON JOB MANAGEMENT
# ==========================================

# View cron jobs
crontab -l

# Edit cron jobs
crontab -e

# Test cron script
~/public_html/cron-deploy.sh

# Check last cron executions
grep "Deploy flag detected" ~/public_html/cron-deploy.log | tail -10

# ==========================================
# TROUBLESHOOTING
# ==========================================

# Check PHP execution functions
php -r "echo 'exec: '.(function_exists('exec')?'enabled':'disabled').PHP_EOL;"

# Test SSH to GitHub
ssh -T git@github.com

# Check git remote
cd ~/public_html && git remote -v

# Fix permissions
cd ~/public_html
chmod +x deploy.sh cron-deploy.sh
chmod -R 755 storage bootstrap/cache
chmod -R 775 storage

# Clear Laravel caches
php artisan cache:clear
php artisan config:clear
php artisan route:clear
php artisan view:clear

# Rebuild caches
php artisan config:cache
php artisan route:cache
php artisan view:cache

# ==========================================
# MONITORING
# ==========================================

# Last 5 deployments
grep "Starting deployment" ~/public_html/cron-deploy.log | tail -5

# Last deployment status
tail -20 ~/public_html/deployment.log | grep "Deployment completed"

# Check errors
grep -i "error\|failed" ~/public_html/deployment.log | tail -10

# Disk usage
du -sh ~/public_html

# ==========================================
# FILES LOCATION
# ==========================================

# Project root:        ~/public_html/
# Deploy script:       ~/public_html/deploy.sh
# Cron script:         ~/public_html/cron-deploy.sh
# Webhook handler:     ~/public_html/public/deploy.php
# Flag file:           ~/public_html/deploy.flag
# Webhook log:         ~/public_html/webhook.log
# Cron log:            ~/public_html/cron-deploy.log
# Deployment log:      ~/public_html/deployment.log
# Laravel log:         ~/public_html/storage/logs/laravel.log
```

---

## Resources

### Official Documentation

- [Laravel Deployment](https://laravel.com/docs/deployment)
- [GitHub Webhooks](https://docs.github.com/en/webhooks)
- [cPanel Documentation](https://docs.cpanel.net/)

### GitHub IP Ranges

- API: https://api.github.com/meta
- Docs: https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/about-githubs-ip-addresses

### Useful Tools

- [Webhook Tester](https://webhook.site/)
- [JSON Validator](https://jsonlint.com/)
- [Cron Expression Generator](https://crontab.guru/)

---

## Support & Contribution

Jika menemukan masalah atau ingin improve dokumentasi:

1. Check [Troubleshooting](#troubleshooting) section
2. Review [FAQ](#faq)
3. Verify semua konfigurasi sesuai dokumentasi
4. Check logs untuk error details

---

## Changelog

### Version 2.0.0 (2025-10-31)

- ✅ **MAJOR:** Tambah Cron Job Flag System untuk handle disabled execution functions
- ✅ Update `deploy.php` untuk create flag instead execute script
- ✅ Tambah `cron-deploy.sh` untuk handle actual deployment
- ✅ Improve troubleshooting section
- ✅ Tambah FAQ tentang cron job system
- ✅ Update deployment flow diagram

### Version 1.0.0 (2025-10-30)

- ✅ Initial documentation
- ✅ Basic CI/CD setup for cPanel
- ✅ Direct execution via shell_exec (deprecated untuk shared hosting)

---

## License

Dokumentasi ini bebas digunakan dan dimodifikasi sesuai kebutuhan Anda.

---

## Credits

Dibuat dengan ❤️ untuk mempermudah deployment Laravel di cPanel shared hosting.

**Happy Deploying! 🚀**

---

**END OF DOCUMENTATION**
