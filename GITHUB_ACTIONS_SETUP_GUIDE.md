# 🤖 GitHub Actions - Setup Guide untuk Build & Sign APK

## 📋 Gambaran Umum

Repository ini sudah dilengkapi dengan 2 workflow GitHub Actions:

1. **`build-release.yml`** - Build dan sign Release APK secara otomatis
2. **`build-debug.yml`** - Build Debug APK untuk testing

---

## 🔧 Persiapan: Convert Keystore ke Base64

Sebelum menggunakan workflow, Anda perlu mengkonversi file `.keystore` atau `.jks` Anda ke format Base64.

### **Command untuk Linux/Mac:**

```bash
# Konversi keystore ke base64 (output ke file)
base64 -w 0 mikuray_release.jks > keystore_base64.txt

# Atau langsung tampilkan di terminal (untuk copy)
base64 -w 0 mikuray_release.jks

# Verifikasi konversi (decode kembali untuk test)
base64 -d keystore_base64.txt > test_keystore.jks
```

### **Command untuk Windows (PowerShell):**

```powershell
# Konversi keystore ke base64
[Convert]::ToBase64String([IO.File]::ReadAllBytes("mikuray_release.jks")) | Out-File keystore_base64.txt

# Atau tampilkan di console
[Convert]::ToBase64String([IO.File]::ReadAllBytes("mikuray_release.jks"))
```

### **Hasil:**

File `keystore_base64.txt` akan berisi string panjang seperti ini:

```
MIIJqwIBAzCCCWQGCSqGSIb3DQEHAaCCCVUEgglRMIIJTTCCBW8GCSqGSIb3DQEH...
(ribuan karakter base64)
```

⚠️ **PENTING:** 
- Copy seluruh isi file (satu baris panjang, tanpa spasi atau newline di tengah)
- Jangan share ke public!

---

## 🔑 Setup GitHub Secrets

Buka repository Anda di GitHub, lalu:

**Settings → Secrets and variables → Actions → New repository secret**

### **Daftar Secrets yang Harus Dibuat:**

| Secret Name | Deskripsi | Contoh Value |
|-------------|-----------|--------------|
| `KEYSTORE_BASE64` | Keystore dalam format Base64 | `MIIJqwIBAzCCC...` (dari langkah sebelumnya) |
| `KEY_ALIAS` | Alias key yang dipakai saat generate keystore | `mikuray-release` atau `my-key-alias` |
| `KEYSTORE_PASSWORD` | Password untuk buka keystore file | `myStrongPassword123` |
| `KEY_PASSWORD` | Password untuk private key (bisa sama dengan keystore password) | `myStrongPassword123` |

### **Cara Menambahkan Secret:**

1. Klik **"New repository secret"**
2. **Name:** Masukkan nama secret (misal: `KEYSTORE_BASE64`)
3. **Secret:** Paste value secret (misal: base64 string dari keystore)
4. Klik **"Add secret"**
5. Ulangi untuk 4 secrets di atas

### **Screenshot Reference:**

```
GitHub Repo → Settings → Secrets and variables → Actions

Repository secrets:
✅ KEYSTORE_BASE64        Updated 2 minutes ago
✅ KEY_ALIAS              Updated 2 minutes ago
✅ KEYSTORE_PASSWORD      Updated 2 minutes ago
✅ KEY_PASSWORD           Updated 2 minutes ago
```

---

## 🚀 Cara Menggunakan Workflow

### **1. Build Release APK (Signed) - Otomatis**

Workflow `build-release.yml` akan **otomatis berjalan** saat:

✅ **Push ke branch `master` atau `main`** dengan perubahan di folder `V2rayNG/`
✅ **Create tag** (misal: `v2.2.9`)

**Contoh:**
```bash
# Push ke master
git add .
git commit -m "fix: clipboard import bug"
git push origin master

# Atau create release tag
git tag v2.2.9
git push origin v2.2.9
```

