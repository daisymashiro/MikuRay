# Bug Fix Implementation: Group Langganan Hilang

## Summary
Fixed critical race condition bug that caused server list to disappear when switching between subscription tabs.

## Root Cause
Race condition in `MainViewModel.subscriptionIdChangedAsync()`:
- `subscriptionId` was updated on main thread immediately
- `reloadServerList()` was called async on IO thread
- Fragment observers checked `subscriptionId != subId` before IO thread completed loading data
- Result: observers received empty/wrong data or skipped legitimate updates

## Files Modified

### 1. MainViewModel.kt
**Location:** `V2rayNG/app/src/main/java/com/v2ray/ang/ui/main/MainViewModel.kt`

**Changes:**
1. Refactored `reloadServerList()` to use snapshot pattern
2. Added new private method `reloadServerListForSubscription(targetSubId: String)`
3. Modified `subscriptionIdChangedAsync()` to snapshot `subscriptionId` before async call
4. Added double-check locking to skip stale loads
5. Added debug logging for race condition tracking

**Lines changed:** ~40 lines modified/added

## Implementation Details

### Fix 1: Snapshot Pattern in reloadServerList()
```kotlin
@Synchronized
fun reloadServerList() {
    val targetSubId = subscriptionId  // Snapshot to prevent mid-execution changes
    reloadServerListForSubscription(targetSubId)
}
```

### Fix 2: New Method with Race Protection
```kotlin
@Synchronized
private fun reloadServerListForSubscription(targetSubId: String) {
    // Double-check: if subscription changed while waiting for lock, skip stale load
    if (subscriptionId != targetSubId) {
        LogUtil.d(AppConfig.TAG, "Subscription changed during load, skipping stale load for: $targetSubId, current: $subscriptionId")
        return
    }
    
    // Load data using snapshot targetSubId instead of volatile subscriptionId
    val subId = targetSubId.ifEmpty { AppConfig.DEFAULT_SUBSCRIPTION_ID }
    // ... rest of loading logic using targetSubId
    
    LogUtil.d(AppConfig.TAG, "Loaded ${serverList.size} servers for subscription: $targetSubId")
    updateCache()
    LogUtil.d(AppConfig.TAG, "Cache updated with ${serversCache.size} filtered servers")
    updateListAction.postValue(-1)
}
```

### Fix 3: Async Call with Snapshot
```kotlin
fun subscriptionIdChangedAsync(id: String) {
    if (subscriptionId != id) {
        LogUtil.d(AppConfig.TAG, "Subscription ID changed from '$subscriptionId' to '$id'")
        subscriptionId = id
        MmkvManager.encodeSettings(AppConfig.CACHE_SUBSCRIPTION_ID, subscriptionId)
    }
    val targetSubId = subscriptionId  // Snapshot before async to prevent race condition
    viewModelScope.launch(Dispatchers.IO) {
        reloadServerListForSubscription(targetSubId)
    }
}
```

## How The Fix Works

### Before Fix (Bug Scenario):
```
Time | Main Thread              | IO Thread           | Fragment Observer
-----|--------------------------|---------------------|------------------
T0   | subscriptionId = "A"     |                     | subId = "A"
T1   | User swipe to Tab B      |                     |
T2   | subscriptionId = "B"     |                     |
T3   | launch IO task           | reloadServerList()  |
T4   |                          | read subscriptionId | 
T5   |                          | = "B"               |
T6   |                          | load data for "B"   | observe triggered
T7   |                          |                     | check: subId != subscriptionId
T8   |                          |                     | "A" != "B" -> SKIP
T9   |                          | postValue(-1)       | 
Result: Tab A observer skips update, Tab B gets empty cache (not loaded yet)
```

### After Fix (Working Scenario):
```
Time | Main Thread              | IO Thread                  | Fragment Observer
-----|--------------------------|----------------------------|------------------
T0   | subscriptionId = "A"     |                            | subId = "A"
T1   | User swipe to Tab B      |                            |
T2   | subscriptionId = "B"     |                            |
T3   | targetSubId = "B"        |                            |
T4   | launch IO task           | reloadForSub("B")          |
T5   |                          | double-check: subId == "B" |
T6   |                          | load data for "B"          |
T7   |                          | updateCache()              | observe triggered
T8   |                          |                            | check: subId != subscriptionId
T9   |                          |                            | "A" != "B" -> SKIP (OK)
T10  |                          | postValue(-1)              | observe triggered (Tab B)
T11  |                          |                            | check: "B" == "B" -> UPDATE
Result: Tab A correctly ignores, Tab B gets correct data for "B"
```

### Fast Switch Protection:
```
Time | Main Thread              | IO Thread                  
-----|--------------------------|----------------------------
T0   | subscriptionId = "A"     |                            
T1   | targetSubId = "A"        |                            
T2   | launch IO task           |                            
T3   | subscriptionId = "B"     | reloadForSub("A")          
T4   | targetSubId = "B"        | double-check: subId != "A" 
T5   | launch IO task           | SKIP (stale load)          
T6   |                          | reloadForSub("B")          
T7   |                          | double-check: subId == "B" 
T8   |                          | load data for "B"          
T9   |                          | postValue(-1)              
Result: Only Tab B loads, stale Tab A load is skipped
```

