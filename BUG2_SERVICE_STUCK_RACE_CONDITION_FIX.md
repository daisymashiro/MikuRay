# Bug #2 Fix: Service Stuck (Race Condition in isStartingLock)

## Fix Implementation Summary

**Date:** 2026-08-21  
**File Modified:** `V2rayNG/app/src/main/java/com/v2ray/ang/service/CoreVpnService.kt`  
**Status:** ✅ IMPLEMENTED

---

## Problems Identified

### 1. **Premature Unlock (Line 126)**
```kotlin
// BEFORE (BROKEN):
if (isSystemVpnStart) {
    unlockStart()  // ❌ Called BEFORE tryLockStart()
}
if (!tryLockStart()) {
    return START_STICKY
}
```
**Issue:** `unlockStart()` called before `tryLockStart()`, causing race condition where lock is released prematurely.

### 2. **No Timeout Mechanism**
```kotlin
// BEFORE (BROKEN):
fun tryLockStart(): Boolean {
    return isStartingLock.compareAndSet(false, true)
}
```
**Issue:** If `unlockStart()` is missed due to exception or crash, lock stays stuck forever.

### 3. **No Lock Reset on Service Creation**
**Issue:** Lock state can persist across service recreations, causing service to be permanently stuck.

---

## Fixes Implemented

### Fix #1: Add Timestamp-Based Timeout (Lines 50, 387-404)

**Added field:**
```kotlin
private val startingTimestamp = AtomicLong(0) // Timestamp for lock timeout detection
```

**Enhanced tryLockStart():**
```kotlin
fun tryLockStart(): Boolean {
    val now = System.currentTimeMillis()
    val lastStart = startingTimestamp.get()
    
    // FIX Bug #2: Add timeout mechanism - if locked for >30 seconds, force unlock
    if (isStartingLock.get() && lastStart > 0 && (now - lastStart) > 30_000) {
        LogUtil.w(AppConfig.TAG, "StartCore-VPN: Lock timeout detected (>30s), force unlock")
        isStartingLock.set(false)
        startingTimestamp.set(0)
    }
    
    val lockAcquired = isStartingLock.compareAndSet(false, true)
    if (lockAcquired) {
        startingTimestamp.set(now)
        LogUtil.w(AppConfig.TAG, "StartCore-VPN: tryLockStart: Lock acquired at $now")
    } else {
        LogUtil.w(AppConfig.TAG, "StartCore-VPN: tryLockStart: Lock already held since $lastStart")
    }
    return lockAcquired
}
```

**Benefits:**
- Lock automatically unlocks after 30 seconds
- Prevents permanent stuck state
- Logs timeout events for debugging

### Fix #2: Enhanced unlockStart() with Defensive Logging (Lines 407-415)

```kotlin
fun unlockStart() {
    val wasLocked = isStartingLock.getAndSet(false)
    startingTimestamp.set(0)
    if (wasLocked) {
        LogUtil.w(AppConfig.TAG, "StartCore-VPN: unlockStart: Lock released")
    } else {
        LogUtil.w(AppConfig.TAG, "StartCore-VPN: unlockStart: Lock was already unlocked (redundant call)")
    }
}
```

**Benefits:**
- Detects redundant unlock calls
- Helps identify sequencing issues in logs

### Fix #3: Lock Reset on onCreate() (Lines 58-61)

```kotlin
override fun onCreate() {
    super.onCreate()
    LogUtil.i(AppConfig.TAG, "StartCore-VPN: Service created")
    
    // FIX Bug #2: Reset lock on service creation to prevent stuck state
    isStartingLock.set(false)
    startingTimestamp.set(0)
    LogUtil.i(AppConfig.TAG, "StartCore-VPN: Lock state reset on onCreate")
    
    // ... rest of code
}
```

**Benefits:**
- Ensures clean state on service creation
- Prevents stuck lock from persisting across service recreations

### Fix #4: Remove Premature unlockStart() Call (Lines 134-138)

**BEFORE:**
```kotlin
if (isSystemVpnStart) {
    unlockStart()  // ❌ WRONG: Called before lock acquired
}
```

**AFTER:**
```kotlin
// FIX Bug #2: Remove premature unlockStart() call
// System VPN restart should rely on timeout mechanism instead of premature unlock
if (isSystemVpnStart) {
    LogUtil.w(AppConfig.TAG, "StartCore-VPN: System VPN start detected, lock will timeout if stuck")
}
```

**Benefits:**
- Removes race condition
- Relies on timeout mechanism for stuck lock recovery
- Proper lock sequencing: tryLock → work → unlock

---

## Lock Usage Points (All Verified)