Workflow akan:
1. Build APK Release
2. Sign dengan keystore Anda
3. Upload ke GitHub Artifacts
4. (Opsional) Create GitHub Release jika pakai tag

---

### **2. Build Release APK (Signed) - Manual Trigger**

Bisa juga trigger manual dari GitHub:

1. Buka **Actions tab** di GitHub
2. Pilih workflow **"Build and Sign Release APK"**
3. Klik **"Run workflow"**
4. Pilih branch (misal: `master`)
5. (Opsional) Masukkan version name
6. Klik **"Run workflow"**

---

### **3. Build Debug APK - Manual Trigger**

Untuk testing tanpa sign:

1. Buka **Actions tab** di GitHub
2. Pilih workflow **"Build Debug APK"**
3. Klik **"Run workflow"**
4. Pilih branch
5. Klik **"Run workflow"**

---

## 📥 Download APK dari GitHub Actions

Setelah workflow selesai:

1. Buka **Actions tab**
2. Klik pada workflow run yang sudah selesai (hijau ✅)
3. Scroll ke bawah ke bagian **"Artifacts"**
4. Download file:
   - **Release:** `MikuRay-Release-APKs-XXX.zip`
   - **Debug:** `MikuRay-Debug-APKs-XXX.zip`
5. Extract ZIP, install APK ke Android device

---

## 📱 Naming Convention APK

### **Release APK:**
```
MikuRay_2.2.9-arm64-v8a-release-20260822-1430.apk
MikuRay_2.2.9-armeabi-v7a-release-20260822-1430.apk
```

Format: `MikuRay_{VERSION}-{ABI}-release-{DATE}.apk`

### **Debug APK:**
```
MikuRay_2.2.9-arm64-v8a-debug.apk
MikuRay_2.2.9-armeabi-v7a-debug.apk
```

---

## 🏗️ Workflow Architecture

### **build-release.yml (Release Signed APK):**

```
Trigger (Push to master / Create tag / Manual)
  ↓
Checkout code
  ↓
Setup Java 21 + Android SDK + NDK
  ↓
Cache Gradle dependencies
  ↓
Build Release APK (unsigned)
  ↓
Sign APK with r0adkll/sign-android-release
  ↓
Rename APK with timestamp
  ↓
Upload to GitHub Artifacts
  ↓
(Optional) Create GitHub Release if tag
```

### **build-debug.yml (Debug APK):**

```
Trigger (Pull Request / Manual)
  ↓
Checkout code
  ↓
Setup Java 21 + Android SDK + NDK
  ↓
Build Debug APK
  ↓
Upload to GitHub Artifacts
```

---

## 🔍 Monitoring & Troubleshooting

### **Cek Status Workflow:**

1. Buka **Actions tab** di GitHub
2. Lihat daftar workflow runs
3. Status:
   - 🟢 **Green checkmark** = Success
   - 🔴 **Red X** = Failed
   - 🟡 **Yellow circle** = Running

### **Debug Build Failure:**

1. Klik workflow run yang failed
2. Klik job **"Build and Sign Release APK"**
3. Expand step yang error (merah ❌)
4. Baca error message
5. Common issues:
   - **Keystore invalid:** Cek `KEYSTORE_BASE64` format benar
   - **Password salah:** Verifikasi `KEYSTORE_PASSWORD` dan `KEY_PASSWORD`
   - **Alias not found:** Cek `KEY_ALIAS` sesuai dengan keystore
   - **Build failed:** Cek code error di V2rayNG/

### **Test Keystore Secrets Locally:**

```bash
# Decode base64 kembali ke keystore
echo "YOUR_BASE64_STRING" | base64 -d > test.jks

# Test sign manual dengan keytool
keytool -list -v -keystore test.jks -storepass YOUR_PASSWORD

# Test sign APK manual dengan apksigner
apksigner sign --ks test.jks \
  --ks-key-alias YOUR_ALIAS \
  --ks-pass pass:YOUR_KEYSTORE_PASSWORD \
  --key-pass pass:YOUR_KEY_PASSWORD \
  --out app-signed.apk \
  app-unsigned.apk
```

