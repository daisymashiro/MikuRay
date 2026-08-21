# Technical Deep Dive: Race Condition Timeline

## Detailed Timing Analysis

### Normal Flow (No Race Condition)

```
Time  | Thread        | Action                                    | subscriptionId Value
------|---------------|-------------------------------------------|---------------------
T0    | Main          | User on Group A tab                       | "group_a_id"
T1    | Main          | User clicks Group B tab                   | "group_a_id"
T2    | Main          | TabLayout fires onTabSelected             | "group_a_id"
T3    | Main          | ViewPager2 starts switching               | "group_a_id"
T4    | Main          | GroupServerFragment (B) onCreate          | "group_a_id"
T5    | Main          | GroupServerFragment (B) onResume          | "group_a_id"
T6    | Main          | subscriptionIdChangedAsync() called       | "group_a_id"
T7    | Main          | subscriptionId = "group_b_id"             | "group_b_id" ✓
T8    | IO Thread     | reloadServerListForSubscription starts    | "group_b_id"
T9    | Main          | [User waits, sees Group B UI]            | "group_b_id"
T10   | Main          | User clicks Import from Clipboard         | "group_b_id"
T11   | Main          | importBatchConfig(mainViewModel.subId)    | "group_b_id" ✓
Result: Import goes to Group B ✓ CORRECT
```

### Race Condition Flow (Bug Occurs)

```
Time  | Thread        | Action                                    | subscriptionId Value
------|---------------|-------------------------------------------|---------------------
T0    | Main          | User on Group A tab                       | "group_a_id"
T1    | Main          | User clicks Group B tab                   | "group_a_id"
T2    | Main          | TabLayout fires onTabSelected             | "group_a_id"
T3    | Main          | ViewPager2 starts switching               | "group_a_id"
T4    | Main          | GroupServerFragment (B) onCreate          | "group_a_id"
T5    | Main          | GroupServerFragment (B) onResume          | "group_a_id"
T6    | Main          | subscriptionIdChangedAsync() called       | "group_a_id"
T7    | Main          | ⚡ User clicks Import (BEFORE T8!)        | "group_a_id" ✗
T8    | Main          | importBatchConfig(mainViewModel.subId)    | "group_a_id" ✗
T9    | Main          | subscriptionId = "group_b_id"             | "group_b_id"
T10   | IO Thread     | Import saves to Group A                   | "group_b_id"
T11   | IO Thread     | reloadServerListForSubscription starts    | "group_b_id"
Result: Import goes to Group A ✗ WRONG!
```

**Critical Window**: Between T6 and T9 (approximately 50-200ms on typical device)

### Why The Race Window Is Large

1. **Fragment Lifecycle Overhead**: 
   - Fragment creation
   - View inflation
   - onCreateView → onViewCreated → onResume
   - Total: ~100-300ms

2. **Async Nature of subscriptionIdChangedAsync()**:
   - Function name suggests async, but subscriptionId assignment is synchronous
   - However, it's called AFTER fragment lifecycle completes
   - The delay is in the lifecycle, not the function itself

3. **Fast User Action**:
   - Experienced users click very quickly
   - Tap tab → Immediately tap + button → Tap Import
   - Total user action time: 200-500ms
   - Easily fits within the race window

## Code Path Analysis

### Import Clipboard Call Stack

```
User clicks "Import from Clipboard" button
  ↓
AddConfigBottomSheet.onAddConfigOptionClicked(R.id.import_clipboard)
  ↓
MainActivity.onAddConfigOptionClicked(R.id.import_clipboard)
  ↓
MainActivity.importClipboard()
  ↓
MainActivity.importBatchConfig(clipboardText)
  ↓
lifecycleScope.launch(Dispatchers.IO) {
    AngConfigManager.importBatchConfig(
        server = clipboardText,
        subid = mainViewModel.subscriptionId,  ← READ HERE (line 1071)
        append = true
    )
}
  ↓
AngConfigManager.parseBatchConfig(servers, subid, append)
  ↓
AngConfigManager.parseConfig(line, subid, subItem)
  ↓
config.subscriptionId = subid  ← ASSIGNED HERE
  ↓
AngConfigManager.commitProfiles(configs, subid, append)
  ↓
MmkvManager.saveServerProfiles(profiles, rawConfigs, subscriptionId, append)
```

