# 06 - Buat Deploy Script

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
