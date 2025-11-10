# 03 - Setup SSH Key untuk GitHub

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