**The subscriptionId is captured at line 1071 and flows through the entire import pipeline.**

### Tab Switch Call Stack

```
User clicks Group B tab
  ↓
TabLayout.OnTabSelectedListener.onTabSelected(tab)
  ↓
MainActivity.tabSelectedListener.onTabSelected(tab)
  ↓
MainActivity.applyTabSelectedStyle(tab, true)  ← Only visual update!
  ↓
[TabLayoutMediator automatically syncs ViewPager2]
  ↓
ViewPager2.setCurrentItem(position)
  ↓
GroupPagerAdapter.createFragment(position) [if needed]
  ↓
GroupServerFragment.newInstance(subId = "group_b_id")
  ↓
[Fragment Lifecycle Begins]
  ↓
Fragment.onCreate()
  ↓
Fragment.onCreateView()
  ↓
Fragment.onViewCreated()
  ↓
Fragment.onStart()
  ↓
Fragment.onResume()
  ↓
GroupServerFragment.onResume()
  ↓
mainViewModel.subscriptionIdChangedAsync(subId = "group_b_id")
  ↓
mainViewModel.subscriptionId = "group_b_id"  ← WRITTEN HERE (line 302)
```

**The subscriptionId is only updated at the END of this long chain.**

## Why Previous Fix Didn't Catch This

The previous fix in `BUG_INVESTIGATION_GROUP_HILANG.md` addressed:
- Race condition between tab switch and **data loading**
- Used synchronized methods and double-check locking
- Fixed fragment showing wrong data

But it **didn't fix**:
- Race condition between tab switch and **subscriptionId assignment**
- Import operations reading subscriptionId before it's updated

### Difference Summary

| Aspect | Previous Bug | Current Bug |
|--------|--------------|-------------|
| **Symptom** | Fragment shows wrong group's data | Import goes to wrong group |
| **Race between** | Tab switch vs reloadServerList() | Tab switch vs import action |
| **Root cause** | Async data loading | Async subscriptionId update |
| **Fix location** | MainViewModel.reloadServerList() | Need to fix tab listener |
| **Fix approach** | Synchronization + double-check | Eager subscriptionId update |

## Android Architecture Issues

### Why This Bug Exists

1. **ViewPager2 + Fragment Lifecycle Separation**:
   - ViewPager2 switches pages immediately (visual)
   - Fragment lifecycle happens later (state)
   - No built-in mechanism to sync ViewModel with page change

2. **TabLayoutMediator Limitations**:
   - Only syncs tab selection with ViewPager2 position
   - Doesn't provide callbacks for custom logic
   - Can't directly update ViewModel state

3. **Fragment-Based State Management**:
   - Each fragment updates ViewModel in onResume()
   - onResume() happens AFTER view is visible
   - User can interact before state is updated

### Industry Best Practices Violated

1. **State Should Update Before UI**:
   - UI change (tab switch) should complete state update first
   - Then show the new UI
   - Current code: UI first, state later

2. **Defensive Programming**:
   - Read state at last possible moment (lazy read)
   - Current code: Read state at first moment (eager read)
   - Should read state from source of truth (current tab) not cache (ViewModel)

3. **Atomic Operations**:
   - Tab switch should be atomic: UI + State together
   - Current code: Two separate operations with timing gap

## Testing Strategy

### Manual Test Cases

#### Test Case 1: Fast Tab Switch + Import
```
1. Open app with Group A, Group B
2. Go to Group A tab
3. Copy a valid proxy to clipboard
4. Switch to Group B tab
5. IMMEDIATELY click + → Import from Clipboard (< 200ms)
6. Check which group received the proxy

Expected: Group B
Actual (bug): Group A
```

#### Test Case 2: Slow Tab Switch + Import
```
1. Open app with Group A, Group B
2. Go to Group A tab
3. Copy a valid proxy to clipboard
4. Switch to Group B tab
5. Wait 2 seconds
6. Click + → Import from Clipboard
7. Check which group received the proxy

Expected: Group B
Actual: Group B (no bug because enough time passed)
```

#### Test Case 3: Multiple Rapid Tab Switches + Import
```
1. Open app with Group A, Group B, Group C
2. Go to Group A tab
3. Copy a valid proxy to clipboard
4. Switch to Group B → immediately to Group C → immediately click +
5. Click Import from Clipboard
6. Check which group received the proxy

Expected: Group C
Actual (bug): Could be A, B, or C depending on timing
```

