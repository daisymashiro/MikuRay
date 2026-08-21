# 🧪 TESTING GUIDE - Bug Fix Verification

**Version:** 2.2.9 (Build 739)  
**Date:** 2026-08-21  
**Fixes:** Bug #1 (Memory Leak) & Bug #2 (Race Condition)

---

## 📱 Test Environment Requirements

### Minimum Test Devices

**For Bug #1 (Memory Leak):**
- ✅ Any Android device (SDK 24+)
- ✅ Recommended: Device with 2GB RAM or less
- ✅ Android Studio Profiler (untuk memory monitoring)

**For Bug #2 (Race Condition):**
- ✅ Any Android device (SDK 24+)
- ✅ adb tools installed

**For ARM7 32-bit Compatibility:**
- ✅ Old device dengan ARM7 processor (32-bit)
- ✅ Contoh: Samsung J-series, Xiaomi Redmi 4A, dll

---

## 🧪 Test Scenario 1: Memory Leak Fix Verification

**Objective:** Verify MediaPlayer di-release setelah audio selesai

**Steps:**
1. Enable sound notification:
   - Settings → Sound on Connect = ON
   
2. Connect Android Profiler:
   ```bash
   # Launch Android Studio Profiler
   # Or use command line:
   adb shell dumpsys meminfo com.miku.ray
   ```

3. Perform 20 connect/disconnect cycles:
   - Connect VPN
   - Wait for audio to finish
   - Disconnect VPN
   - Wait for audio to finish
   - Repeat 20x

4. Monitor memory usage:
   ```bash
   # Before test
   adb shell dumpsys meminfo com.miku.ray | grep "TOTAL"
   
   # After 20 cycles
   adb shell dumpsys meminfo com.miku.ray | grep "TOTAL"
   ```

**Expected Result:**
- ✅ Memory usage stabil (±5% variance)
- ✅ No gradual memory increase
- ✅ MediaPlayer instances = 0 atau 1 (not accumulating)

**Failure Criteria:**
- ❌ Memory increases by >500KB after 20 cycles
- ❌ Multiple MediaPlayer instances in heap dump

---

## 🧪 Test Scenario 2: Race Condition Fix Verification

**Objective:** Verify service restart konsisten dan tidak stuck

### Test 2A: Rapid Start Test

**Steps:**
1. Tap connect button rapidly 5x dalam 2 detik
2. Observe service state
3. Check logs:
   ```bash
   adb logcat | grep "StartCore-VPN"
   ```

**Expected Result:**
- ✅ Service starts successfully (tidak stuck)
- ✅ Log shows "Start already in progress" untuk duplicate attempts
- ✅ VPN connects normally setelah lock released

**Failure Criteria:**
- ❌ Service stuck dengan notification "Connecting..."
- ❌ Must force stop app to recover

### Test 2B: System Kill Recovery Test

**Steps:**
1. Connect VPN
2. Force kill service:
   ```bash
   adb shell am force-stop com.miku.ray
   # Or
   adb shell am kill com.miku.ray
   ```
3. Wait 5-10 seconds
4. Check if service auto-restarts:
   ```bash
   adb shell dumpsys activity services | grep CoreVpnService
   ```

**Expected Result:**
- ✅ Service restarts automatically
- ✅ VPN reconnects tanpa user action
- ✅ No "silent disconnect"

**Failure Criteria:**
- ❌ Service tidak restart
- ❌ VPN notification hilang permanent
- ❌ Must manually reconnect

### Test 2C: Low Memory Kill Test

**Steps:**
1. Connect VPN
2. Simulate low memory:
   ```bash
   # Launch heavy apps atau:
   adb shell am start -a android.intent.action.VIEW -d "chrome://about"
   # Open multiple tabs
   ```
3. Wait for system to kill background processes
4. Return to MikuRay

**Expected Result:**
- ✅ Service survives atau restarts automatically
- ✅ START_STICKY behavior working

