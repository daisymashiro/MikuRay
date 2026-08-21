# Bug Report: Clipboard Import Goes to Wrong Group

## Bug Description
When user imports proxy from clipboard to Group B, the proxy gets pasted to Group A instead (the first/earliest group). This happens because of a **race condition** between tab switching and clipboard import.

## Root Cause Analysis

### The Problem Flow

1. **User is on Group A (first group)**
2. **User switches to Group B tab**
3. **User immediately clicks "Import from Clipboard"**
4. **The import uses Group A's subscriptionId instead of Group B**

### Why This Happens

#### 1. Tab Selection Does NOT Update `mainViewModel.subscriptionId` Immediately

When user clicks a tab in MainActivity:
- `TabLayoutMediator` syncs the tab with ViewPager2
- ViewPager2 switches to the new fragment
- The new fragment's `onResume()` is called
- **Only in `onResume()` does the subscriptionId get updated**

**File: `GroupServerFragment.kt:176-178`**
```kotlin
override fun onResume() {
    super.onResume()
    mainViewModel.subscriptionIdChangedAsync(subId)  // ← Updates subscriptionId ASYNC
    // ...
}
```

**File: `MainViewModel.kt:299-309`**
```kotlin
fun subscriptionIdChangedAsync(id: String) {
    if (subscriptionId != id) {
        LogUtil.d(AppConfig.TAG, "Subscription ID changed from '$subscriptionId' to '$id'")
        subscriptionId = id
        MmkvManager.encodeSettings(AppConfig.CACHE_SUBSCRIPTION_ID, subscriptionId)
    }
    val targetSubId = subscriptionId
    viewModelScope.launch(Dispatchers.IO) {  // ← Launches async, not blocking
        reloadServerListForSubscription(targetSubId)
    }
}
```

#### 2. The Race Condition Window

There's a **timing window** between:
- User switches tab → Fragment.onResume() called → subscriptionId update starts
- User clicks "Import from Clipboard" button

If the user clicks "Import from Clipboard" **before** `subscriptionIdChangedAsync()` completes the assignment on line 302, the old subscriptionId is used.

#### 3. Import Uses Stale subscriptionId

**File: `MainActivity.kt:1055-1063`**
```kotlin
private fun importClipboard(): Boolean {
    return try {
        Utils.getClipboard(this).let { importBatchConfig(it) }
        true
    } catch (e: Exception) {
        LogUtil.e(AppConfig.TAG, "Failed to import config from clipboard", e)
        false
    }
}
```

**File: `MainActivity.kt:1065-1103`**
```kotlin
private fun importBatchConfig(server: String?) {
    if (server.isNullOrEmpty()) return

    showLoading()
    lifecycleScope.launch(Dispatchers.IO) {
        try {
            val (count, countSub) = AngConfigManager.importBatchConfig(
                server, 
                mainViewModel.subscriptionId,  // ← Uses current subscriptionId value
                true
            )
            // ...
```

**The `mainViewModel.subscriptionId` is read synchronously here**, but it may still contain the OLD group's ID if the fragment's `onResume()` hasn't finished updating it yet.

#### 4. Why It Defaults to "Group A" (First Group)

When the app starts:
- `mainViewModel.subscriptionId` is initialized from cache or defaults to empty ""
- If user was last on Group A, subscriptionId = Group A's ID
- The race condition means this stale value is used for import

### The Same Bug Affects Other Import Methods

The same race condition affects:
- **Import QR Code** (`MainActivity.kt:1002-1009`)
- **Import from File** (`MainActivity.kt:1402-1419`)
- **Manual Config Creation** (`MainActivity.kt:969-1000`)

All of these read `mainViewModel.subscriptionId` synchronously and pass it to import functions.

## Evidence from Code

### 1. Tab Selection Flow (No Direct subscriptionId Update)

**File: `MainActivity.kt:130-140`**
```kotlin
private val tabSelectedListener = object : TabLayout.OnTabSelectedListener {
    override fun onTabSelected(tab: TabLayout.Tab) {
        applyTabSelectedStyle(tab, true, tab.position, binding.tabGroup.tabCount)
        // ← No subscriptionId update here!
    }

    override fun onTabUnselected(tab: TabLayout.Tab) {
        applyTabSelectedStyle(tab, false, tab.position, binding.tabGroup.tabCount)
    }

    override fun onTabReselected(tab: TabLayout.Tab) {}
}
```

TabLayoutMediator handles the ViewPager2 page switching, but **never explicitly updates `mainViewModel.subscriptionId`**. It relies on the fragment's `onResume()` to do this.

### 2. Fragment Lifecycle Dependency

The subscriptionId update is **lifecycle-dependent**:
- Fragment must be created
- Fragment must enter `onResume()` state
- `subscriptionIdChangedAsync()` must be called
- The assignment `subscriptionId = id` must execute

This creates a race window of **hundreds of milliseconds** where the user can trigger an import with the wrong subscriptionId.

### 3. No Synchronization Mechanism

There's **no locking or synchronization** between:
- Tab switching (ViewPager2 + Fragment lifecycle)
- Import actions (triggered from bottom sheet menu)

Both can happen concurrently, and import doesn't wait for the tab switch to complete.

