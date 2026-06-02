# pixeneos + Hail Rootless OTA — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the rootless OTA flavor (bluejay, tangorpro, tokay) with a pixeneos-built OTA carrying the full pixeneos module set plus Hail as a privileged system app with a new no-root combined force-stop+freeze mode, signed with the user's existing keys and published in place to `mrbathwater/ota` releases + `gh-pages/rootless/`.

**Architecture:** Four repos. `mrbathwater/Hail` gains a priv-app combined mode + a release workflow that publishes a signed APK. `mrbathwater/my-avbroot-setup` (new fork) gains a `hail` module so `patch.py` can inject Hail as a priv-app. `mrbathwater/pixeneos` (new fork) is the rootless build engine: it downloads the Hail APK, assembles a priv-app module zip, injects it, signs with the user's keys, and emits a device JSON whose `location` points at `mrbathwater/ota`. `mrbathwater/ota`'s `release-single.yaml` restricts the existing schnatterer job to magisk-only and adds a rootless job that runs the pixeneos fork and publishes the artifacts into this repo. The magisk/root flavor is untouched.

**Tech Stack:** Bash, Python 3 (avbroot / my-avbroot-setup), Kotlin/Gradle (Hail), GitHub Actions, GrapheneOS avbroot + Custota toolchain.

**Spec:** `docs/superpowers/specs/2026-06-02-rootless-pixeneos-hail-design.md`

**Fixed names used throughout this plan (do not vary them):**
- Hail release tag: `v1.10.0-priv1`; Hail APK asset name: `Hail.apk`
- my-avbroot-setup fork pin branch: `hail-pinned`
- Local clones live as siblings of this repo under `/home/andy/Desktop/ota/`:
  `Hail/`, `my-avbroot-setup/`, `pixeneos/` (this repo is `ota/`)

---

## Phase 0: Create forks and local clones

### Task 0.1: Fork and clone the three external repos

**Files:** none (repo/clone setup)

- [ ] **Step 1: Confirm gh auth (must be `mrbathwater`)**

Run: `gh auth status && gh api user --jq .login`
Expected: shows `Logged in to github.com account mrbathwater` and prints `mrbathwater`.

- [ ] **Step 2: Fork pixeneos and my-avbroot-setup**

```bash
gh repo fork pixincreate/pixeneos --clone=false --fork-name pixeneos
gh repo fork chenxiaolong/my-avbroot-setup --clone=false --fork-name my-avbroot-setup
```
Expected: each prints `Created fork mrbathwater/<name>` (or "already exists").

- [ ] **Step 3: Clone all three as siblings of this repo**

```bash
cd /home/andy/Desktop/ota
git clone git@github.com:mrbathwater/Hail.git Hail
git clone git@github.com:mrbathwater/pixeneos.git pixeneos
git clone git@github.com:mrbathwater/my-avbroot-setup.git my-avbroot-setup
ls -d Hail pixeneos my-avbroot-setup ota
```
Expected: all four directories listed.

---

## Phase 1: Hail fork — priv-app combined mode + release workflow

Working directory for this phase: `/home/andy/Desktop/ota/Hail` (branch `master`).

### Task 1.1: Add the `MODE_PRIVAPP_STOP_DISABLE` constant

**Files:**
- Modify: `app/src/main/kotlin/com/aistra/hail/app/HailData.kt`

- [ ] **Step 1: Add the constant after `MODE_PRIVAPP_DISABLE`**

Find:
```kotlin
    const val MODE_PRIVAPP_STOP = PRIVAPP + STOP
    const val MODE_PRIVAPP_DISABLE = PRIVAPP + DISABLE
```
Replace with:
```kotlin
    const val MODE_PRIVAPP_STOP = PRIVAPP + STOP
    const val MODE_PRIVAPP_DISABLE = PRIVAPP + DISABLE
    const val MODE_PRIVAPP_STOP_DISABLE = PRIVAPP + STOP_DISABLE
```

- [ ] **Step 2: Add it to `WORKING_MODE_VALUES`**

Find:
```kotlin
        MODE_PRIVAPP_STOP,
        MODE_PRIVAPP_DISABLE
    )
```
Replace with:
```kotlin
        MODE_PRIVAPP_STOP,
        MODE_PRIVAPP_DISABLE,
        MODE_PRIVAPP_STOP_DISABLE
    )
```

### Task 1.2: Wire the combined action in `AppManager`

**Files:**
- Modify: `app/src/main/kotlin/com/aistra/hail/app/AppManager.kt`

- [ ] **Step 1: Add the branch in `setAppFrozen()` after `MODE_PRIVAPP_DISABLE`**

