# GitHub Actions Workflow Fix Report

## Date
2026-08-21

## Issue
Both `build-release.yml` and `build-debug.yml` workflows were missing critical native library build steps, causing build failures due to missing `libhev-socks5-tunnel.so` and `libv2ray.aar` dependencies.

## Root Cause
The workflows were incomplete compared to the working `build.yml` workflow. They were missing:
1. Native library compilation step for hev-socks5-tunnel
2. AndroidLibXrayLite tag fetching and libv2ray.aar download
3. Proper NDK_HOME environment variable setup
4. Library caching mechanism

## Files Modified

### 1. `.github/workflows/build-release.yml`
**Changes Made:**
- Added `NDK_HOME` environment variable export after NDK installation (Step 5)
- Added libhevtun cache restoration step (Step 6)
- Added libhevtun native library build step using `compile-hevtun.sh` (Step 7)
- Added libhevtun cache save step for future builds (Step 8)
- Added libhevtun copy to V2rayNG/app/libs (Step 9)
- Added AndroidLibXrayLite tag fetching step (Step 10)
- Added libv2ray.aar download from 2dust/AndroidLibXrayLite releases (Step 11)
- Renumbered all subsequent steps (12-21) to maintain proper sequence

**Key Code Additions:**
```yaml
# Step 5: Export NDK_HOME
echo "NDK_HOME=$ANDROID_HOME/ndk/29.0.14206865" >> $GITHUB_ENV

# Step 6-8: Cache-aware libhevtun build
- name: Restore cached libhevtun
  id: cache-libhevtun-restore
  uses: actions/cache/restore@v6
  with:
    path: ${{ github.workspace }}/libs
    key: libhevtun-${{ runner.os }}-${{ env.NDK_HOME }}-${{ hashFiles('.git/modules/hev-socks5-tunnel/HEAD') }}-${{ hashFiles('compile-hevtun.sh') }}

- name: Build libhevtun
  if: steps.cache-libhevtun-restore.outputs.cache-hit != 'true'
  run: bash compile-hevtun.sh

- name: Save libhevtun
  if: steps.cache-libhevtun-restore.outputs.cache-hit != 'true'
  uses: actions/cache/save@v6

# Step 9: Copy to app directory
- name: Copy libhevtun
  run: cp -r ${{ github.workspace }}/libs ${{ github.workspace }}/V2rayNG/app

# Step 10-11: Fetch and download libv2ray
- name: Fetch AndroidLibXrayLite tag
  run: |
    pushd AndroidLibXrayLite
    CURRENT_TAG=$(git describe --tags --abbrev=0)
    echo "CURRENT_TAG=$CURRENT_TAG" >> $GITHUB_ENV
    popd

- name: Download libv2ray
  uses: robinraju/release-downloader@v1.13
  with:
    repository: '2dust/AndroidLibXrayLite'
    tag: ${{ env.CURRENT_TAG }}
    fileName: 'libv2ray.aar'
    out-file-path: V2rayNG/app/libs/
```

### 2. `.github/workflows/build-debug.yml`
**Changes Made:**
- Added `NDK_HOME` environment variable export after NDK installation (Step 5)
- Added libhevtun cache restoration step (Step 6)
- Added libhevtun native library build step using `compile-hevtun.sh` (Step 7)
- Added libhevtun cache save step for future builds (Step 8)
- Added libhevtun copy to V2rayNG/app/libs (Step 9)
- Added AndroidLibXrayLite tag fetching step (Step 10)
- Added libv2ray.aar download from 2dust/AndroidLibXrayLite releases (Step 11)
- Renumbered all subsequent steps (12-16) to maintain proper sequence

**Identical Structure to Release Workflow:**
Both workflows now follow the same native library build pattern as the working `build.yml`.

### 3. `compile-hevtun.sh`
**Changes Made:**
- Made the script executable: `chmod +x compile-hevtun.sh`

## Technical Details

### Native Library Build Process
1. **hev-socks5-tunnel compilation:**
   - Uses Android NDK 29.0.14206865
   - Builds for 4 ABIs: armeabi-v7a, arm64-v8a, x86, x86_64
   - Creates two artifacts:
     - `libhev-socks5-tunnel.so` (JNI shared library for VpnService)
     - `libhevsockstun.so` (standalone executable for root mode)
   - Output directory: `libs/<abi>/`

2. **libv2ray.aar download:**
   - Fetches latest tag from AndroidLibXrayLite submodule
   - Downloads matching release from 2dust/AndroidLibXrayLite repository
   - Places in `V2rayNG/app/libs/`

### Caching Strategy
- Cache key includes: OS, NDK version, hev-socks5-tunnel commit hash, and script hash
- Skips compilation when cache hit occurs
- Significantly speeds up subsequent builds

## Verification Steps
The workflows should now:
1. ✅ Checkout code with submodules
2. ✅ Setup Java JDK 21
3. ✅ Setup Android SDK
4. ✅ Install NDK and set NDK_HOME
5. ✅ Build or restore cached libhevtun
6. ✅ Download libv2ray.aar
7. ✅ Build APK successfully
8. ✅ Upload artifacts

## Reference Workflow
The implementation matches the working `build.yml` workflow (lines 46-90) which has been successfully building APKs.

## Expected Outcomes
- **build-release.yml:** Will now successfully build and sign release APKs
- **build-debug.yml:** Will now successfully build debug APKs
- Both workflows will use caching to speed up builds
- No more missing native library errors

## Files Involved
- `.github/workflows/build-release.yml` (modified, 232 lines)
- `.github/workflows/build-debug.yml` (modified, 142 lines)
- `compile-hevtun.sh` (made executable)
- `hev-socks5-tunnel/` (submodule, source for native libraries)
- `AndroidLibXrayLite/` (submodule, reference for libv2ray version)

## Testing Recommendations
1. Push changes to trigger workflows
2. Verify libhevtun cache is created on first run
3. Verify cache is reused on subsequent runs
4. Confirm APK artifacts contain native libraries
5. Test APK installation on target devices

## Notes
- The workflows now match the proven working pattern from `build.yml`
- Cache invalidation occurs when NDK version, script, or submodule changes
- Native libraries are built before Gradle build process
- Both JNI library and standalone executable are included for different run modes
