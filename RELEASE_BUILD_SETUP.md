# 🔐 SETUP RELEASE BUILD - Signed APK

**Issue:** Debug APK tidak ada protection/signature  
**Solution:** Setup keystore untuk release build yang signed proper  
**Date:** 2026-08-21

---

## ✅ KEYSTORE SUDAH DI-GENERATE

**File:** `keystore_temp/mikuray_release.jks`  
**Alias:** `mikuray_key`  
**Algorithm:** RSA 2048-bit  
**Validity:** 10,000 days (~27 years)

**⚠️ PENTING:** Keystore ini harus di-backup dan dijaga kerahasiaannya!

---

## 📋 CREDENTIALS INFORMATION

**Simpan info ini dengan aman:**

```
Keystore File: mikuray_release.jks
Store Password: MikuRay2026Secure!
Key Alias: mikuray_key
Key Password: MikuRay2026Secure!
```

**⚠️ JANGAN share password ini di public!**

---

## 🚀 SETUP GITHUB SECRETS

### Step 1: Add Secrets ke GitHub Repository

1. **Buka GitHub Settings:**
   ```
   https://github.com/daisymashiro/MikuRay/settings/secrets/actions
   ```

2. **Click "New repository secret"**

3. **Add 4 secrets:**

#### Secret #1: APP_KEYSTORE_BASE64
- **Name:** `APP_KEYSTORE_BASE64`
- **Value:** (copy dari file `mikuray_keystore_base64.txt`)

#### Secret #2: APP_KEYSTORE_PASSWORD
- **Name:** `APP_KEYSTORE_PASSWORD`
- **Value:** `MikuRay2026Secure!`

#### Secret #3: APP_KEYSTORE_ALIAS
- **Name:** `APP_KEYSTORE_ALIAS`
- **Value:** `mikuray_key`

#### Secret #4: APP_KEY_PASSWORD
- **Name:** `APP_KEY_PASSWORD`
- **Value:** `MikuRay2026Secure!`

---

## 🏗️ BUILD RELEASE APK

### Method 1: Via GitHub Actions (Recommended)

**Trigger manual release build:**

1. Buka: https://github.com/daisymashiro/MikuRay/actions/workflows/build.yml

2. Click "Run workflow"

3. Select:
   - **Branch:** `master`
   - **Build Type:** `release`
   - **Release Tag:** (optional, biarkan kosong untuk test)

4. Click "Run workflow"

5. Tunggu ~10-15 menit

6. Download APK dari Artifacts:
   - ✅ **armeabi-v7a-release.apk** (ARM7 32-bit, SIGNED)
   - ✅ arm64-v8a-release.apk (SIGNED)
   - ✅ x86-release.apk (SIGNED)

### Method 2: Build Local (jika urgent)

```bash
cd /home/daisy/mayumi/Experimen/golang/github/MikuRay/V2rayNG

./gradlew assembleRelease \
  -PKS_PATH="../keystore_temp/mikuray_release.jks" \
  -PKS_PASS="MikuRay2026Secure!" \
  -PKS_ALIAS="mikuray_key" \
  -PKEY_PASS="MikuRay2026Secure!"

# APK output:
# app/build/outputs/apk/release/*.apk
```

---

## 🔒 PERBEDAAN DEBUG vs RELEASE

| Feature | Debug APK | Release APK (Signed) |
|---------|-----------|----------------------|
| **Signature** | Debug key (auto) | Production key ✅ |
| **Protection** | ❌ None | ✅ Signed & verified |
| **Installable** | ✅ Yes | ✅ Yes |
| **Play Store** | ❌ No | ✅ Yes |
| **Update** | ❌ Conflict | ✅ Smooth update |
| **Minify** | ❌ No | ✅ Yes (smaller) |
| **Obfuscation** | ❌ No | ✅ Yes (ProGuard) |
| **Size** | ~50 MB | ~40 MB (optimized) |

---

## 📦 VERIFY APK SIGNATURE

Setelah build release, verify signature:

```bash
# Check APK signature
keytool -printcert -jarfile MikuRay_2.2.9-armeabi-v7a-release.apk

# Should show:
# Owner: CN=MikuRay, OU=Development, O=MikuRay, L=Jakarta, ST=Jakarta, C=ID
# Certificate fingerprints:
#   SHA256: ...
```

Atau pakai Android Studio:
```
Menu → Build → Analyze APK → Select APK → Lihat "Signing certificate"
```

---

## ⚠️ BACKUP KEYSTORE

**SANGAT PENTING:**

```bash
# Backup keystore ke tempat aman
cp keystore_temp/mikuray_release.jks ~/Backups/MikuRay_Keystore_BACKUP_2026.jks

# Atau upload ke cloud storage (encrypted)
# JANGAN commit ke git!
```

**Jika keystore hilang:**
- ❌ Tidak bisa update app di Play Store
- ❌ Tidak bisa update app untuk existing users
- ❌ Harus publish sebagai app baru

**Keystore ini = "kunci" app Anda selamanya!**

---

## 🎯 RECOMMENDED WORKFLOW

### Untuk Development (Testing):
```bash
# Build debug (quick, no signature needed)
git push origin master
# GitHub Actions build debug APK automatically
```

### Untuk Production (Release ke User):
```bash
# 1. Test dulu dengan debug
# 2. Jika OK, trigger release build:
#    - GitHub Actions → Run workflow → release
# 3. Download signed APK
# 4. Verify signature
# 5. Distribute ke users
```

---

## 📝 KEYSTORE INFO (For Reference)

```
Certificate Details:
- Owner: CN=MikuRay, OU=Development, O=MikuRay, L=Jakarta, ST=Jakarta, C=ID
- Algorithm: RSA (2048-bit)
- Signature: SHA256withRSA
- Valid: 10,000 days (until ~2053)
- Keystore Type: JKS
- Keystore Format: PKCS12 compatible
```

---

## 🔄 JIKA PERLU GANTI PASSWORD

```bash
# Change keystore password
keytool -storepasswd -keystore mikuray_release.jks

# Change key password
keytool -keypasswd -alias mikuray_key -keystore mikuray_release.jks
```

Lalu update GitHub Secrets dengan password baru.

---

## 🚨 TROUBLESHOOTING

### Error: "Failed to decode keystore"
- Check apakah base64 encoding benar
- Re-generate: `base64 mikuray_release.jks > keystore_base64.txt`
- Copy ulang ke GitHub Secrets

### Error: "Incorrect password"
- Pastikan password di GitHub Secrets sama persis
- Case-sensitive: `MikuRay2026Secure!`

### Error: "Key alias not found"
- Check alias: `keytool -list -keystore mikuray_release.jks`
- Pastikan alias di Secrets = `mikuray_key`

---

## ✅ NEXT STEPS

1. **Add secrets ke GitHub** (4 secrets)
2. **Trigger release build** (GitHub Actions)
3. **Download signed APK** dari artifacts
4. **Verify signature** (keytool -printcert)
5. **Test install** di device
6. **Backup keystore** ke tempat aman

---

## 📊 FILES LOCATION

```
keystore_temp/
├── mikuray_release.jks              # Keystore file (2.7 KB)
└── mikuray_keystore_base64.txt      # Base64 encoded (for GitHub Secret)
```

**⚠️ JANGAN commit folder ini ke git!**

---

**Created:** 2026-08-21  
**Keystore Validity:** Until 2053  
**Status:** Ready for GitHub Secrets setup
