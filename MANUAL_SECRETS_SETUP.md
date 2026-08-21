# 🔐 SETUP GITHUB SECRETS - MANUAL GUIDE

**Karena setup otomatis butuh library tambahan, ini cara manual yang simple.**

---

## 📋 STEP-BY-STEP MANUAL SETUP

### Step 1: Buka GitHub Secrets Settings

**Link:** https://github.com/daisymashiro/MikuRay/settings/secrets/actions

Atau navigasi manual:
1. Buka: https://github.com/daisymashiro/MikuRay
2. Click **Settings**
3. Sidebar kiri → **Secrets and variables** → **Actions**
4. Click **"New repository secret"**

---

### Step 2: Add Secret #1 - Keystore File

**Click "New repository secret"**

**Name:** `APP_KEYSTORE_BASE64`

**Value:** (Copy dari file `keystore_temp/keystore_single_line.txt`)

```bash
# Di terminal, jalankan ini untuk copy ke clipboard:
cat /home/daisy/mayumi/Experimen/golang/github/MikuRay/keystore_temp/keystore_single_line.txt

# Atau buka file dan copy manual
```

**Paste value ke GitHub**, lalu click **"Add secret"**

---

### Step 3: Add Secret #2 - Keystore Password

**Click "New repository secret"**

**Name:** `APP_KEYSTORE_PASSWORD`

**Value:** `MikuRay2026Secure!`

Click **"Add secret"**

---

### Step 4: Add Secret #3 - Key Alias

**Click "New repository secret"**

**Name:** `APP_KEYSTORE_ALIAS`

**Value:** `mikuray_key`

Click **"Add secret"**

---

### Step 5: Add Secret #4 - Key Password

**Click "New repository secret"**

**Name:** `APP_KEY_PASSWORD`

**Value:** `MikuRay2026Secure!`

Click **"Add secret"**

---

## ✅ VERIFY SECRETS

Setelah semua di-add, Anda harus lihat **4 secrets** di page ini:
https://github.com/daisymashiro/MikuRay/settings/secrets/actions

```
✅ APP_KEYSTORE_BASE64      Updated now
✅ APP_KEYSTORE_PASSWORD    Updated now
✅ APP_KEYSTORE_ALIAS       Updated now
✅ APP_KEY_PASSWORD         Updated now
```

---

## 🚀 BUILD RELEASE APK

### Option 1: Via GitHub Actions (Recommended)

**Link:** https://github.com/daisymashiro/MikuRay/actions/workflows/build.yml

**Steps:**
1. Click **"Run workflow"** button (kanan atas)
2. Pilih:
   - **Use workflow from:** `Branch: master`
   - **Build Type:** `release` ← **PENTING!**
   - **Release Tag:** (biarkan kosong untuk test)
3. Click **"Run workflow"** (hijau)
4. Tunggu ~10-15 menit
5. Refresh page, workflow akan muncul
6. Setelah selesai (✅ hijau), click workflow
7. Scroll ke bawah → **Artifacts** section
8. Download:
   - ✅ **armeabi-v7a-release** (ARM7 32-bit, SIGNED)
   - ✅ arm64-v8a-release (SIGNED)
   - ✅ x86-apk-release (SIGNED)

---

### Option 2: Trigger via Push

Setiap kali Anda push ke master, GitHub Actions akan:
- Build **debug** APK (default)

Untuk build **release**, harus manual trigger via UI (Option 1).

---

## 🔍 VERIFY SIGNED APK

Setelah download APK release:

```bash
# Extract ZIP artifact
unzip armeabi-v7a-release.zip

# Check signature
keytool -printcert -jarfile MikuRay_*.apk

# Output harus menunjukkan:
# Owner: CN=MikuRay, OU=Development, O=MikuRay, L=Jakarta...
# Certificate fingerprints:
#   SHA256: ...
```

---

## 📊 DEBUG vs RELEASE COMPARISON

| Feature | Debug (Current) | Release (After Setup) |
|---------|-----------------|------------------------|
| Signature | Debug key | ✅ Production key |
| Protected | ❌ No | ✅ Yes |
| Minified | ❌ No (~50MB) | ✅ Yes (~40MB) |
| Obfuscated | ❌ No | ✅ ProGuard |
| Play Store | ❌ No | ✅ Yes |
| Updates | ❌ Conflict | ✅ Smooth |

---

## 🎯 QUICK REFERENCE

**Secrets yang perlu di-add:**

```
1. APP_KEYSTORE_BASE64     = (isi file keystore_single_line.txt)
2. APP_KEYSTORE_PASSWORD   = MikuRay2026Secure!
3. APP_KEYSTORE_ALIAS      = mikuray_key
4. APP_KEY_PASSWORD        = MikuRay2026Secure!
```

**Links:**
- Add Secrets: https://github.com/daisymashiro/MikuRay/settings/secrets/actions
- Run Workflow: https://github.com/daisymashiro/MikuRay/actions/workflows/build.yml

---

## ⚠️ IMPORTANT NOTES

1. **JANGAN commit keystore file ke git!**
   - File `keystore_temp/` sudah di-ignore
   - Jangan pindahkan ke folder lain yang tracked

2. **BACKUP keystore file!**
   ```bash
   cp keystore_temp/mikuray_release.jks ~/Backups/MikuRay_Keystore_BACKUP.jks
   ```

3. **Password = "kunci" app selamanya**
   - Jangan lupa password: `MikuRay2026Secure!`
   - Simpan di password manager

4. **Keystore valid sampai 2053**
   - Masa berlaku: 10,000 hari
   - Cukup untuk lifetime app

---

## 🚨 TROUBLESHOOTING

**Error: "encodedString value is not set"**
- Secret `APP_KEYSTORE_BASE64` belum di-add atau kosong
- Re-copy value dari `keystore_single_line.txt`

**Error: "Incorrect keystore password"**
- Check password di secret `APP_KEYSTORE_PASSWORD`
- Harus exact: `MikuRay2026Secure!` (case-sensitive)

**Error: "Key alias not found"**
- Check alias di secret `APP_KEYSTORE_ALIAS`
- Harus exact: `mikuray_key`

**Build masih pakai debug key**
- Pastikan pilih **"release"** bukan "debug" saat run workflow
- Check di dropdown "Build Type"

---

## ✅ NEXT STEPS AFTER SETUP

1. ✅ Add 4 secrets ke GitHub
2. ✅ Trigger release build
3. ✅ Download signed APK
4. ✅ Verify signature
5. ✅ Test install
6. ✅ Backup keystore

**Setelah itu, Anda punya APK production-ready yang:**
- ✅ Signed dengan keystore Anda
- ✅ Bisa di-update smooth
- ✅ Bisa di-upload ke Play Store
- ✅ Lebih kecil size (minified)
- ✅ Protected dengan ProGuard

---

**Created:** 2026-08-21  
**For:** MikuRay Release Build Setup  
**Keystore:** Valid until 2053
