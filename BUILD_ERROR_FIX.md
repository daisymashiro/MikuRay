# ⚠️ GITHUB ACTIONS BUILD ERROR - FIX REQUIRED

**Error:** Build failed at "Decode Keystore" step  
**Reason:** Missing GitHub Secrets for signing keystore  
**Time:** 2026-08-21 07:08 UTC

---

## 🔴 ERROR MESSAGE

```
Error: encodedString value is not set
```

**Step yang gagal:**
```yaml
- name: Decode Keystore
  uses: timheuer/base64-to-file@v2.0.0
  id: android_keystore
  with:
    fileName: "android_keystore.jks"
    encodedString: ${{ secrets.APP_KEYSTORE_BASE64 }}  # ← MISSING SECRET
```

---

## 🔍 ROOT CAUSE

GitHub Actions workflow memerlukan **signing keystore secrets** untuk sign APK, tapi secrets belum di-setup di repository.

**Secrets yang dibutuhkan:**
1. `APP_KEYSTORE_BASE64` - Base64 encoded keystore file
2. `APP_KEYSTORE_PASSWORD` - Keystore password
3. `APP_KEYSTORE_ALIAS` - Key alias
4. `APP_KEY_PASSWORD` - Key password

---

## ✅ SOLUSI - ADA 2 OPSI

### **Option 1: Build Debug APK Tanpa Signing (Recommended untuk Testing)**

Modifikasi workflow untuk skip signing pada debug build.

**File:** `.github/workflows/build.yml`

**Change:**
```yaml
# Line 98-104: Make keystore optional for debug
- name: Decode Keystore
  uses: timheuer/base64-to-file@v2.0.0
  id: android_keystore
  if: github.event.inputs.build_type == 'release'  # ← ADD THIS
  with:
    fileName: "android_keystore.jks"
    encodedString: ${{ secrets.APP_KEYSTORE_BASE64 }}

# Line 105-126: Update build step
- name: Build APK (${{ env.BUILD_TYPE }})
  run: |
    cd ${{ github.workspace }}/V2rayNG
    echo "sdk.dir=${ANDROID_HOME}" > local.properties
    chmod 755 gradlew
    
    if [ "${{ env.BUILD_TYPE }}" == "debug" ]; then
      echo "Starting DEBUG build process..."
      # Remove signing params for debug
      ./gradlew assembleDebug --console=plain
    else
      echo "Starting RELEASE build process..."
      ./gradlew assembleRelease --console=plain \
        -PKS_PATH="${{ steps.android_keystore.outputs.filePath }}" \
        -PKS_PASS="${{ secrets.APP_KEYSTORE_PASSWORD }}" \
        -PKS_ALIAS="${{ secrets.APP_KEYSTORE_ALIAS }}" \
        -PKEY_PASS="${{ secrets.APP_KEY_PASSWORD }}"
    fi
```

**Keuntungan:**
- ✅ Debug build tidak perlu signing secrets
- ✅ APK bisa di-install untuk testing
- ✅ Fast fix, no secrets needed

**Kerugian:**
- ⚠️ Debug APK signed dengan debug key (auto-generated)
- ⚠️ Tidak bisa upload ke Play Store
- ⚠️ Tidak bisa update app yang signed dengan key berbeda

---

### **Option 2: Setup Signing Secrets (For Production)**

Setup proper signing keystore untuk production release.

**Step 1: Generate atau gunakan existing keystore**

Jika belum punya keystore:
```bash
keytool -genkey -v -keystore android_keystore.jks \
  -alias miku_ray_key \
  -keyalg RSA -keysize 2048 -validity 10000 \
  -storepass YOUR_PASSWORD \
  -keypass YOUR_KEY_PASSWORD
```

**Step 2: Encode keystore ke Base64**
```bash
base64 android_keystore.jks > keystore_base64.txt
```

**Step 3: Add secrets ke GitHub**
1. Buka: https://github.com/daisymashiro/MikuRay/settings/secrets/actions
2. Click "New repository secret"
3. Add 4 secrets:
   - Name: `APP_KEYSTORE_BASE64`, Value: (isi dari keystore_base64.txt)
   - Name: `APP_KEYSTORE_PASSWORD`, Value: YOUR_PASSWORD
   - Name: `APP_KEYSTORE_ALIAS`, Value: miku_ray_key
   - Name: `APP_KEY_PASSWORD`, Value: YOUR_KEY_PASSWORD

**Keuntungan:**
- ✅ Proper production signing
- ✅ Bisa upload ke Play Store
- ✅ Consistent signing untuk updates

**Kerugian:**
- ⚠️ Perlu setup keystore
- ⚠️ Harus manage passwords safely

---

## 🚀 QUICK FIX (Recommended)

**Saya akan fix workflow untuk skip signing pada debug build:**

Ini paling cepat dan cocok untuk testing. APK debug tetap bisa di-install dan test.

**Apakah saya fix sekarang?**

---

## 📊 BUILD STATUS

**Current Status:** ❌ FAILED  
**Failed Step:** Decode Keystore  
**Reason:** Missing secrets  
**Solution:** Option 1 (skip signing for debug) or Option 2 (add secrets)

---

## ⏭️ NEXT ACTION

**Pilihan Anda:**
1. **Saya fix workflow (skip signing untuk debug)** - 5 menit ✅ Recommended
2. **Anda setup secrets dulu** - 15-30 menit (need keystore)
3. **Skip build, pakai APK lokal** - compile manual

Mana yang Anda pilih?

---

**Time:** 2026-08-21 07:08 UTC  
**Status:** Waiting for decision  
**Priority:** MEDIUM (APK tetap bisa di-build setelah fix)
