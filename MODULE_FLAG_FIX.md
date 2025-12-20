# Module Flag Fix

## Error
```
patch.py: error: ambiguous option: --module could match --module-alterinstaller, --module-alterinstaller-sig, --module-bcr, --module-bcr-sig, --module-custota, --module-custota-sig, --module-msd, --module-msd-sig, --module-oemunlockonboot, --module-oemunlockonboot-sig
```

## Problem

The script was using generic `--module` flags, but `patch.py` requires **specific named module flags**.

## Solution

Changed from generic to specific module flags:

### Before (WRONG):
```bash
args+=("--module" ".tmp/bcr-module.zip")
args+=("--module" ".tmp/msd-module.zip")
args+=("--module" ".tmp/alterinstaller-module.zip")
args+=("--module" ".tmp/bindhosts-module.zip")
args+=("--module" ".tmp/appmanager-module.zip")
```

### After (CORRECT):
```bash
args+=("--module-bcr" ".tmp/bcr-module.zip")
args+=("--module-msd" ".tmp/msd-module.zip")
args+=("--module-alterinstaller" ".tmp/alterinstaller-module.zip")
args+=("--module-bindhosts" ".tmp/bindhosts-module.zip")
args+=("--module-appmanager" ".tmp/appmanager-module.zip")
```

## Location

File: `rooted-ota.sh`, lines 513-517

## Status

✅ **FIXED** - Now using correct named module flags that match `patch.py` expectations.