---

## 🧪 Test Scenario 3: ARM7 32-bit Compatibility

**Objective:** Verify APK berjalan di ARM7 devices

**Steps:**
1. Install APK armeabi-v7a variant:
   ```bash
   adb install MikuRay_2.2.9-armeabi-v7a-debug-*.apk
   ```

2. Launch app
3. Test basic functionality:
   - ✅ App launches
   - ✅ Can add server
   - ✅ Can connect VPN
   - ✅ Audio notification plays
   - ✅ No crashes

**Expected Result:**
- ✅ App berjalan lancar di ARM7 device
- ✅ All features working

**Failure Criteria:**
- ❌ App crashes on launch
- ❌ "Unsupported architecture" error
- ❌ Native library load failure

---

## 🧪 Test Scenario 4: Regression Testing

**Objective:** Ensure fixes tidak introduce new bugs

**Areas to Test:**
1. **Audio Notification:**
   - ✅ Connect sound plays
   - ✅ Disconnect sound plays
   - ✅ Sound can be disabled in settings
   - ✅ No audio glitches

2. **VPN Connection:**
   - ✅ Connect successful
   - ✅ Disconnect successful
   - ✅ Network traffic routes through VPN
   - ✅ DNS working

3. **Service Lifecycle:**
   - ✅ Service starts properly
   - ✅ Service stops properly
   - ✅ No zombie processes
   - ✅ Notification updates correctly

4. **Performance:**
   - ✅ No UI lag
   - ✅ No ANR (Application Not Responding)
   - ✅ Battery usage normal
   - ✅ Network speed normal

---

## 📊 Test Report Template

```markdown
## Test Report

**Date:** YYYY-MM-DD
**Tester:** [Name]
**Device:** [Model]
**Android Version:** [Version]
**APK Variant:** [arm64-v8a / armeabi-v7a / x86_64 / x86]

### Scenario 1: Memory Leak
- [ ] PASS / [ ] FAIL
- Notes: 

### Scenario 2A: Rapid Start
- [ ] PASS / [ ] FAIL
- Notes:

### Scenario 2B: System Kill Recovery
- [ ] PASS / [ ] FAIL
- Notes:

### Scenario 2C: Low Memory Kill
- [ ] PASS / [ ] FAIL
- Notes:

### Scenario 3: ARM7 Compatibility (if applicable)
- [ ] PASS / [ ] FAIL
- Notes:

### Scenario 4: Regression
- [ ] PASS / [ ] FAIL
- Notes:

### Overall Result
- [ ] ✅ APPROVED FOR RELEASE
- [ ] ❌ NEEDS MORE WORK
- [ ] ⚠️ APPROVED WITH KNOWN ISSUES

### Additional Notes:
[Any other observations]
```

---

## 🔍 Debug Commands

**Check service status:**
```bash
adb shell dumpsys activity services | grep -A 20 CoreVpnService
```

**Monitor logs in real-time:**
```bash
adb logcat -c && adb logcat | grep -E "StartCore|SoundPlayer"
```

**Check memory info:**
```bash
adb shell dumpsys meminfo com.miku.ray
```

**List installed APK info:**
```bash
adb shell pm dump com.miku.ray | grep -E "versionCode|versionName|nativeLibraryPath"
```

**Check native libraries:**
```bash
adb shell ls -la /data/app/com.miku.ray-*/lib/
```

---

## ✅ Sign-off Checklist

Before marking as "Ready for Release":

- [ ] Memory leak test: PASSED
- [ ] Race condition test: PASSED
- [ ] ARM7 compatibility: VERIFIED (if targeting old devices)
- [ ] Regression test: NO NEW BUGS
- [ ] Performance: NORMAL
- [ ] Battery usage: NORMAL
- [ ] User acceptance test: PASSED
- [ ] Documentation: UPDATED

---

**Testing Guide Version:** 1.0  
**Last Updated:** 2026-08-21
