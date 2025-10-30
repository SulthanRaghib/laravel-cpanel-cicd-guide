# 🚀 Laravel CI/CD Setup on cPanel

Dokumentasi lengkap langkah demi langkah untuk mengatur **Continuous Integration & Deployment (CI/CD)** otomatis menggunakan **GitHub Webhooks** di **shared hosting cPanel**.

> Cocok untuk developer Laravel yang ingin auto-deploy setiap push ke GitHub tanpa VPS.

---

## 📖 Quick Overview

- ✅ Setup Composer lokal di cPanel
- 🔐 Integrasi SSH Key dengan GitHub
- 🧰 Auto deploy script (`deploy.sh`)
- 🪝 GitHub Webhook handler (`deploy.php`)
- 🛡️ Security hardening (.htaccess + token)
- 📊 Monitoring logs dan troubleshooting

---

## 🧱 Struktur Panduan

| Tahap | File                                                      | Deskripsi                          |
| ----- | --------------------------------------------------------- | ---------------------------------- |
| 1     | [01-prerequisites.md](docs/01-prerequisites.md)           | Persiapan & kebutuhan sistem       |
| 2     | [02-server-setup.md](docs/02-server-setup.md)             | Setup Composer dan PHP environment |
| 3     | [03-ssh-setup.md](docs/03-ssh-setup.md)                   | Setup SSH Key untuk GitHub         |
| 4     | [06-deploy-script.md](docs/06-deploy-script.md)           | Membuat script otomatisasi deploy  |
| 5     | [07-webhook-handler.md](docs/07-webhook-handler.md)       | Membuat webhook receiver PHP       |
| 6     | [09-security.md](docs/09-security.md)                     | Konfigurasi keamanan production    |
| 7     | [10-testing-deployment.md](docs/10-testing-deployment.md) | Testing end-to-end deployment      |
| 8     | [SUMMARY.md](docs/SUMMARY.md)                             | Daftar isi lengkap dokumentasi     |

---

## ⚙️ Struktur Folder

```
/home/username/public_html/
├── public/
│   ├── deploy.php
│   ├── index.php
│   └── .htaccess
├── deploy.sh
├── composer.phar
├── deployment.log
└── webhook.log
```

---

## ⚖️ Lisensi

Distribusi bebas di bawah lisensi [MIT](LICENSE).
