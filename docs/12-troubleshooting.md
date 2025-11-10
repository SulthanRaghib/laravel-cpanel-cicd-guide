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
