# 10 - Security Configuration

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
