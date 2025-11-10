# 08 - Setup Cron Job

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