Find:
```kotlin
            HailData.MODE_PRIVAPP_STOP -> !frozen || HPackages.forceStopApp(packageName)
            HailData.MODE_PRIVAPP_DISABLE -> HPackages.setAppDisabled(packageName, frozen)
            else -> false
```
Replace with:
```kotlin
            HailData.MODE_PRIVAPP_STOP -> !frozen || HPackages.forceStopApp(packageName)
            HailData.MODE_PRIVAPP_DISABLE -> HPackages.setAppDisabled(packageName, frozen)
            HailData.MODE_PRIVAPP_STOP_DISABLE -> if (frozen) {
                HPackages.forceStopApp(packageName) && HPackages.setAppDisabled(packageName, true)
            } else {
                HPackages.setAppDisabled(packageName, false)
            }
            else -> false
```

Note: `isAppFrozen()` already handles this mode — its `endsWith(HailData.STOP_DISABLE)` branch returns `HPackages.isAppDisabled(packageName)`, which is correct for the priv-app combined mode. No change needed there.

### Task 1.3: Add the UI label and picker entry

**Files:**
- Modify: `app/src/main/res/values/strings.xml`
- Modify: `app/src/main/res/values/arrays.xml`

- [ ] **Step 1: Add the string after `mode_privapp_disable`**

Find:
```xml
    <string name="mode_privapp_disable">System App - Disable</string>
```
Replace with:
```xml
    <string name="mode_privapp_disable">System App - Disable</string>
    <string name="mode_privapp_stop_disable">System App - Force Stop + Disable</string>
```

- [ ] **Step 2: Add the entry to `working_mode_entries`**

Find:
```xml
        <item>@string/mode_privapp_stop</item>
        <item>@string/mode_privapp_disable</item>
    </string-array>
```
Replace with:
```xml
        <item>@string/mode_privapp_stop</item>
        <item>@string/mode_privapp_disable</item>
        <item>@string/mode_privapp_stop_disable</item>
    </string-array>
```

(The array order must match `WORKING_MODE_VALUES`; both now end with `…privapp_stop, …privapp_disable, …privapp_stop_disable`.)

### Task 1.4: Verify the app compiles

**Files:** none

- [ ] **Step 1: Assemble release to confirm the Kotlin + resources compile**