---

## 🛡️ Security Best Practices

### ✅ **DO:**
- Store keystore di GitHub Secrets (encrypted)
- Gunakan strong password untuk keystore
- Rotate keystore password secara berkala
- Backup keystore file di tempat aman (offline)
- Gunakan 2FA di GitHub account

### ❌ **DON'T:**
- Commit keystore file ke git repository
- Share keystore password via chat/email
- Hardcode password di workflow file
- Upload keystore ke public storage

---

## 📊 Retention Policy

- **Release APKs:** Disimpan 30 hari di GitHub Artifacts
- **Debug APKs:** Disimpan 7 hari di GitHub Artifacts
- **GitHub Releases:** Permanent (jika pakai tag)

Untuk permanent storage, gunakan GitHub Releases dengan tag.

---

## 🎯 Advanced: Custom Version Name

### **Manual Trigger dengan Custom Version:**

```yaml
# Di GitHub Actions UI
Run workflow:
  Branch: master
  Version name: 2.3.0-beta  ← Custom version
```

APK output akan pakai version name yang diinput:
```
MikuRay_2.3.0-beta-arm64-v8a-release-20260822-1430.apk
```

---

## 📝 Changelog untuk GitHub Release

Jika Anda push tag, workflow akan otomatis create GitHub Release dengan auto-generated release notes dari commits.

**Contoh:**

```bash
# Create tag dengan message
git tag -a v2.2.9 -m "Release 2.2.9 - Bug fixes"

# Push tag ke GitHub
git push origin v2.2.9
```

Workflow akan:
1. Build & sign APK
2. Create GitHub Release dengan title "v2.2.9"
3. Attach APK files ke release
4. Generate changelog dari commits sejak tag sebelumnya

---

## 🔧 Customization

### **Ubah Branch Trigger:**

Edit `.github/workflows/build-release.yml`:

```yaml
on:
  push:
    branches:
      - main          # ← Ubah sesuai branch Anda
      - develop       # Bisa tambah branch lain
```

### **Ubah ABIs yang Dibangun:**

Edit `V2rayNG/app/build.gradle.kts`:

```kotlin
include(
    "arm64-v8a",      // 64-bit ARM (modern devices)
    "armeabi-v7a",    // 32-bit ARM (older devices)
    // "x86",         // Uncomment untuk emulator
    // "x86_64"       // Uncomment untuk emulator
)
```

### **Ubah Retention Days:**

Edit workflow YAML:

```yaml
- name: Upload signed APKs
  uses: actions/upload-artifact@v4
  with:
    retention-days: 90  # ← Ubah sesuai kebutuhan (max 90)
```

---

## 📚 Resources

- **GitHub Actions Docs:** https://docs.github.com/en/actions
- **r0adkll/sign-android-release:** https://github.com/r0adkll/sign-android-release
- **Android Developer Guide:** https://developer.android.com/studio/publish/app-signing

---

## ✅ Checklist Setup

Sebelum push pertama kali, pastikan:

- [ ] File keystore `.jks` sudah di-convert ke Base64
- [ ] 4 GitHub Secrets sudah ditambahkan (`KEYSTORE_BASE64`, `KEY_ALIAS`, `KEYSTORE_PASSWORD`, `KEY_PASSWORD`)
- [ ] File workflow sudah di-commit (`.github/workflows/build-release.yml` dan `build-debug.yml`)
- [ ] Branch trigger sudah sesuai (master/main)
- [ ] Test build lokal berhasil (`./gradlew assembleRelease`)
- [ ] Keystore password dan alias sudah dicatat dengan aman

---

**Setup Complete! 🎉**

Push code Anda ke GitHub dan workflow akan otomatis build & sign APK untuk Anda!