## Impact

### Primary Impact
- **Data Corruption**: Proxies are added to wrong subscription group
- **User Confusion**: User expects proxy in Group B, but it appears in Group A
- **Workflow Disruption**: User must manually move proxies to correct group

### Secondary Impact
- **Lost Context**: If user deletes Group A thinking proxies are in Group B, they lose the imported proxies
- **Reconnection Issues**: The bug report mentions "reconnect or proxy confusion" - this could be related to proxies being in unexpected groups

## Reproduction Steps

1. Open MikuRay app
2. Ensure you have Group A and Group B created
3. Navigate to Group A tab
4. Switch to Group B tab
5. **IMMEDIATELY** click "+" button → "Import from Clipboard" (with valid proxy in clipboard)
6. Observe: Proxy appears in Group A instead of Group B

**Note**: The faster you click after switching tabs, the higher chance to reproduce. On slower devices, the race window is larger.

## Proposed Fix

### Solution 1: Synchronous subscriptionId Update in Tab Listener (Recommended)

Update `mainViewModel.subscriptionId` **immediately** when tab is selected, before the fragment lifecycle runs.

**File: `MainActivity.kt:130-140`**
```kotlin
private val tabSelectedListener = object : TabLayout.OnTabSelectedListener {
    override fun onTabSelected(tab: TabLayout.Tab) {
        applyTabSelectedStyle(tab, true, tab.position, binding.tabGroup.tabCount)
        
        // Get the subscription ID from the selected tab position
        val selectedSubId = groupPagerAdapter.groups.getOrNull(tab.position)?.id.orEmpty()
        
        // Update subscriptionId IMMEDIATELY (synchronous, not async)
        if (mainViewModel.subscriptionId != selectedSubId) {
            mainViewModel.subscriptionId = selectedSubId
            MmkvManager.encodeSettings(AppConfig.CACHE_SUBSCRIPTION_ID, selectedSubId)
            LogUtil.d(AppConfig.TAG, "Tab selected: updated subscriptionId to '$selectedSubId'")
        }
    }

    override fun onTabUnselected(tab: TabLayout.Tab) {
        applyTabSelectedStyle(tab, false, tab.position, binding.tabGroup.tabCount)
    }

    override fun onTabReselected(tab: TabLayout.Tab) {}
}
```

**Pros:**
- Fixes the race condition completely
- subscriptionId is updated **before** user can click any import button
- Simple and centralized fix
- No performance impact

**Cons:**
- Duplicates logic (also updated in fragment onResume)
- Need to ensure both updates stay in sync

### Solution 2: Capture subscriptionId from Current Tab (Alternative)

Instead of reading `mainViewModel.subscriptionId`, capture it from the currently visible fragment/tab.

**File: `MainActivity.kt:1065-1071`**
```kotlin
private fun importBatchConfig(server: String?) {
    if (server.isNullOrEmpty()) return

    // Capture subscriptionId from current tab position
    val currentTabPosition = binding.viewPager.currentItem
    val targetSubId = groupPagerAdapter.groups.getOrNull(currentTabPosition)?.id.orEmpty()

    showLoading()
    lifecycleScope.launch(Dispatchers.IO) {
        try {
            val (count, countSub) = AngConfigManager.importBatchConfig(
                server, 
                targetSubId,  // ← Use captured subscriptionId instead
                true
            )
```

**Pros:**
- Guarantees correct subscriptionId (from visible tab)
- Doesn't depend on ViewModel state

**Cons:**
- Need to update multiple import functions
- Requires checking if `groupPagerAdapter.groups` is populated

### Solution 3: Block Import Until subscriptionId Sync Complete (Heavy)

Add a flag that blocks import operations until fragment onResume completes.

**Not recommended** due to complexity and poor UX (button would be disabled temporarily).

## Recommended Fix: Solution 1

Implement **Solution 1** because:
1. Fixes the root cause (delayed subscriptionId update)
2. Simple one-location fix
3. No impact on existing functionality
4. Makes subscriptionId update **eager** instead of lazy

## Files Affected

### Primary Fix Location
- **V2rayNG/app/src/main/java/com/v2ray/ang/ui/main/MainActivity.kt:130-140**
  - Update `tabSelectedListener.onTabSelected()` to set subscriptionId immediately

### Verification Points
- **V2rayNG/app/src/main/java/com/v2ray/ang/ui/main/GroupServerFragment.kt:178**
  - Keep the `subscriptionIdChangedAsync()` call for async data loading
  - But subscriptionId itself will already be set by tab listener

### Testing Required
- Import from Clipboard after tab switch
- Import QR Code after tab switch  
- Import from File after tab switch
- Manual config creation after tab switch
- Verify no duplicate data loading issues

## Related Issues

This bug is **different** from the previous tab switching bug fixed in `BUG_INVESTIGATION_GROUP_HILANG.md`:
- Previous bug: Fragment displayed wrong data due to async race between tab switch and data load
- **This bug**: Import uses wrong subscriptionId due to async race between tab switch and subscriptionId update

Both are timing/race condition bugs, but affect different operations.

## Priority

**HIGH** - This causes data corruption and breaks a core user workflow (importing proxies to specific groups).
