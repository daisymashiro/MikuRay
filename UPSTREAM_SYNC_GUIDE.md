# 🔄 PANDUAN: Sync Fork dengan Upstream (2dust/v2rayNG)

**Repo Anda:** daisymashiro/MikuRay  
**Repo Asli (Upstream):** 2dust/v2rayNG  
**Tujuan:** Update dari upstream sambil maintain bug fixes Anda

---

## 📋 SETUP (Sudah Selesai)

```bash
# Upstream sudah ditambahkan ✅
git remote -v

origin    https://github.com/daisymashiro/MikuRay.git (your fork)
upstream  https://github.com/2dust/v2rayNG.git (original repo)
```

---

## 🔄 CARA SYNC DENGAN UPSTREAM

### **Metode 1: Merge (Recommended untuk Anda)**

Merge update dari upstream ke branch Anda sambil keep bug fixes.

**Langkah:**

```bash
cd /home/daisy/mayumi/Experimen/golang/github/MikuRay

# 1. Fetch update terbaru dari upstream
git fetch upstream

# 2. Pastikan di branch master
git checkout master

# 3. Merge upstream/master ke master Anda
git merge upstream/master

# 4. Jika ada conflict, resolve dulu (lihat section RESOLVE CONFLICT di bawah)

# 5. Push ke fork Anda
git push origin master
```

**Keuntungan:**
- ✅ Bug fixes Anda tetap ada
- ✅ Update dari upstream masuk
- ✅ History lengkap (commit Anda + upstream)

**Kerugian:**
- ⚠️ Bisa ada merge conflict (tapi bisa di-resolve)
- ⚠️ History jadi ada "merge commit"

---

### **Metode 2: Rebase (Advanced)**

Rebase bug fixes Anda di atas update upstream terbaru.

**Langkah:**

```bash
cd /home/daisy/mayumi/Experimen/golang/github/MikuRay

# 1. Fetch update
git fetch upstream

# 2. Checkout master
git checkout master

# 3. Rebase di atas upstream/master
git rebase upstream/master

# 4. Jika ada conflict, resolve lalu:
git rebase --continue

# 5. Force push (karena history berubah)
git push origin master --force
```

**Keuntungan:**
- ✅ History bersih (linear)
- ✅ Bug fixes Anda tetap di atas upstream

**Kerugian:**
- ⚠️ Harus force push (berbahaya jika ada collab)
- ⚠️ Lebih susah resolve conflict
- ⚠️ **Tidak recommended jika repo public atau ada user lain**

---

## 🛠️ RESOLVE MERGE CONFLICT

Jika ada conflict saat merge/rebase:

### **Step 1: Check conflict files**
```bash
git status

# Output:
# Unmerged paths:
#   both modified:   V2rayNG/app/src/main/java/com/v2ray/ang/util/SoundPlayer.kt
#   both modified:   V2rayNG/app/src/main/java/com/v2ray/ang/service/CoreVpnService.kt
```

### **Step 2: Open conflict file**
```bash
# Conflict akan terlihat seperti ini:
<<<<<<< HEAD (your changes)
private fun playSound(context: Context, resId: Int) {
    player?.release()
    player = MediaPlayer.create(context, resId)?.apply {
        setOnCompletionListener { mp ->
            mp.release()
            if (player === mp) {
                player = null
            }
        }
        start()
    }
}
=======
// Upstream changes (different code)
private fun playSound(context: Context, resId: Int) {
    player?.release()
    player = MediaPlayer.create(context, resId)
    player?.start()
}
>>>>>>> upstream/master
```

### **Step 3: Resolve conflict**

**Pilih salah satu:**

**A. Keep your changes (bug fix):**
```kotlin
private fun playSound(context: Context, resId: Int) {
    player?.release()
    player = MediaPlayer.create(context, resId)?.apply {
        setOnCompletionListener { mp ->
            mp.release()
            if (player === mp) {
                player = null
            }
        }
        start()
    }
}
```

**B. Atau merge keduanya (combine changes)**

### **Step 4: Mark as resolved**
```bash
git add V2rayNG/app/src/main/java/com/v2ray/ang/util/SoundPlayer.kt
```

### **Step 5: Complete merge**
```bash
git commit -m "merge: Sync with upstream/master, keep bug fixes"
```

### **Step 6: Push**
```bash
git push origin master
```

---

## 📅 WORKFLOW RUTIN

**Setiap kali upstream update (misalnya seminggu sekali):**

```bash
# 1. Fetch upstream
git fetch upstream

# 2. Check apakah ada update
git log HEAD..upstream/master --oneline

# Jika ada update:

# 3. Merge upstream
git merge upstream/master

# 4. Jika ada conflict, resolve
# (lihat section RESOLVE CONFLICT)

# 5. Push ke fork
git push origin master

# 6. Build & test
# GitHub Actions akan auto-build
```

---

## 🎯 STRATEGI: MAINTAIN BUG FIXES

### **Option A: Keep Fixes in Master (Current)**

**Struktur:**
```
master (your fork)
  ├─ upstream updates (merged)
  └─ your bug fixes (on top)
```

**Pros:**
- ✅ Simple
- ✅ Semua fix langsung available

**Cons:**
- ⚠️ Harus resolve conflict tiap sync
- ⚠️ Sulit track mana fix Anda vs upstream

---

### **Option B: Separate Branch untuk Fixes (Recommended)**