| Line | Location | Action | Status |
|------|----------|--------|--------|
| 59-60 | onCreate() | Reset lock | ✅ ADDED (Fix #3) |
| 122 | onDestroy() | unlockStart() | ✅ OK |
| 134-138 | onStartCommand() | ~~unlockStart()~~ → Removed | ✅ FIXED (Fix #4) |
| 140 | onStartCommand() | tryLockStart() | ✅ OK |
| 156 | onStartCommand failure | unlockStart() | ✅ OK |
| 161 | onStartCommand success | unlockStart() | ✅ OK |
| 347 | stopAllService() | unlockStart() | ✅ OK |
| 387-404 | tryLockStart() | Lock with timeout | ✅ ENHANCED (Fix #1) |
| 407-415 | unlockStart() | Unlock with logging | ✅ ENHANCED (Fix #2) |

---

## Changes Summary

### Modified Imports (Line 40-41)
```kotlin
import java.util.concurrent.atomic.AtomicBoolean
import java.util.concurrent.atomic.AtomicLong  // NEW
```

### Modified Fields (Line 49-50)
```kotlin
private val isStartingLock = AtomicBoolean(false)
private val startingTimestamp = AtomicLong(0)  // NEW
```

### Total Lines Changed: 4 sections
1. **Imports** (Line 40-41): Added AtomicLong import
2. **Fields** (Line 49-50): Added startingTimestamp field
3. **onCreate()** (Line 54-69): Added lock reset logic
4. **onStartCommand()** (Line 130-143): Removed premature unlock
5. **tryLockStart()** (Line 386-405): Added timeout mechanism
6. **unlockStart()** (Line 407-415): Added defensive logging

---

## Verification Checklist

✅ **Lock cannot get stuck permanently**
- 30-second timeout ensures automatic recovery
- onCreate() reset prevents persistent stuck state

✅ **Service can recover from stuck lock**
- Timeout mechanism auto-unlocks after 30s
- System VPN restart will trigger timeout check

✅ **No premature unlocks**
- Removed line 126 premature `unlockStart()` call
- Lock sequencing now correct: tryLock → work → unlock

✅ **Defensive logging added**
- Lock acquisition logged with timestamp
- Lock release logged with state verification
- Timeout events logged for debugging
- Redundant unlock calls detected and logged

✅ **No breaking changes**
- Existing error handling preserved
- All unlock points maintained
- Return values unchanged
- Service lifecycle unchanged

---

## Testing Recommendations

### Test Case 1: Normal Operation
1. Start VPN service
2. Stop VPN service
3. **Expected:** Lock acquired and released normally

### Test Case 2: Stuck Lock Recovery (Timeout)
1. Simulate stuck lock (comment out unlockStart() temporarily)
2. Wait 30 seconds
3. Try to start service again
4. **Expected:** Lock times out, service starts successfully

### Test Case 3: Service Recreation
1. Start VPN service
2. Force kill app
3. System recreates service
4. **Expected:** Lock reset on onCreate(), service starts successfully

### Test Case 4: System VPN Restart
1. Start VPN via app
2. Toggle VPN via system settings (disable/enable)
3. **Expected:** No premature unlock, timeout handles stuck lock if needed

### Test Case 5: Rapid Start Attempts
1. Tap connect button multiple times rapidly
2. **Expected:** First call acquires lock, subsequent calls blocked until unlock

---

## Expected Log Output

### Normal Start/Stop:
```
I/mikuray: StartCore-VPN: Service created
I/mikuray: StartCore-VPN: Lock state reset on onCreate
W/mikuray: StartCore-VPN: tryLockStart: Lock acquired at 1724256566432
W/mikuray: StartCore-VPN: unlockStart: Lock released
```

### Timeout Recovery:
```
W/mikuray: StartCore-VPN: Lock timeout detected (>30s), force unlock
W/mikuray: StartCore-VPN: tryLockStart: Lock acquired at 1724256596432
```

### Redundant Unlock:
```
W/mikuray: StartCore-VPN: unlockStart: Lock was already unlocked (redundant call)
```

---

## Root Cause Analysis

**Original Issue:** Race condition caused by:
1. Lock released before acquired (line 126)
2. No timeout → permanent stuck if unlock missed
3. No reset on service creation → stuck state persists

**Why it caused "Service Stuck":**
- User taps connect
- System VPN start detected → premature `unlockStart()` at line 126
- `tryLockStart()` at line 128 acquires lock
- If exception occurs before `unlockStart()` at line 149/156
- Lock stays true forever
- All future connect attempts blocked at line 128-131
- User must force close app to reset

**Fix Strategy:**
- Remove premature unlock (line 126)
- Add timeout to auto-recover (30s)
- Reset lock on onCreate() for clean state
- Add defensive logging to detect issues

---

## Files Modified

**Single file changed:**
- `V2rayNG/app/src/main/java/com/v2ray/ang/service/CoreVpnService.kt`

**Total changes:**
- **1 import added** (AtomicLong)
- **1 field added** (startingTimestamp)
- **3 sections modified** (onCreate, onStartCommand, lock methods)
- **0 breaking changes**
- **~30 lines modified/added**

---

## Completion Status

✅ All problems identified  
✅ All fixes implemented  
✅ Minimal changes principle followed  
✅ Clear comments added  
✅ Defensive logging added  
✅ No breaking changes  
✅ Lock sequencing corrected  
✅ Timeout mechanism added  
✅ Lock reset on onCreate added  

**Status: READY FOR TESTING**
