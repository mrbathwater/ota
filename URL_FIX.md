# URL Fix for Privileged Apps Download

## Problem

The script was failing with **exit code 22 (HTTP 404 errors)** when trying to download privileged apps because the URLs were incorrect.

### Root Cause

The original implementation assumed all apps were released as **APK files**, but actually:
- **BCR, MSD, and AlterInstaller** are released as **Magisk modules** (`.zip` files)
- **bindhosts** is also a **Magisk module** (`.zip` file)
- Only **AppManager** is released as an APK

## What Was Fixed

### 1. Download URLs Corrected

**BCR:**
- ❌ OLD: `BCR-${BCR_VERSION}-release.apk`
- ✅ NEW: `BCR-${BCR_VERSION}-release.zip`

**MSD:**
- ❌ OLD: `MSD-${MSD_VERSION}-release.apk`
- ✅ NEW: `MSD-${MSD_VERSION}-release.zip`

**AlterInstaller:**
- ❌ OLD: `AlterInstaller-${ALTER_INSTALLER_VERSION}-release.apk`
- ✅ NEW: `AlterInstaller-${ALTER_INSTALLER_VERSION}-release.zip`

**bindhosts:**
- ❌ OLD: `bindhosts-v${BINDHOSTS_VERSION}.zip`
- ✅ NEW: `bindhosts.zip`

**AppManager:**
- ✅ Already correct: `AppManager_v${APPMANAGER_VERSION}.apk`

### 2. Module Creation Logic Simplified

**Before:**
- Tried to create Magisk modules from APK files for BCR, MSD, AlterInstaller
- Created module structure, copied APK, wrote module.prop, customize.sh
- Packaged everything as .zip

**After:**
- BCR, MSD, AlterInstaller are **already Magisk modules** - just rename them!
- bindhosts is **already a Magisk module** - just rename it!
- Only AppManager needs a custom module created (because it's an APK)

## Changes Made

### File: `rooted-ota.sh`

#### Function: `downloadPrivilegedApps()`
```bash
# BCR - Now downloads .zip instead of .apk
curl --fail -sL ".../BCR-${BCR_VERSION}-release.zip" > .tmp/bcr-${BCR_VERSION}.zip
curl --fail -sL ".../BCR-${BCR_VERSION}-release.zip.sig" > .tmp/bcr-${BCR_VERSION}.zip.sig

# MSD - Now downloads .zip instead of .apk
curl --fail -sL ".../MSD-${MSD_VERSION}-release.zip" > .tmp/msd-${MSD_VERSION}.zip
curl --fail -sL ".../MSD-${MSD_VERSION}-release.zip.sig" > .tmp/msd-${MSD_VERSION}.zip.sig

# AlterInstaller - Now downloads .zip instead of .apk
curl --fail -sL ".../AlterInstaller-${ALTER_INSTALLER_VERSION}-release.zip" > .tmp/alterinstaller-${ALTER_INSTALLER_VERSION}.zip
curl --fail -sL ".../AlterInstaller-${ALTER_INSTALLER_VERSION}-release.zip.sig" > .tmp/alterinstaller-${ALTER_INSTALLER_VERSION}.zip.sig

# bindhosts - Fixed filename (no version in filename)
curl --fail -sL ".../bindhosts.zip" > .tmp/bindhosts-${BINDHOSTS_VERSION}.zip

# AppManager - Unchanged (already correct)
curl --fail -sL ".../AppManager_v${APPMANAGER_VERSION}.apk" > .tmp/appmanager-${APPMANAGER_VERSION}.apk
```

#### Function: `createPrivilegedAppModules()`
```bash
# BCR, MSD, AlterInstaller - Just rename the downloaded modules
cp ".tmp/bcr-${BCR_VERSION}.zip" ".tmp/bcr-module.zip"
cp ".tmp/msd-${MSD_VERSION}.zip" ".tmp/msd-module.zip"
cp ".tmp/alterinstaller-${ALTER_INSTALLER_VERSION}.zip" ".tmp/alterinstaller-module.zip"

# bindhosts - Just rename
cp ".tmp/bindhosts-${BINDHOSTS_VERSION}.zip" ".tmp/bindhosts-module.zip"

# AppManager - Create custom module (only this one needs it)
# ... (module creation code for AppManager only)
```

## Benefits of This Fix

1. **Correct URLs** - No more 404 errors
2. **Simpler Code** - No need to create module structures for pre-built modules
3. **Faster Execution** - Less processing (no zip creation for 4 out of 5 apps)
4. **Maintains Signatures** - Pre-built modules keep their original signatures from authors
5. **Better Maintainability** - Uses official modules as released by authors

## Testing

To verify the fix:

```bash
# Test script syntax
bash -n rooted-ota.sh

# Test download URLs (dry run - don't actually download)
for url in \
  "https://github.com/chenxiaolong/BCR/releases/download/v1.87/BCR-1.87-release.zip" \
  "https://github.com/chenxiaolong/MSD/releases/download/v1.20/MSD-1.20-release.zip" \
  "https://github.com/chenxiaolong/AlterInstaller/releases/download/v2.3/AlterInstaller-2.3-release.zip" \
  "https://github.com/bindhosts/bindhosts/releases/download/v2.1.0/bindhosts.zip" \
  "https://github.com/MuntashirAkon/AppManager/releases/download/v4.0.5/AppManager_v4.0.5.apk"
do
  echo "Testing: $url"
  curl -I --fail -sL "$url" | grep "HTTP.*200"
done
```

All URLs should return HTTP 200 OK.

## Impact

- ✅ Downloads will now succeed
- ✅ Module creation will work correctly
- ✅ Both rootless and magisk builds will include all 5 privileged apps
- ✅ All apps will have proper privileges and permissions
- ✅ No more exit code 22 errors

## Next Steps

1. Commit these changes to your repository
2. Push to GitHub (after resolving the `workflow` scope issue)
3. Run the GitHub Actions workflow
4. Verify all 5 apps are included in the built OTAs