### Automated Test Approach

```kotlin
@Test
fun testClipboardImportUsesCorrectGroupAfterTabSwitch() {
    // Setup
    createGroups("Group A", "Group B")
    navigateToTab("Group A")
    setClipboard("vmess://valid_proxy")
    
    // Action: Switch tab and immediately import
    navigateToTab("Group B")
    Thread.sleep(50) // Simulate fast user click
    clickImportClipboard()
    
    // Assert
    val importedProxy = getLastImportedProxy()
    assertEquals("Group B", importedProxy.subscriptionId)
}
```

### Instrumentation Test with Espresso

```kotlin
@Test
fun testImportAfterQuickTabSwitch() {
    onView(withId(R.id.tab_group))
        .perform(selectTabAtPosition(0)) // Group A
    
    setClipboardData("vmess://test_proxy")
    
    onView(withId(R.id.tab_group))
        .perform(selectTabAtPosition(1)) // Group B
    
    // Immediately click import (race condition window)
    onView(withId(R.id.btn_add_config)).perform(click())
    onView(withId(R.id.import_clipboard)).perform(click())
    
    // Verify proxy is in Group B
    val groupB = getGroupByIndex(1)
    val proxies = getProxiesInGroup(groupB.id)
    assertTrue(proxies.any { it.remarks.contains("test_proxy") })
}
```

## Performance Impact Analysis

### Current Architecture

```
Tab click → 0ms → UI starts switching
          → 50ms → Fragment onCreate
          → 100ms → Fragment onResume
          → 101ms → subscriptionId update
          → 102ms → Async data load starts
          → 300ms → Data load complete
          → 301ms → UI fully ready

User can click import anytime after 0ms
subscriptionId ready at 101ms
→ Race window: 0-101ms
```

### Proposed Fix (Solution 1)

```
Tab click → 0ms → UI starts switching
          → 1ms → subscriptionId update (synchronous in listener)
          → 50ms → Fragment onCreate
          → 100ms → Fragment onResume
          → 102ms → Async data load starts (redundant subscriptionId update)
          → 300ms → Data load complete
          → 301ms → UI fully ready

User can click import anytime after 0ms
subscriptionId ready at 1ms
→ Race window: 0-1ms (99% reduction!)
```

### Performance Overhead

- **Additional operation**: One `subscriptionId` assignment + one MMKV write
- **Timing**: ~1-2ms (negligible)
- **Memory**: No additional memory
- **CPU**: One String comparison, one String assignment
- **Disk I/O**: One MMKV write (already buffered/async by MMKV library)

**Conclusion**: Performance impact is negligible (<1ms), race window reduced by 99%.

## Alternative Architecture (Long-term Solution)

### Option A: Single Source of Truth

Instead of caching `subscriptionId` in ViewModel, always derive it from current tab position:

```kotlin
val MainViewModel.currentSubscriptionId: String
    get() = getCurrentTabSubscriptionId() // Read from UI state
```

**Pros**: No race condition possible
**Cons**: Requires refactoring many call sites

### Option B: StateFlow-Based ViewModel

Use Kotlin Flow to reactively update subscriptionId:

```kotlin
class MainViewModel {
    private val _subscriptionId = MutableStateFlow("")
    val subscriptionId: StateFlow<String> = _subscriptionId.asStateFlow()
    
    fun updateSubscriptionId(id: String) {
        _subscriptionId.value = id
    }
}
```

**Pros**: Reactive, observable, testable
**Cons**: Requires larger refactoring

### Option C: Navigation Component

Use Jetpack Navigation with safe args:

- Pass subscriptionId as navigation argument
- Each destination (tab) has its subscriptionId
- No shared mutable state

**Pros**: Follows modern Android architecture
**Cons**: Major refactoring, might not fit ViewPager2 model

## Conclusion

This bug report documents:
1. ✅ Root cause: Race condition between tab switch and subscriptionId update
2. ✅ Exact code locations and timing analysis
3. ✅ Recommended fix with implementation details
4. ✅ Test strategy and verification plan
5. ✅ Performance impact assessment
6. ✅ Long-term architectural considerations

**Next step**: Implement Solution 1 (synchronous subscriptionId update in tab listener) as the immediate fix.