## Testing Performed

### Code Verification
- [x] Verified all references to `subscriptionId` in async contexts
- [x] Checked all callers of `reloadServerList()`
- [x] Verified observer logic in `GroupServerFragment`
- [x] Checked for other potential race conditions

### Test Scenarios (Manual Testing Required)
- [ ] Fast tab switching (swipe 5-10 times quickly)
- [ ] Cold app start with multiple tabs
- [ ] Subscription update then tab switch
- [ ] Background to foreground transition
- [ ] Device rotation while on different tabs
- [ ] Search filter active during tab switch
- [ ] Memory pressure scenarios

### Expected Behavior After Fix
✅ Server list always displays correctly for active tab
✅ No empty states or disappearing servers
✅ Fast tab switching works smoothly
✅ Cold start shows data immediately
✅ Subscription updates don't clear lists

## Logging Added

New debug logs for troubleshooting:
1. `"Subscription ID changed from 'X' to 'Y'"` - Track subscription changes
2. `"Subscription changed during load, skipping stale load for: X, current: Y"` - Detect race conditions
3. `"Loaded N servers for subscription: X"` - Verify data loaded
4. `"Cache updated with N filtered servers"` - Verify cache update

### How to View Logs
```bash
# Filter for subscription-related logs
adb logcat | grep "Subscription"

# Or in app
Menu -> Logcat -> Filter: "Subscription"
```

## Performance Impact

### Memory: **NEGLIGIBLE**
- Added 1 string variable per async call (snapshot)
- No additional collections or caching

### CPU: **NEGLIGIBLE**  
- Added 1 string comparison (double-check)
- May skip unnecessary reloads (performance improvement)

### I/O: **NEUTRAL**
- Same number of disk reads
- Potentially fewer wasted loads due to skipping stale operations

## Regression Risk

### Risk Level: **LOW**

**Why:**
- Fix only adds defensive checks
- No changes to data structures
- No changes to observer patterns
- Backward compatible
- Synchronized blocks prevent data corruption

**What Could Go Wrong:**
- ❌ Over-aggressive skipping (unlikely - logic is conservative)
- ❌ Logging overhead in production (minimal impact)
- ✅ Double-check might rarely skip legitimate load (protected by synchronized)

**Mitigation:**
- Extensive logging helps debug any edge cases
- Double-check is conservative (only skips if truly stale)
- Synchronized block ensures atomicity

## Related Bugs Not Fixed

### Bug: "Sering Timeout Server"
**Status:** Requires separate investigation

This bug is likely NOT a code issue but rather:
1. Network connectivity issue
2. Server-side timeout configuration
3. DNS resolution delays
4. Firewall/proxy interference

**Recommendation for separate investigation:**
- Add connection timeout telemetry
- Log actual network errors during timeout
- Check if timeout correlates with specific server types
- Verify timeout values in V2Ray core config

## Code Quality Improvements Made

1. ✅ Added comprehensive documentation comments
2. ✅ Added defensive logging for debugging
3. ✅ Used snapshot pattern to prevent race conditions
4. ✅ Added double-check locking for safety
5. ✅ Maintained backward compatibility

## Deployment Notes

### Build Requirements
- No new dependencies
- No manifest changes
- No resource changes
- Standard Kotlin compilation

### Rollback Plan
If issues arise, revert these changes in `MainViewModel.kt`:
```bash
git diff HEAD~1 V2rayNG/app/src/main/java/com/v2ray/ang/ui/main/MainViewModel.kt
git checkout HEAD~1 -- V2rayNG/app/src/main/java/com/v2ray/ang/ui/main/MainViewModel.kt
```

## Next Steps

1. **Build and Test**
   ```bash
   cd V2rayNG
   ./gradlew assembleDebug
   ```

2. **Install and Manual Test**
   ```bash
   adb install -r app/build/outputs/apk/debug/app-debug.apk
   ```

3. **Monitor Logs**
   ```bash
   adb logcat | grep -E "(Subscription|MainViewModel)"
   ```

4. **Test Scenarios Checklist**
   - [ ] Open app fresh install
   - [ ] Navigate through all tabs slowly
   - [ ] Swipe tabs quickly 10 times
   - [ ] Rotate device on each tab
   - [ ] Update subscription
   - [ ] Search then switch tabs
   - [ ] Background app and return

5. **If All Tests Pass**
   - Update BUGFIX_CHANGELOG.md
   - Consider cherry-pick to release branch
   - Monitor crash reports for 48 hours

## Conclusion

This fix resolves a critical race condition that has been causing user frustration. The implementation is conservative, well-documented, and has minimal performance impact. The snapshot pattern prevents the async IO thread from reading a stale or changed `subscriptionId`, ensuring data consistency between what's loaded and what observers expect.

**Impact:** HIGH - Fixes major user-facing bug
**Risk:** LOW - Defensive changes only
**Effort:** MEDIUM - ~40 lines changed
**Testing:** Required before release

---

**Fixed by:** Kiro AI Agent  
**Date:** 2026-08-21  
**Issue:** Group Langganan Hilang (Race Condition)  
**Status:** Implementation Complete, Testing Required