**Struktur:**
```
master (clean, sync dengan upstream)
  └─ bugfix-branch (your fixes on top)
```

**Setup:**
```bash
# 1. Buat branch untuk bug fixes
git checkout -b mikuray-bugfixes

# 2. Cherry-pick bug fixes Anda
git cherry-pick 00a8942d  # Bug #1 & #2 fix commit
git cherry-pick dba7e517  # CI/CD fix commit

# 3. Push branch
git push origin mikuray-bugfixes

# 4. Update build.yml untuk build dari branch ini
```

**Workflow:**
```bash
# Sync upstream:
git checkout master
git fetch upstream
git merge upstream/master
git push origin master

# Rebase bug fixes di atas master baru:
git checkout mikuray-bugfixes
git rebase master
git push origin mikuray-bugfixes --force

# Build dari mikuray-bugfixes branch
```

**Pros:**
- ✅ Master tetap clean (mirror upstream)
- ✅ Bug fixes di branch terpisah
- ✅ Mudah track changes
- ✅ Bisa compare dengan upstream

**Cons:**
- ⚠️ Lebih complex
- ⚠️ Perlu rebase tiap update

---

## 🔔 NOTIFIKASI UPDATE UPSTREAM

**Setup GitHub Watch:**
1. Buka: https://github.com/2dust/v2rayNG
2. Click "Watch" → "Custom" → "Releases"
3. Anda akan dapat notif email tiap ada release baru

**Atau check manual:**
```bash
# Check update
git fetch upstream
git log HEAD..upstream/master --oneline

# Lihat changes
git diff HEAD..upstream/master
```

---

## 📝 CONTOH PRAKTIS

### **Scenario 1: Upstream update, no conflict**

```bash
git fetch upstream
git merge upstream/master
# Auto-merge successful
git push origin master
# Done! ✅
```

### **Scenario 2: Upstream update, dengan conflict**

```bash
git fetch upstream
git merge upstream/master
# CONFLICT in SoundPlayer.kt

# Open file, resolve conflict
# Keep your bug fix + any new upstream features

git add SoundPlayer.kt
git commit -m "merge: Sync with upstream, preserve memory leak fix"
git push origin master
# Done! ✅
```

### **Scenario 3: Upstream fix bug yang sama**

```bash
git fetch upstream
git log upstream/master --oneline | grep -i "memory\|leak"
# Output: abc1234 fix: Fix MediaPlayer memory leak

# Upstream sudah fix bug yang sama!
# Option A: Revert your fix, pakai upstream
git revert 00a8942d
git merge upstream/master

# Option B: Keep your fix (jika lebih baik)
git merge upstream/master
# Resolve conflict, keep your implementation
```

---

## 🚨 EMERGENCY: RESET KE UPSTREAM

Jika fork Anda rusak dan mau reset total ke upstream:

```bash
# ⚠️ WARNING: Ini akan HAPUS semua changes Anda!

# Backup bug fixes dulu
git branch backup-bugfixes

# Reset ke upstream
git fetch upstream
git reset --hard upstream/master
git push origin master --force

# Lalu cherry-pick bug fixes dari backup
git cherry-pick <commit-hash-bugfix>
```

---

## 📊 CHECK STATUS

**Lihat commits Anda yang belum di upstream:**
```bash
git log upstream/master..HEAD --oneline
```

**Lihat commits upstream yang belum Anda merge:**
```bash
git log HEAD..upstream/master --oneline
```

**Compare files:**
```bash
git diff upstream/master HEAD -- V2rayNG/app/src/main/java/com/v2ray/ang/util/SoundPlayer.kt
```

---

## ✅ RECOMMENDED WORKFLOW UNTUK ANDA

**Karena bug fixes Anda hanya 2 file:**

### **Simple Approach:**

1. **Sync master dengan upstream (merge)**
   ```bash
   git fetch upstream
   git merge upstream/master
   # Resolve conflict jika ada
   git push origin master
   ```

2. **Jika conflict di SoundPlayer.kt atau CoreVpnService.kt:**
   - Keep your bug fix implementation
   - Add any new features dari upstream di bagian lain

3. **Build & test**

**Frequency:** Setiap 1-2 minggu atau ada release baru

---

## 📖 DOKUMENTASI BUG FIXES ANDA

Untuk track mana code Anda vs upstream, tambahkan comment:

```kotlin
// MikuRay-specific: Fix memory leak by releasing MediaPlayer on completion
private fun playSound(context: Context, resId: Int) {
    player?.release()
    player = MediaPlayer.create(context, resId)?.apply {
        setOnCompletionListener { mp ->  // MikuRay fix
            mp.release()
            if (player === mp) {
                player = null
            }
        }
        start()
    }
}
```

Atau buat file `MIKURAY_CHANGES.md` yang list semua changes Anda.

---

## 🎯 SUMMARY

**Setup:** ✅ Done (upstream added)  
**Current state:** Fork dengan 2 bug fixes  
**Upstream:** 2dust/v2rayNG  

**Recommended workflow:**
1. Fetch upstream: `git fetch upstream`
2. Merge: `git merge upstream/master`
3. Resolve conflict (keep your fixes)
4. Push: `git push origin master`
5. Build auto-trigger

**Frequency:** Check update setiap 1-2 minggu

---

**Created:** 2026-08-21  
**Upstream Repo:** https://github.com/2dust/v2rayNG  
**Your Fork:** https://github.com/daisymashiro/MikuRay
