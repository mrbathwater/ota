# Privileged Apps Integration

This document describes the privileged apps that have been integrated into the GrapheneOS OTA build process.

## Overview

Five privileged apps have been integrated into both rootless and magisk OTA builds:

1. **BCR (Basic Call Recorder)** v1.87
2. **MSD (Material Storage Disk)** v1.20
3. **AlterInstaller** v2.3
4. **bindhosts** v2.1.0
5. **AppManager** v4.0.5

## Implementation Details

### Architecture

All apps are installed as privileged system apps using avbroot's module system:
- **Rootless builds**: Apps are extracted directly into `/system/priv-app/` during OTA patching
- **Magisk builds**: Apps are installed as Magisk modules with privileged app capabilities

### Version Management

App versions are pinned in `rooted-ota.sh` with renovate bot integration for automatic updates:

```bash
BCR_VERSION=1.87
MSD_VERSION=1.20
ALTER_INSTALLER_VERSION=2.3
BINDHOSTS_VERSION=2.1.0
APPMANAGER_VERSION=4.0.5
```

### Download and Verification

Apps from chenxiaolong (BCR, MSD, AlterInstaller) are verified using SSH signature verification with the author's public key.

### Module Structure

Each app is packaged as a Magisk module with:
- `module.prop` - Module metadata
- `system/priv-app/AppName/AppName.apk` - The app APK
- `customize.sh` - SELinux context setup script
- For AppManager: `system/etc/permissions/privapp-permissions-appmanager.xml` - Privileged permissions allowlist

## App Descriptions

### BCR (Basic Call Recorder)
- **Author**: chenxiaolong
- **Purpose**: Records phone calls with support for various audio formats
- **Repository**: https://github.com/chenxiaolong/BCR

### MSD (Material Storage Disk)
- **Author**: chenxiaolong
- **Purpose**: Advanced storage management for Android
- **Repository**: https://github.com/chenxiaolong/MSD

### AlterInstaller
- **Author**: chenxiaolong
- **Purpose**: Alternative app installer for Android
- **Repository**: https://github.com/chenxiaolong/AlterInstaller

### bindhosts
- **Author**: bindhosts team
- **Purpose**: Hosts file management with module and app
- **Repository**: https://github.com/bindhosts/bindhosts

### AppManager
- **Author**: MuntashirAkon
- **Purpose**: Full-featured package manager and viewer for Android
- **Repository**: https://github.com/MuntashirAkon/AppManager
- **Special Requirements**: Requires 30+ privileged permissions for full functionality

## AppManager Privileged Permissions

AppManager requires extensive privileged permissions to function properly. These are granted via `/system/etc/permissions/privapp-permissions-appmanager.xml`:

### Core App Management
- `INSTALL_PACKAGES` - Install/uninstall apps
- `DELETE_PACKAGES` - Remove apps
- `CLEAR_APP_USER_DATA` - Clear app data
- `CLEAR_APP_CACHE` - Clear app cache
- `FORCE_STOP_PACKAGES` - Stop apps
- `CHANGE_COMPONENT_ENABLED_STATE` - Enable/disable components
- `SUSPEND_APPS` - Suspend apps

### Permission Management
- `GRANT_RUNTIME_PERMISSIONS` - Grant permissions
- `REVOKE_RUNTIME_PERMISSIONS` - Revoke permissions
- `ADJUST_RUNTIME_PERMISSIONS_POLICY` - Permission policy

### AppOps Management
- `GET_APP_OPS_STATS` - Read app ops
- `MANAGE_APP_OPS_MODES` - Modify app ops
- `UPDATE_APP_OPS_STATS` - Update app ops

### Multi-User Support
- `INTERACT_ACROSS_USERS` - Multi-user support
- `INTERACT_ACROSS_USERS_FULL` - Full multi-user support
- `MANAGE_USERS` - User management

### System Control
- `KILL_UID` - Kill processes
- `REAL_GET_TASKS` - Get running tasks
- `START_ANY_ACTIVITY` - Launch any activity
- `DUMP` - Dump system state
- `WRITE_SECURE_SETTINGS` - Modify secure settings
- `DEVICE_POWER` - Power management
- `INJECT_EVENTS` - Input injection

### Network & Sensors
- `MANAGE_NETWORK_POLICY` - Network policy control
- `MANAGE_SENSORS` - Sensor management
- `NETWORK_SETTINGS` - Network settings

### System Information
- `READ_LOGS` - Read system logs
- `BACKUP` - Backup functionality

### Advanced Features
- `INSTALL_EXISTING_PACKAGES` - Install existing packages
- `UPDATE_DOMAIN_VERIFICATION_USER_SELECTION` - Domain verification
- `CHANGE_OVERLAY_PACKAGES` - Overlay management
- `DELETE_CACHE_FILES` - Cache management
- `INTERNAL_DELETE_CACHE_FILES` - Internal cache deletion
- `MANAGE_NOTIFICATION_LISTENERS` - Notification listener management

## SELinux Configuration

All apps have proper SELinux contexts set via `customize.sh`:
```bash
chcon -R u:object_r:system_file:s0 "$MODPATH/system/priv-app/AppName"
```

For AppManager, permissions directory also gets proper context:
```bash
chcon -R u:object_r:system_file:s0 "$MODPATH/system/etc/permissions"
```

## Build Process Integration

The apps are integrated into the build process via three main functions in `rooted-ota.sh`:

1. **`downloadPrivilegedApps()`** - Downloads all APKs and modules with signature verification
2. **`createPrivilegedAppModules()`** - Creates Magisk module structures for each app
3. **`patchOTAs()`** - Integrates modules into the OTA patching process using avbroot's `--module` flag

## Skipping Modules

To skip installation of these privileged apps, set the environment variable:
```bash
SKIP_MODULES=true
```

This will skip both the existing modules (custota, oemunlockonboot) and the new privileged apps.

## GitHub Actions Compatibility

The implementation is fully compatible with the existing GitHub Actions workflows. No workflow changes are required as the workflows simply call `rooted-ota.sh`.

## Testing

To test the implementation:

1. **Syntax Check**: `bash -n rooted-ota.sh`
2. **Function Verification**: Source the script and verify functions are defined
3. **Build Test**: Run a full OTA build for your device
4. **Install Test**: Flash the OTA and verify all apps are installed in `/system/priv-app/`
5. **Permission Test**: For AppManager, verify all privileged permissions are granted

## Maintenance

Version updates are managed via renovate bot comments in `rooted-ota.sh`. The bot will automatically create PRs when new versions are released.

## Troubleshooting

### Apps not appearing after OTA
- Verify `SKIP_MODULES` is not set to `true`
- Check that modules were created in `.tmp/` directory
- Verify avbroot successfully processed the modules

### AppManager missing permissions
- Check that `privapp-permissions-appmanager.xml` is present in `/system/etc/permissions/`
- Verify SELinux contexts are correct
- Check system logs for permission denial messages

### Signature verification failures
- Ensure chenxiaolong's public key is correctly defined in `CHENXIAOLONG_PK`
- Verify `.sig` files are being downloaded correctly
- Check network connectivity during download

## References

- [avbroot documentation](https://github.com/chenxiaolong/avbroot)
- [Android Privileged Permission Allowlist](https://source.android.com/docs/core/permissions/perms-allowlist)
- [Magisk Module Format](https://topjohnwu.github.io/Magisk/guides.html)

