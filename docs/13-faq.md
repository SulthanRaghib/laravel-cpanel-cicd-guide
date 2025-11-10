# 13 - FAQ

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
