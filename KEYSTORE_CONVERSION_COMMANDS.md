# 🔐 Keystore Conversion Commands - Quick Reference

## 📋 Konversi Keystore ke Base64

### **Linux / macOS:**

```bash
# Konversi keystore ke base64 (satu baris, no wrap)
base64 -w 0 mikuray_release.jks > keystore_base64.txt

# Atau tampilkan langsung di terminal
base64 -w 0 mikuray_release.jks

# Di macOS, gunakan -b 0 untuk no wrap
base64 -b 0 mikuray_release.jks > keystore_base64.txt
```

### **Windows PowerShell:**

```powershell
# Konversi keystore ke base64
[Convert]::ToBase64String([IO.File]::ReadAllBytes("mikuray_release.jks")) | Out-File keystore_base64.txt

# Atau tampilkan di console
[Convert]::ToBase64String([IO.File]::ReadAllBytes("mikuray_release.jks"))
```

### **Windows Command Prompt (dengan certutil):**

```cmd
certutil -encode mikuray_release.jks keystore_base64.txt
```

⚠️ **Note:** `certutil` menambahkan header/footer, hapus baris `-----BEGIN CERTIFICATE-----` dan `-----END CERTIFICATE-----`

---

## ✅ Verifikasi Konversi

### **Linux / macOS:**

```bash
# Decode base64 kembali ke keystore (untuk verifikasi)
base64 -d keystore_base64.txt > test_keystore.jks

# Cek keystore valid dengan keytool
keytool -list -v -keystore test_keystore.jks

# Input password saat diminta
# Jika berhasil list aliases, konversi benar ✅
```

### **Windows PowerShell:**

```powershell
# Decode base64 kembali ke keystore
[IO.File]::WriteAllBytes("test_keystore.jks", [Convert]::FromBase64String((Get-Content keystore_base64.txt)))

# Cek keystore valid
keytool -list -v -keystore test_keystore.jks
```

---

## 📝 Informasi Keystore yang Perlu Dicatat

Setelah konversi, catat informasi ini untuk GitHub Secrets:

```bash
# List aliases di keystore (untuk KEY_ALIAS)
keytool -list -keystore mikuray_release.jks

# Output akan tampil seperti:
# Alias name: mikuray-release  ← Ini yang dipakai di KEY_ALIAS
# Creation date: Aug 21, 2026
# Entry type: PrivateKeyEntry
```

### **Tabel Mapping:**

| GitHub Secret | Value dari Mana | Contoh |
|---------------|-----------------|--------|
| `KEYSTORE_BASE64` | Output `base64 -w 0 mikuray_release.jks` | `MIIJqwIBAzCCC...` |
| `KEY_ALIAS` | Alias name dari `keytool -list` | `mikuray-release` |
| `KEYSTORE_PASSWORD` | Password yang Anda gunakan saat generate keystore | `YourStrongPassword123` |
| `KEY_PASSWORD` | Password untuk private key (biasanya sama dengan keystore password) | `YourStrongPassword123` |

---

## 🔍 Troubleshooting

### **Error: "Keystore was tampered with, or password was incorrect"**

```bash
# Test password keystore
keytool -list -keystore mikuray_release.jks -storepass YOUR_PASSWORD

# Jika gagal, password salah atau keystore corrupt
```

### **Error: "Cannot find alias"**

```bash
# List semua aliases di keystore
keytool -list -keystore mikuray_release.jks

# Pastikan KEY_ALIAS sesuai dengan output
```

### **Base64 string terlalu panjang (GitHub Secrets limit 64KB)**

```bash
# Cek ukuran keystore
ls -lh mikuray_release.jks

# Jika > 48 KB, pertimbangkan:
# 1. Generate keystore baru dengan expiry lebih pendek
# 2. Atau gunakan alternative signing method
```

---

## 🛡️ Security Tips

```bash
# ❌ JANGAN commit keystore ke git
echo "*.jks" >> .gitignore
echo "*.keystore" >> .gitignore
echo "keystore_base64.txt" >> .gitignore

# ✅ Backup keystore ke safe storage
cp mikuray_release.jks ~/backup/mikuray_keystore_backup_$(date +%Y%m%d).jks

# ✅ Encrypt backup (opsional)
gpg -c ~/backup/mikuray_keystore_backup_20260822.jks

# ✅ Hapus file temporary
rm keystore_base64.txt
rm test_keystore.jks
```

---

## 📦 File yang Sudah Ada di Repository

Berdasarkan struktur project Anda:

```
keystore_temp/
├── keystore_single_line.txt      ← Base64 keystore (kemungkinan)
├── mikuray_keystore_base64.txt   ← Base64 keystore (kemungkinan)
└── mikuray_release.jks           ← Keystore asli
```

**Anda sudah punya file base64!** Cek isi file:

```bash
# Cek keystore_single_line.txt
head -c 100 keystore_temp/keystore_single_line.txt

# Atau mikuray_keystore_base64.txt
head -c 100 keystore_temp/mikuray_keystore_base64.txt

# Jika output adalah string base64 (MIIJqw...), bisa langsung dipakai!
```

⚠️ **INGAT:** Jangan commit folder `keystore_temp/` ke git public!

---

## 🚀 Quick Setup (Jika File Sudah Ada)

```bash
# 1. Copy base64 string
cat keystore_temp/keystore_single_line.txt

# 2. Verifikasi keystore info
keytool -list -v -keystore keystore_temp/mikuray_release.jks

# 3. Catat:
#    - Alias name (untuk KEY_ALIAS)
#    - Password (untuk KEYSTORE_PASSWORD & KEY_PASSWORD)

# 4. Tambahkan ke GitHub Secrets:
#    - KEYSTORE_BASE64: (paste isi keystore_single_line.txt)
#    - KEY_ALIAS: (dari keytool output)
#    - KEYSTORE_PASSWORD: (password Anda)
#    - KEY_PASSWORD: (password Anda)

# 5. Push workflow files
git add .github/workflows/
git commit -m "ci: add GitHub Actions workflow for build & sign"
git push origin master
```

**Done! 🎉**

---

## 📚 Reference Commands

```bash
# Generate keystore baru (jika belum punya)
keytool -genkeypair -v \
  -keystore mikuray_release.jks \
  -alias mikuray-release \
  -keyalg RSA \
  -keysize 2048 \
  -validity 10000 \
  -storepass YourPassword123 \
  -keypass YourPassword123 \
  -dname "CN=MikuRay, OU=Development, O=YourCompany, L=Jakarta, ST=DKI, C=ID"

# Change keystore password (jika lupa)
keytool -storepasswd -keystore mikuray_release.jks

# Change key password
keytool -keypasswd -keystore mikuray_release.jks -alias mikuray-release

# Export certificate (untuk app signing verification)
keytool -exportcert -keystore mikuray_release.jks -alias mikuray-release -file app.crt

# Show certificate fingerprint (SHA1, SHA256)
keytool -list -v -keystore mikuray_release.jks -alias mikuray-release
```

---

**All commands ready! Copy-paste dan go! 🚀**