Run: `cd /home/andy/Desktop/ota/Hail && ./gradlew :app:assembleRelease`
Expected: `BUILD SUCCESSFUL`. The release build has no `applicationIdSuffix`, so the package is `com.aistra.hail`. (If gradle complains about `signing.properties`, that's handled in Task 1.5 — this step only proves compilation; a `BUILD SUCCESSFUL` with the debug-fallback signature is fine here.)

- [ ] **Step 2: Commit**

```bash
cd /home/andy/Desktop/ota/Hail
git add app/src/main/kotlin/com/aistra/hail/app/HailData.kt \
        app/src/main/kotlin/com/aistra/hail/app/AppManager.kt \
        app/src/main/res/values/strings.xml \
        app/src/main/res/values/arrays.xml
git commit -m "Add privileged-system-app combined force-stop + disable mode"
git push origin master
```

### Task 1.5: Add a release workflow that publishes a signed APK

**Files:**
- Create: `.github/workflows/release-apk.yml`

- [ ] **Step 1: Create the workflow**

```yaml
name: Release APK

on:
  workflow_dispatch:
  push:
    tags:
      - 'v*'

permissions:
  contents: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Java JDK
        uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '21'

      - name: Setup Gradle
        uses: gradle/actions/setup-gradle@v4

      - name: Generate an ephemeral signing keystore
        run: |
          keytool -genkeypair -v \
            -keystore keystore.jks \
            -storepass hailota -keypass hailota \
            -alias hail -keyalg RSA -keysize 2048 -validity 10000 \
            -dname "CN=Hail OTA, OU=ota, O=mrbathwater, C=US"
          {
            echo "storeFile=../keystore.jks"
            echo "storePassword=hailota"
            echo "keyAlias=hail"
            echo "keyPassword=hailota"
          } > signing.properties

      - name: Build release APK
        run: ./gradlew :app:assembleRelease

      - name: Rename APK to stable asset name
        run: |
          src=$(ls app/build/outputs/apk/release/*.apk | head -n1)
          cp "$src" Hail.apk
          ls -l Hail.apk

      - name: Publish release
        uses: softprops/action-gh-release@v2
        with:
          tag_name: ${{ github.ref_type == 'tag' && github.ref_name || 'v1.10.0-priv1' }}
          name: ${{ github.ref_type == 'tag' && github.ref_name || 'v1.10.0-priv1' }}
          files: Hail.apk
```

- [ ] **Step 2: Commit and push**

```bash
cd /home/andy/Desktop/ota/Hail
git add .github/workflows/release-apk.yml
git commit -m "Add release-apk workflow that publishes a signed Hail.apk"
git push origin master
```

- [ ] **Step 3: Create the release tag and run the workflow**

```bash
cd /home/andy/Desktop/ota/Hail
git tag v1.10.0-priv1
git push origin v1.10.0-priv1
gh run watch "$(gh run list --workflow=release-apk.yml --limit 1 --json databaseId --jq '.[0].databaseId')" || true
```
Expected: the workflow succeeds.

- [ ] **Step 4: Verify the APK asset exists and has the correct package**

```bash
cd /home/andy/Desktop/ota/Hail
gh release view v1.10.0-priv1 --json assets --jq '.assets[].name'
gh release download v1.10.0-priv1 -p Hail.apk -O /tmp/Hail.apk --clobber
# Confirm the package id is com.aistra.hail (NOT .debug):
"$ANDROID_HOME"/build-tools/*/aapt2 dump badging /tmp/Hail.apk 2>/dev/null | grep "package: name" || \
  unzip -p /tmp/Hail.apk AndroidManifest.xml | strings | grep -m1 "com.aistra.hail"
```
Expected: asset `Hail.apk` is listed; package name is exactly `com.aistra.hail`.

---

## Phase 2: my-avbroot-setup fork — the `hail` module

Working directory: `/home/andy/Desktop/ota/my-avbroot-setup`.

### Task 2.1: Create the pinned branch from the commit pixeneos uses

**Files:** none (branch setup)

- [ ] **Step 1: Branch from the pinned upstream commit**

```bash
cd /home/andy/Desktop/ota/my-avbroot-setup
git remote add upstream https://github.com/chenxiaolong/my-avbroot-setup.git 2>/dev/null || true
git fetch upstream
git checkout -b hail-pinned ae7c68c8e90a74991cc91bb0caa8c7e68b180846
```
Expected: `Switched to a new branch 'hail-pinned'`.

### Task 2.2: Add the Hail module

**Files:**
- Create: `lib/modules/hail.py`
- Modify: `lib/modules/__init__.py`

- [ ] **Step 1: Create `lib/modules/hail.py`**

```python
# SPDX-License-Identifier: GPL-3.0-only

from collections.abc import Iterable
import logging
from pathlib import Path
from typing import override
import zipfile

from lib import modules
from lib.filesystem import CpioFs, ExtFs
from lib.modules import Module, ModuleRequirements


logger = logging.getLogger(__name__)


class HailModule(Module):
    def __init__(self, zip: Path, sig: Path) -> None:
        super().__init__()

        # Hail is built from our own fork in CI and is not signed with
        # chenxiaolong's key, so signature verification is intentionally
        # skipped. The `sig` argument is accepted for interface compatibility
        # with patch.py and is unused.
        self.zip: Path = zip

    @override
    def requirements(self) -> ModuleRequirements:
        return ModuleRequirements(
            boot_images=set(),
            ext_images={'system'},
            selinux_patching=False,
        )

    @override
    def inject(
        self,
        boot_fs: dict[str, CpioFs],
        ext_fs: dict[str, ExtFs],
        sepolicies: Iterable[Path],
    ) -> None:
        logger.info(f'Injecting Hail: {self.zip}')

        system_fs = ext_fs['system']

        with zipfile.ZipFile(self.zip, 'r') as z:
            for path in z.namelist():
                if path.endswith('/'):
                    continue
                if not (path.endswith('.apk') or path.endswith('.xml')):
                    continue
                modules.zip_extract(z, path, system_fs)
```

- [ ] **Step 2: Register the module in `lib/modules/__init__.py`**

Find:
```python
    from lib.modules.alterinstaller import AlterInstallerModule
    from lib.modules.bcr import BCRModule
    from lib.modules.custota import CustotaModule
    from lib.modules.msd import MSDModule
    from lib.modules.oemunlockonboot import OEMUnlockOnBootModule

    return {
        'alterinstaller': AlterInstallerModule,
        'bcr': BCRModule,
        'custota': CustotaModule,
        'msd': MSDModule,
        'oemunlockonboot': OEMUnlockOnBootModule,
    }
```
Replace with:
```python
    from lib.modules.alterinstaller import AlterInstallerModule
    from lib.modules.bcr import BCRModule
    from lib.modules.custota import CustotaModule
    from lib.modules.hail import HailModule
    from lib.modules.msd import MSDModule
    from lib.modules.oemunlockonboot import OEMUnlockOnBootModule

    return {
        'alterinstaller': AlterInstallerModule,
        'bcr': BCRModule,
        'custota': CustotaModule,
        'hail': HailModule,
        'msd': MSDModule,
        'oemunlockonboot': OEMUnlockOnBootModule,
    }
```

### Task 2.3: Verify `patch.py` exposes `--module-hail`

**Files:** none

- [ ] **Step 1: Confirm the module registers and the CLI flag exists**

Run:
```bash
cd /home/andy/Desktop/ota/my-avbroot-setup
python3 -c "from lib.modules import all_modules; assert 'hail' in all_modules(), all_modules(); print('hail registered:', sorted(all_modules()))"
python3 patch.py --help 2>&1 | grep -- '--module-hail'
```
Expected: prints `hail registered: [...]` including `hail`, and the help output shows `--module-hail` and `--module-hail-sig`.

(If `patch.py --help` errors on missing imports, install requirements first: `python3 -m venv /tmp/avbenv && /tmp/avbenv/bin/pip install -r requirements.txt && /tmp/avbenv/bin/python patch.py --help | grep module-hail`.)

- [ ] **Step 2: Push the branch**

```bash
cd /home/andy/Desktop/ota/my-avbroot-setup
git add lib/modules/hail.py lib/modules/__init__.py
git commit -m "Add Hail privileged-system-app injection module"
git push -u origin hail-pinned
```

---

## Phase 3: pixeneos fork — Hail integration + repoint to mrbathwater/ota

Working directory: `/home/andy/Desktop/ota/pixeneos` (branch `main`).

### Task 3.1: Declare Hail + repoint repos in `declarations.sh`

**Files:**
- Modify: `src/declarations.sh`

- [ ] **Step 1: Repoint the device-JSON location to mrbathwater/ota**

Find:
```bash
REPOSITORY="PixeneOS" # GitHub repository name
USER="pixincreate"    # GitHub username
```
Replace with:
```bash
REPOSITORY="ota"        # GitHub repository name (OTA release + Pages host)
USER="mrbathwater"      # GitHub username
```

- [ ] **Step 2: Pin my-avbroot-setup to the fork branch that has the Hail module**

Find:
```bash
VERSION[AVBROOT_SETUP]="ae7c68c8e90a74991cc91bb0caa8c7e68b180846" # Commit hash
```
Replace with:
```bash
VERSION[AVBROOT_SETUP]="hail-pinned" # mrbathwater/my-avbroot-setup branch with the Hail module
```

- [ ] **Step 3: Add Hail version + repo coordinates next to the other VERSION entries**

Find:
```bash
VERSION[OEMUNLOCKONBOOT]="${VERSION[OEMUNLOCKONBOOT]:-1.3}"
```
Replace with:
```bash
VERSION[OEMUNLOCKONBOOT]="${VERSION[OEMUNLOCKONBOOT]:-1.3}"
VERSION[HAIL]="${VERSION[HAIL]:-v1.10.0-priv1}"

# Hail (privileged system app injected into the rootless OTA)
HAIL_REPOSITORY="mrbathwater/Hail"
HAIL_APK_URL="${DOMAIN}/${HAIL_REPOSITORY}/releases/download/${VERSION[HAIL]}/Hail.apk"
HAIL_PACKAGE="com.aistra.hail"
```

- [ ] **Step 4: Add the Hail enable flag next to the other module flags**

Find:
```bash
ADDITIONALS[OEMUNLOCKONBOOT]="${ADDITIONALS[OEMUNLOCKONBOOT]:-true}" # toggle OEM unlock button on boot
```
Replace with:
```bash
ADDITIONALS[OEMUNLOCKONBOOT]="${ADDITIONALS[OEMUNLOCKONBOOT]:-true}" # toggle OEM unlock button on boot
ADDITIONALS[HAIL]="${ADDITIONALS[HAIL]:-true}"                       # Hail privileged system app (rootless freeze/force-stop)
```

### Task 3.2: Point the my-avbroot-setup clone at the fork

**Files:**
- Modify: `src/util_functions.sh` (`url_constructor`)

- [ ] **Step 1: Use the mrbathwater fork URL for my-avbroot-setup**

Find:
```bash
  # `my-avbroot-setup` is git repository
  if [[ "${repository}" == "my-avbroot-setup" ]]; then
    URL="${DOMAIN}/${user}/${repository}"
  else
```
Replace with:
```bash
  # `my-avbroot-setup` is git repository (use our fork, which carries the Hail module)
  if [[ "${repository}" == "my-avbroot-setup" ]]; then
    URL="${DOMAIN}/mrbathwater/${repository}"
  else
```

### Task 3.3: Build the Hail priv-app module zip

**Files:**
- Modify: `src/util_functions.sh` (add `build_hail_module`, call it from `check_and_download_dependencies`)

- [ ] **Step 1: Add the `build_hail_module` function (place it directly above `flag_check`)**

Insert before `function flag_check() {`:
```bash
# Build the Hail privileged-system-app module zip consumed by `--module-hail`.
# Downloads the prebuilt Hail.apk and lays it out at its target system paths
# together with a privapp-permissions allowlist.
function build_hail_module() {
  local module_root="${WORKDIR}/hail-module"
  local apk_dir="${module_root}/system/priv-app/Hail"
  local perm_dir="${module_root}/system/etc/permissions"
  local out_zip="${WORKDIR}/modules/hail.zip"

  if [ -f "${out_zip}" ]; then
    echo -e "\`hail.zip\` already exists in \`${WORKDIR}/modules\`."
    return
  fi

  echo -e "Building Hail module from ${HAIL_APK_URL}..."
  rm -rf "${module_root}"
  mkdir -p "${apk_dir}" "${perm_dir}"

  curl -sLf "${HAIL_APK_URL}" --output "${apk_dir}/Hail.apk"

  # Privileged-permission allowlist. MUST list exactly the privileged
  # permissions Hail declares (see Task 3.6 verification), or GrapheneOS
  # (ro.control_privapp_permissions=enforce) can boot-loop.
  cat >"${perm_dir}/privapp-permissions-${HAIL_PACKAGE}.xml" <<EOF
<?xml version="1.0" encoding="utf-8"?>
<permissions>
    <privapp-permissions package="${HAIL_PACKAGE}">
        <permission name="android.permission.FORCE_STOP_PACKAGES"/>
        <permission name="android.permission.CHANGE_COMPONENT_ENABLED_STATE"/>
        <permission name="android.permission.MANAGE_APP_OPS_MODES"/>
        <permission name="android.permission.PACKAGE_USAGE_STATS"/>
    </privapp-permissions>
</permissions>
EOF

  ( cd "${module_root}" && zip -qr "$(realpath "${out_zip}")" system )
  rm -rf "${module_root}"
  echo -e "\`hail.zip\` built at \`${out_zip}\`."
}
```

- [ ] **Step 2: Call it at the end of `check_and_download_dependencies`**

Find (the end of `check_and_download_dependencies`, the magisk retry block):
```bash
  # Retry logic for magisk
  if [[ "${ADDITIONALS[ROOT]}" == 'true' ]]; then
    RETRY_COUNT=0 # Reset retry count for magisk
    while true; do
      # Magisk is an exception as it is an APK and hence we do the get call directly and verify
      URL="${MAGISK[URL]}/releases/download/${VERSION[MAGISK]}/app-release.apk"
      echo "URL for \`magisk\`: ${URL}"
      get "magisk" "${URL}"
      verify_downloads "magisk"

      [[ "${ADDITIONALS[RETRY]}" == "true" ]] && [[ "${RETRY}" == "true" ]] || break
    done
  fi
}
```
Replace with:
```bash
  # Retry logic for magisk
  if [[ "${ADDITIONALS[ROOT]}" == 'true' ]]; then
    RETRY_COUNT=0 # Reset retry count for magisk
    while true; do
      # Magisk is an exception as it is an APK and hence we do the get call directly and verify
      URL="${MAGISK[URL]}/releases/download/${VERSION[MAGISK]}/app-release.apk"
      echo "URL for \`magisk\`: ${URL}"
      get "magisk" "${URL}"
      verify_downloads "magisk"

      [[ "${ADDITIONALS[RETRY]}" == "true" ]] && [[ "${RETRY}" == "true" ]] || break
    done
  fi

  # Build the Hail module (rootless only; no signature verification needed)
  if [[ "${ADDITIONALS[HAIL]}" == 'true' ]]; then
    build_hail_module
  fi
}
```

### Task 3.4: Inject Hail in `patch_ota`

**Files:**
- Modify: `src/util_functions.sh` (`patch_ota`)

- [ ] **Step 1: Append the `--module-hail` arg after the alterinstaller module/sig args**

Find:
```bash
    # Module signatures
    args+=("--module-custota-sig" "${WORKDIR}/signatures/custota.zip.sig")
    args+=("--module-msd-sig" "${WORKDIR}/signatures/msd.zip.sig")
    args+=("--module-bcr-sig" "${WORKDIR}/signatures/bcr.zip.sig")
    args+=("--module-oemunlockonboot-sig" "${WORKDIR}/signatures/oemunlockonboot.zip.sig")
    args+=("--module-alterinstaller-sig" "${WORKDIR}/signatures/alterinstaller.zip.sig")
```
Replace with:
```bash
    # Module signatures
    args+=("--module-custota-sig" "${WORKDIR}/signatures/custota.zip.sig")
    args+=("--module-msd-sig" "${WORKDIR}/signatures/msd.zip.sig")
    args+=("--module-bcr-sig" "${WORKDIR}/signatures/bcr.zip.sig")
    args+=("--module-oemunlockonboot-sig" "${WORKDIR}/signatures/oemunlockonboot.zip.sig")
    args+=("--module-alterinstaller-sig" "${WORKDIR}/signatures/alterinstaller.zip.sig")

    # Hail privileged system app (rootless freeze/force-stop). No signature:
    # the module is built locally and HailModule skips signature verification.
    if [[ "${ADDITIONALS[HAIL]}" == 'true' ]]; then
      args+=("--module-hail" "${WORKDIR}/modules/hail.zip")
    fi
```

### Task 3.5: Lint the modified bash

**Files:** none

- [ ] **Step 1: Syntax-check the scripts**

Run:
```bash
cd /home/andy/Desktop/ota/pixeneos
bash -n src/declarations.sh && bash -n src/util_functions.sh && echo "syntax OK"
```
Expected: `syntax OK`.

- [ ] **Step 2: Unit-test the module builder produces the right layout**

Run:
```bash
cd /home/andy/Desktop/ota/pixeneos
mkdir -p .tmp/modules
# Stub the download so the test is offline: create a fake apk and override curl via a wrapper.
cat > /tmp/fake_curl.sh <<'SH'
#!/usr/bin/env bash
# crude stub: last arg after --output is the destination
out=""
prev=""
for a in "$@"; do [[ "$prev" == "--output" ]] && out="$a"; prev="$a"; done
printf 'FAKE_APK' > "$out"
SH
chmod +x /tmp/fake_curl.sh
( source src/declarations.sh
  HAIL_APK_URL="file:///dev/null"
  curl() { /tmp/fake_curl.sh "$@"; }
  source src/util_functions.sh
  build_hail_module
  echo "--- contents of hail.zip ---"
  unzip -l .tmp/modules/hail.zip )
```
Expected: the listing contains `system/priv-app/Hail/Hail.apk` and `system/etc/permissions/privapp-permissions-com.aistra.hail.xml`.

- [ ] **Step 3: Clean up the test artifacts and commit**

```bash
cd /home/andy/Desktop/ota/pixeneos
rm -rf .tmp
git add src/declarations.sh src/util_functions.sh
git commit -m "Inject Hail as a privileged system app; target mrbathwater/ota"
git push origin main
```

### Task 3.6: Verify the allowlist matches Hail's manifest

**Files:** none (correctness gate for the boot-loop risk)

- [ ] **Step 1: Dump the privileged permissions Hail declares and reconcile**

```bash
cd /home/andy/Desktop/ota/Hail
unzip -p /tmp/Hail.apk AndroidManifest.xml | \
  "$ANDROID_HOME"/build-tools/*/aapt2 dump permissions /tmp/Hail.apk 2>/dev/null | grep uses-permission || \
  grep -oE 'android.permission.[A-Z_]+' app/src/main/AndroidManifest.xml | sort -u
```
Expected: every `signature|privileged` permission Hail declares — at minimum `FORCE_STOP_PACKAGES`, `CHANGE_COMPONENT_ENABLED_STATE`, `MANAGE_APP_OPS_MODES`, `PACKAGE_USAGE_STATS` — appears in the allowlist written in Task 3.3 Step 1. If Hail declares an additional protected permission, add a matching `<permission name="…"/>` line to the heredoc in `src/util_functions.sh` and re-commit. Normal/runtime permissions (e.g. `POST_NOTIFICATIONS`, `FOREGROUND_SERVICE*`, `QUERY_ALL_PACKAGES`, `REQUEST_DELETE_PACKAGES`) must NOT be added.

---

## Phase 4: mrbathwater/ota — magisk-only schnatterer + new rootless job

Working directory: `/home/andy/Desktop/ota/ota` (branch `rootless-pixeneos-hail`).

### Task 4.1: Restrict the schnatterer job to magisk-only

**Files:**
- Modify: `.github/workflows/release-single.yaml`

- [ ] **Step 1: Force `SKIP_ROOTLESS=true` in the existing `build-device` job's "Set inputs" step**

Find:
```yaml
          echo "SKIP_ROOTLESS=$(echo '${{ github.event.inputs.skip-rootless || '' }}' | xargs)" >> $GITHUB_ENV
```
Replace with:
```yaml
          # Rootless is now built by the pixeneos engine in the build-rootless job below.
          # Force the schnatterer build to magisk-only so it does not also publish rootless/<device>.json.
          echo "SKIP_ROOTLESS=true" >> $GITHUB_ENV
```

### Task 4.2: Add the rootless build job

**Files:**
- Modify: `.github/workflows/release-single.yaml`

- [ ] **Step 1: Add a `publish-pages` input to both input lists**

In the `workflow_call.inputs` block, find:
```yaml
      magisk-preinit-device:
        type: string
        default: ''
```
Replace with:
```yaml
      magisk-preinit-device:
        type: string
        default: ''
      publish-pages:
        type: string
        default: 'true'
```

In the `workflow_dispatch.inputs` block, find:
```yaml
      additional-env:
        description: Additional env var key value pairs, space separated, e.g. "A=1 B=2"
        default: ''
```
Replace with:
```yaml
      additional-env:
        description: Additional env var key value pairs, space separated, e.g. "A=1 B=2"
        default: ''
      publish-pages:
        description: Commit rootless device JSON to gh-pages (set false for test builds)
        default: 'true'
```

- [ ] **Step 2: Append the rootless job at the end of the file (same indentation level as `build-device`)**

```yaml
  build-rootless:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    permissions:
      contents: write
    steps:
      - name: Checkout OTA repository (release assets + GitHub Pages)
        uses: actions/checkout@v4
        with:
          fetch-depth: 0 # Allow switching to the gh-pages branch

      - name: Checkout pixeneos build engine
        uses: actions/checkout@v4
        with:
          repository: mrbathwater/pixeneos
          ref: main
          path: pixeneos
          fetch-depth: 1

      - name: Resolve inputs
        run: |
          echo "DEVICE_NAME=$(echo '${{ github.event.inputs.device-id || inputs.device-id || 'shiba' }}' | xargs)" >> $GITHUB_ENV
          echo "PUBLISH_PAGES=$(echo '${{ github.event.inputs.publish-pages || inputs.publish-pages || 'true' }}' | xargs)" >> $GITHUB_ENV

      - run: sudo apt-get update && sudo apt-get install -y jq curl git unzip xxd zip

      - name: Install Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Build rootless OTA with pixeneos (+ Hail)
        shell: bash
        working-directory: pixeneos
        env:
          INTERACTIVE_MODE: false
          CLEANUP: true
          DEVICE_NAME: ${{ env.DEVICE_NAME }}
          GRAPHENEOS_UPDATE_CHANNEL: stable-security-preview
          ADDITIONALS_ROOT: false
          PASSPHRASE_AVB: ${{ secrets.PASSPHRASE_AVB }}
          PASSPHRASE_OTA: ${{ secrets.PASSPHRASE_OTA }}
          KEYS_AVB_BASE64: ${{ secrets.KEY_AVB_BASE64 }}
          KEYS_OTA_BASE64: ${{ secrets.KEY_OTA_BASE64 }}
          KEYS_CERT_OTA_BASE64: ${{ secrets.CERT_OTA_BASE64 }}
        run: |
          set -euo pipefail
          # Build only (download deps + patch + sign + csig + device JSON).
          # Publishing is done by this repo in the next steps.
          . src/main.sh
          echo "OUTPUTS_PATCHED_OTA=${OUTPUTS[PATCHED_OTA]}" >> "$GITHUB_ENV"
          echo "GRAPHENEOS_VERSION=${VERSION[GRAPHENEOS]}" >> "$GITHUB_ENV"

      - name: Upload build artifacts (for inspection)
        uses: actions/upload-artifact@v4
        with:
          name: rootless-${{ env.DEVICE_NAME }}
          path: |
            pixeneos/${{ env.OUTPUTS_PATCHED_OTA }}.csig
            pixeneos/${{ env.DEVICE_NAME }}.json

      - name: Upload OTA + csig to the release
        shell: bash
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          set -euo pipefail
          asset="pixeneos/${OUTPUTS_PATCHED_OTA}"
          # The magisk job runs in parallel and may not have created this
          # version's release yet (versions can differ per device), so create
          # it if missing. `|| true` tolerates a concurrent create race.
          if ! gh release view "${GRAPHENEOS_VERSION}" --repo "${{ github.repository }}" >/dev/null 2>&1; then
            gh release create "${GRAPHENEOS_VERSION}" --repo "${{ github.repository }}" \
              --title "${GRAPHENEOS_VERSION}" \
              --notes "Update to [GrapheneOS ${GRAPHENEOS_VERSION}](https://grapheneos.org/releases#${GRAPHENEOS_VERSION})." || true
          fi
          gh release upload "${GRAPHENEOS_VERSION}" \
            "${asset}" "${asset}.csig" \
            --repo "${{ github.repository }}" --clobber

      - name: Publish device JSON to gh-pages/rootless
        if: env.PUBLISH_PAGES != 'false'
        shell: bash
        run: |
          set -euo pipefail
          # Copy the device JSON out before switching branches (it is an
          # untracked file in the main checkout; stash is not needed because
          # gh-pages has no conflicting path).
          cp "pixeneos/${DEVICE_NAME}.json" "/tmp/${DEVICE_NAME}.json"
          git config user.name "GitHub Actions"
          git config user.email "actions@github.com"
          git checkout gh-pages
          mkdir -p rootless
          target="rootless/${DEVICE_NAME}.json"
          if ! grep -q "${GRAPHENEOS_VERSION}" "${target}" 2>/dev/null; then
            cp "/tmp/${DEVICE_NAME}.json" "${target}"
            git add "${target}"
          else
            echo "rootless/${DEVICE_NAME}.json already at ${GRAPHENEOS_VERSION}; skipping."
          fi
          if ! git diff-index --quiet HEAD; then
            git commit -m "rootless: ${DEVICE_NAME} -> GrapheneOS ${GRAPHENEOS_VERSION} (pixeneos + Hail)"
            for i in $(seq 1 10); do
              git pull --rebase origin gh-pages && git push origin gh-pages && break
              echo "retry $i"; sleep 2
            done
          fi
```

- [ ] **Step 3: Validate the workflow YAML parses**

Run:
```bash
cd /home/andy/Desktop/ota/ota
python3 -c "import yaml,sys; yaml.safe_load(open('.github/workflows/release-single.yaml')); print('YAML OK')"
```
Expected: `YAML OK`.

- [ ] **Step 4: Commit**

```bash
cd /home/andy/Desktop/ota/ota
git add .github/workflows/release-single.yaml
git commit -m "Build rootless flavor via pixeneos fork (+ Hail); schnatterer is now magisk-only"
```

---

## Phase 5: Integration test on bluejay (no live overwrite)

### Task 5.1: Dispatch a single-device rootless TEST build (gh-pages not touched)

**Files:** none

- [ ] **Step 1: Ensure a release exists, then test-build bluejay with `publish-pages=false`**

```bash
cd /home/andy/Desktop/ota/ota
git push -u origin rootless-pixeneos-hail
# Ensure the release for the latest GrapheneOS version exists (creates it if missing):
gh workflow run create-release.yaml --ref main
sleep 30
gh run watch "$(gh run list --workflow=create-release.yaml --limit 1 --json databaseId --jq '.[0].databaseId')" || true
# Test build for bluejay WITHOUT publishing to gh-pages:
gh workflow run release-single.yaml --ref rootless-pixeneos-hail -f device-id=bluejay -f publish-pages=false
sleep 30
gh run watch "$(gh run list --workflow=release-single.yaml --limit 1 --json databaseId --jq '.[0].databaseId')"
```
Expected: the `build-rootless` job succeeds. (The `build-device`/magisk job also runs and is a safe no-op when the magisk asset for this version already exists.)

- [ ] **Step 2: Inspect the produced device JSON and confirm the live JSON is unchanged**

```bash
cd /home/andy/Desktop/ota/ota
runid=$(gh run list --workflow=release-single.yaml --limit 1 --json databaseId --jq '.[0].databaseId')
gh run download "$runid" -n rootless-bluejay -D /tmp/rootless-bluejay
echo "=== built (candidate) JSON ===" && jq -r '.full.location // .location' /tmp/rootless-bluejay/bluejay.json
ver=$(gh release list --repo mrbathwater/ota --limit 1 --json tagName --jq '.[0].tagName')
echo "=== release assets ===" && gh release view "$ver" --repo mrbathwater/ota --json assets --jq '.assets[].name' | grep -E "bluejay-.*-rootless-.*\.zip(\.csig)?$"
echo "=== LIVE gh-pages JSON (must be unchanged) ===" && git fetch origin gh-pages -q && git show origin/gh-pages:rootless/bluejay.json | jq -r '.full.location // .location'
```
Expected: the candidate JSON's `location` is `github.com/mrbathwater/ota/releases/download/<ver>/bluejay-<ver>-rootless-<commit>.zip`; that `.zip` and `.zip.csig` are present on the release; the LIVE gh-pages JSON still shows the OLD (pre-change) location.

### Task 5.2: Device validation (manual, on a bluejay)

**Files:** none

- [ ] **Step 1: Sideload the candidate OTA and confirm the device boots**

```bash
cd /home/andy/Desktop/ota/ota
ver=$(gh release list --repo mrbathwater/ota --limit 1 --json tagName --jq '.[0].tagName')
gh release download "$ver" --repo mrbathwater/ota -p "bluejay-*-rootless-*.zip" -D /tmp --clobber
# Reboot the bluejay to recovery, then:
adb sideload /tmp/bluejay-*-rootless-*.zip
```
Expected: the device boots normally. A boot-loop means the privapp-permissions allowlist is wrong — revisit Task 3.6 before continuing.

- [ ] **Step 2: Confirm Hail is present and the combined mode works without root**

On device: open Hail → Settings → Working mode → select **System App - Force Stop + Disable** → select an app → Freeze.
Expected: the target app is force-stopped and then disabled, with no root and no Shizuku.

---

## Phase 6: Roll out to all three devices

### Task 6.1: Merge and let the daily matrix run

**Files:** none

- [ ] **Step 1: Merge the branch to main**

```bash
cd /home/andy/Desktop/ota/ota
git checkout main
git merge --no-ff rootless-pixeneos-hail -m "Rootless OTA via pixeneos + Hail for all devices"
git push origin main
```

- [ ] **Step 2: Trigger a full run and confirm all three rootless JSONs update**

```bash
gh workflow run release-multiple.yaml --ref main
gh run watch "$(gh run list --workflow=release-multiple.yaml --limit 1 --json databaseId --jq '.[0].databaseId')"
git fetch origin gh-pages
for d in bluejay tangorpro tokay; do
  echo "== $d =="; git show origin/gh-pages:rootless/$d.json | jq '.full.location // .location'
done
```
Expected: all three `rootless/<device>.json` point at `github.com/mrbathwater/ota/...rootless...` for the current GrapheneOS version. Custota clients on the rootless channel now receive the pixeneos + Hail build.

---

## Notes / guardrails

- **No new secrets required.** The rootless job reuses this repo's existing secrets (`KEY_AVB_BASE64`, `KEY_OTA_BASE64`, `CERT_OTA_BASE64`, `PASSPHRASE_AVB`, `PASSPHRASE_OTA`), mapped to the env names pixeneos expects. The Hail release workflow generates its own ephemeral keystore (priv-app privileges come from the allowlist, not the signature).
- **Version alignment.** `create-release.yaml` (schnatterer, `stable-security-preview`) creates the release tag; the rootless job uses the same channel so `VERSION[GRAPHENEOS]` matches the tag the assets are uploaded to.
- **If `avbroot`/`afsr` fail to download as prebuilt binaries** in the rootless job, add the Rust toolchain step from pixeneos's `release.yml` (`dtolnay/rust-toolchain@master`) before the build step.
- **Magisk/root flavor is never touched** — the `build-device` job still runs schnatterer exactly as before, only with rootless disabled.
