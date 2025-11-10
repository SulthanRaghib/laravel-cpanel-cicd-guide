# 09 - Setup GitHub Webhook

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
