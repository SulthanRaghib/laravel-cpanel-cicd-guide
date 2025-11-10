# 04 - Clone Repository ke cPanel

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
