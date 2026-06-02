# Design: pixeneos + Hail rootless OTA

- **Date:** 2026-06-02
- **Status:** Approved (design); pending implementation plan
- **Scope:** Rootless flavor only. The magisk/root flavor is untouched.

## 1. Goal

Replace the current schnatterer-built **rootless** OTA flavor for **bluejay (Pixel 6a)**,
**tangorpro (Pixel Tablet)**, and **tokay (Pixel 9)** with a **pixeneos-built** rootless OTA
that carries pixeneos's full module set **plus Hail as a privileged system app**. Everything is
signed with the user's existing AVB/OTA keys (matching their custom locked bootloader) and
published in place to `mrbathwater/ota` GitHub Releases + `gh-pages/rootless/<device>.json`, so
Custota on-device upgrades seamlessly. The magisk/root flavor continues to be built by
`schnatterer/rooted-graphene` exactly as it is today.

## 2. Approved decisions

1. **Build engine:** Fork pixeneos (`mrbathwater/pixeneos`) and have this repo's workflow call it
   for the rootless build — rather than porting pixeneos's logic into this repo's `rooted-ota.sh`.
   Chosen for fidelity to "use pixeneos" and to inherit pixeneos's upstream version sync (renovate).
2. **Module set:** Full pixeneos set — Custota, OEMUnlockOnBoot, BCR, MSD, AlterInstaller — **plus
   Hail**.
3. **Hail combined action:** Add a new privileged-system-app combined mode
   (`MODE_PRIVAPP_STOP_DISABLE`) to the Hail fork. The fork's existing combined force-stop+freeze
   (`MODE_SU_STOP_DISABLE`) is **root-only**; the rootless OTA has no root, so a priv-app equivalent
   is required for the feature to function.

### Chosen defaults

- **Asset hosting:** OTA zip, `.csig`, and device JSON live on `mrbathwater/ota` (uniform with the
  magisk flavor), **not** on the pixeneos fork's own releases/gh-pages.
- **Update channel:** `stable-security-preview` (matches the current rootless for version
  continuity).
- **Hail APK signing:** any key (a generated release keystore). Privileged-permission grants come
  from the priv-app allowlist, not the APK signature.

## 3. Background: how the repo works today

- `.github/workflows/create-release.yaml` — cron (every 2h). Checks out **this** repo's
  `rooted-ota.sh`, detects the latest GrapheneOS version for an arbitrary device, creates the
  GitHub Release shell, then triggers `release-multiple`.
- `.github/workflows/release-multiple.yaml` — matrix of the three devices
  (tangorpro/sda5, tokay/sda10, bluejay/sda8), calls `release-single` per device.
- `.github/workflows/release-single.yaml` — **checks out `schnatterer/rooted-graphene` (not this
  repo's copy)** at `release-single.yaml:55-60` and runs its `rooted-ota.sh`. Builds the magisk
  and/or rootless flavors, signs with the user's keys (base64 secrets), generates Custota `.csig` +
  device JSON, uploads release assets, and commits device JSON to `gh-pages`.
- `gh-pages` branch layout (what Custota connects to via `mrbathwater.github.io/ota/`):
  - `magisk/{bluejay,tangorpro,tokay}.json` (root flavor)
  - `rootless/{bluejay,tangorpro,tokay}.json` (rootless flavor)
- Signing keys are provided as base64 secrets (`KEY_AVB_BASE64`, `KEY_OTA_BASE64`,
  `CERT_OTA_BASE64`) + passphrases (`PASSPHRASE_AVB`, `PASSPHRASE_OTA`). `avb_pkmd.bin` is committed.

### pixeneos (upstream `pixincreate/pixeneos`)

- Same toolchain as schnatterer: avbroot + `chenxiaolong/my-avbroot-setup` `patch.py` + custota-tool.
- `src/declarations.sh` defines an `ADDITIONALS` map of modules; **rootless by default**
  (`ADDITIONALS[ROOT]=false`, Magisk only when true).
- `src/util_functions.sh:patch_ota()` (≈ lines 162–238) builds `patch.py` args:
  `--module-custota`, `--module-msd`, `--module-bcr`, `--module-oemunlockonboot`,
  `--module-alterinstaller` (+ matching `--module-*-sig`).
- `my_avbroot_setup()` (≈ lines 246–255) builds the device-JSON `location` URL from `USER` /
  `REPOSITORY` vars → this is the override point to make the location target `mrbathwater/ota`.
- `generate_ota_info()` (≈ line 459) names the output:
  `${DEVICE_NAME}-${VERSION[GRAPHENEOS]}-${flavor}-<commit>.zip`.
- Build vs. publish are separable functions (`create_ota` builds/signs; csig/update-info generated
  separately), so a fork can build+sign+csig and publish into an external repo.
- Versions of note: Custota **6.1** (vs. this repo's 5.22), my-avbroot-setup pinned commit
  `ae7c68c8e90a74991cc91bb0caa8c7e68b180846`.
- pixeneos is device-agnostic (`DEVICE_NAME` param); it publicly releases only bluejay, but
  tangorpro/tokay are standard GrapheneOS devices the same script handles.

### my-avbroot-setup module injection (at pinned commit `ae7c68c8`)

- `lib/modules/__init__.py` defines `Module` (ABC: `requirements()`, `inject()`) and
  `all_modules()` → `{alterinstaller, bcr, custota, msd, oemunlockonboot}`.
- `patch.py` auto-generates `--module-<name>` and `--module-<name>-sig` flags from `all_modules()`
  (≈ lines 152–161), constructs each module (≈ lines 199–209), and calls `module.inject()` (≈ line
  283).
- A module zip is just system-partition files laid out at target paths. `bcr.py` pattern:
  `verify_ssh_sig(...)` against chenxiaolong's key, then extract every `.apk`/`.xml` into the
  `system` ext image, optionally add an init script. `requirements()` →
  `ext_images={'system'}, selinux_patching=False`.

### Hail fork (`mrbathwater/Hail`, branch `master`)

- Package `com.aistra.hail`. The combined force-stop+freeze feature is already merged (PR #1).
- `HailData.kt` working modes (≈ lines 53–89): includes `MODE_SU_STOP_DISABLE` (root combined),
  and separate `MODE_PRIVAPP_STOP` / `MODE_PRIVAPP_DISABLE` — but **no** privapp combined mode.
- `AppManager.kt:setAppFrozen()` (≈ lines 54–76): `MODE_SU_STOP_DISABLE` →
  `HShell.forceStopApp(pkg) && HShell.setAppDisabled(pkg, true)`; `MODE_PRIVAPP_STOP` →
  `HPackages.forceStopApp(pkg)`; `MODE_PRIVAPP_DISABLE` → `HPackages.setAppDisabled(pkg, frozen)`.
  Both `HPackages` calls already exist → the new combined mode is a ~3-line addition.
- `AndroidManifest.xml` declares the privileged perms needed: `FORCE_STOP_PACKAGES`,
  `CHANGE_COMPONENT_ENABLED_STATE`, `MANAGE_APP_OPS_MODES`, `PACKAGE_USAGE_STATS`,
  plus `INTERACT_ACROSS_USERS_FULL` (component-level).
- Build: `.github/workflows/build-test-apk.yml` (`assembleDebug`/`assembleRelease`) and
  `android.yml` (release signing via keystore secrets, gated to `aistra0528`). **No releases/tags
  exist yet** → an APK must be produced by a fork workflow.

## 4. The four repos and their roles

| Repo | Status | Role in this design |
|---|---|---|
| `mrbathwater/Hail` | exists | Add `MODE_PRIVAPP_STOP_DISABLE`; add a release workflow that publishes a signed APK |
| `mrbathwater/my-avbroot-setup` | **new fork** of `chenxiaolong/my-avbroot-setup` @ `ae7c68c8` | Add `lib/modules/hail.py`; register it so `patch.py` exposes `--module-hail` |
| `mrbathwater/pixeneos` | **new fork** of `pixincreate/pixeneos` | Rootless build engine; add Hail module; point at the my-avbroot-setup fork; publish into `mrbathwater/ota` |
| `mrbathwater/ota` | local clone | `release-single.yaml`: magisk job stays schnatterer (magisk-only); add a rootless job that runs the pixeneos fork |

`gh` is authenticated as `mrbathwater` with `repo` + `workflow` scopes, so the two new forks can be
created during implementation.

## 5. Detailed component changes

### 5.1 `mrbathwater/Hail`

- `app/src/main/kotlin/com/aistra/hail/app/HailData.kt`
  - Add `const val MODE_PRIVAPP_STOP_DISABLE = PRIVAPP + STOP_DISABLE`.
  - Add it to `WORKING_MODE_VALUES`.
- `app/src/main/kotlin/com/aistra/hail/app/AppManager.kt:setAppFrozen()`
  - Add branch:
    ```kotlin
    HailData.MODE_PRIVAPP_STOP_DISABLE -> if (frozen) {
        HPackages.forceStopApp(packageName) && HPackages.setAppDisabled(packageName, true)
    } else {
        HPackages.setAppDisabled(packageName, false)
    }
    ```
- `app/src/main/res/values/arrays.xml` + `values/strings.xml`: add the picker entry/label
  (mirror the `MODE_SU_STOP_DISABLE` strings added in PR #1). Verify other `values-*/strings.xml`
  don't need the new key (fallback to default locale is acceptable).
- New `.github/workflows/release-apk.yml`: on tag/dispatch, build a signed release APK (generated
  keystore from repo secrets) and publish a GitHub Release with a stable asset name
  (e.g. `Hail-<versionName>.apk`). This is the artifact pixeneos downloads.

### 5.2 `mrbathwater/my-avbroot-setup` (fork @ `ae7c68c8`)

- New `lib/modules/hail.py` — `HailModule(Module)` modeled on `bcr.py`:
  - `requirements()` → `ModuleRequirements(boot_images=set(), ext_images={'system'}, selinux_patching=False)`.
  - `inject()` → extract the module zip's `.apk`/`.xml` into `ext_fs['system']` (no init script
    needed). **Skip** `verify_ssh_sig` (our own module). Make the `sig` argument optional.
- `lib/modules/__init__.py` → register `'hail': HailModule` in `all_modules()`.
- If `patch.py`'s arg validation requires a `--module-<name>-sig` whenever `--module-<name>` is
  given, relax it so `hail` may be passed without a sig (or accept an optional user-signed sig).

### 5.3 `mrbathwater/pixeneos` (fork)

- `src/declarations.sh`:
  - Add `ADDITIONALS[HAIL]="${ADDITIONALS[HAIL]:-true}"`, `VERSION[HAIL]=<Hail release tag>`, and
    Hail repo coordinates (`mrbathwater/Hail`).
  - Repoint the my-avbroot-setup source (user/repo and pinned ref) to `mrbathwater/my-avbroot-setup`.
- Dependency download (`fetcher.sh` / `util_functions.sh`): download the Hail release APK; assemble
  the **Hail module zip** with this layout:
  - `system/priv-app/Hail/Hail.apk`
  - `system/etc/permissions/privapp-permissions-com.aistra.hail.xml` — allowlists exactly the
    privileged perms Hail declares (see §8 risk).
- `patch_ota()`: when `ADDITIONALS[HAIL]` is true, append `--module-hail <hail.zip>` (no sig, or an
  optional user sig) to the `patch.py` args.
- Publishing: configure the fork to target `mrbathwater/ota` — release assets uploaded to the
  `mrbathwater/ota` release for the version, device JSON committed to `mrbathwater/ota`
  `gh-pages/rootless/<device>.json`, and the device-JSON `location` URL pointing at
  `mrbathwater/ota` release downloads (schnatterer-style external-repo targeting via env, e.g.
  `GITHUB_REPO` / a pages-repo-folder). Build/sign/csig run without the fork self-releasing.

### 5.4 `mrbathwater/ota` (this repo)

- `.github/workflows/release-single.yaml`:
  - **Magisk job (existing schnatterer path):** force `SKIP_ROOTLESS=true` so it builds only the
    magisk flavor. Keep `magisk-preinit-device` per device.
  - **New rootless job:** checkout `mrbathwater/pixeneos`, install deps, run it per device with:
    `DEVICE_NAME`, the key/passphrase secrets, `GRAPHENEOS_UPDATE_CHANNEL=stable-security-preview`,
    Hail enabled, and the `mrbathwater/ota` publish target (release + `gh-pages/rootless/`). Checkout
    `mrbathwater/ota` for the gh-pages push (as the magisk path already does).
- `release-multiple.yaml` (3-device matrix) and `create-release.yaml` (version detection + release
  shell creation) remain unchanged.
- After validation, `gh-pages/rootless/*.json` is overwritten in place → existing rootless users
  receive the new pixeneos+Hail build through Custota with no client reconfiguration.

## 6. Data flow (end to end)

1. `create-release` (cron) detects a new GrapheneOS version → creates Release `<version>` on
   `mrbathwater/ota` → triggers `release-multiple`.
2. `release-multiple` → `release-single` per device (bluejay, tangorpro, tokay).
3. `release-single`:
   - **(a) Magisk job** (schnatterer, `SKIP_ROOTLESS=true`): builds `<device>-…-magisk-….zip`, signs
     with the user keys, generates csig + device JSON, uploads to the `mrbathwater/ota` release and
     commits `gh-pages/magisk/<device>.json`. *(unchanged)*
   - **(b) Rootless job** (pixeneos fork): downloads the GrapheneOS OTA, injects
     BCR/MSD/AlterInstaller/OEMUnlockOnBoot/Custota 6.1 **+ Hail** (via the forked my-avbroot-setup),
     signs with the user keys, generates csig + device JSON (location → `mrbathwater/ota`), uploads
     to the `mrbathwater/ota` release and commits `gh-pages/rootless/<device>.json`. *(new)*
4. Custota on-device polls `mrbathwater.github.io/ota/rootless/<device>.json`, downloads the new OTA,
   and installs it (the custom locked bootloader accepts it because it is signed with the user keys).
5. The user opens Hail (now a privileged system app), selects working mode
   "Privileged: force-stop + disable," selects apps, and the combined action runs without root.

## 7. Testing & rollout

- Build the rootless flavor for **one device (bluejay) first** via `workflow_dispatch`, using the
  existing test path (`UPLOAD_TEST_OTA` / test folder) so it does **not** overwrite the live
  `rootless/` JSONs.
- Sideload-verify the test OTA on a bluejay (or at minimum confirm the patched OTA builds, signs,
  and that custota-tool produces a valid csig + device JSON).
- Confirm the device **boots** (validates the privapp-permissions allowlist) and that Hail appears as
  a system app with the new combined mode functioning.
- Only then enable the rootless job for all three devices and let it overwrite the live JSONs.

## 8. Risks

- **Privileged-permission allowlist (highest risk).** GrapheneOS enforces
  `ro.control_privapp_permissions=enforce`. If `privapp-permissions-com.aistra.hail.xml` does not
  exactly match the privileged permissions Hail's manifest declares, the device can **boot-loop**.
  Mitigation: derive the allowlist directly from the built APK's manifest; test boot on one device
  before rollout.
- **Hail APK signature.** Irrelevant for privileged grants (allowlist-based), but the app must be
  signed consistently. A generated release key is fine.
- **my-avbroot-setup drift.** The Hail module must be added on top of pixeneos's pinned commit
  `ae7c68c8`; keep the fork pinned so injection internals stay compatible.
- **Custota version mismatch across flavors.** Rootless uses Custota 6.1, magisk uses 5.22. These
  are independent device JSONs and Custota apps; no shared state, so acceptable.
- **External-repo publishing from the fork.** Targeting `mrbathwater/ota`'s releases + gh-pages from
  the pixeneos fork needs the workflow's `GITHUB_TOKEN`/PAT to have write access (it does — same
  repo, `secrets.GITHUB_TOKEN`).

## 9. Out of scope

- The magisk/root flavor (unchanged).
- Devices beyond bluejay, tangorpro, tokay.
- Changing or regenerating the user's signing keys.

## 10. Open implementation details (resolve during planning)

- Exact env interface the pixeneos fork exposes for "build + publish to external repo `mrbathwater/ota`"
  (reuse pixeneos vars vs. add schnatterer-style `GITHUB_REPO` / `PAGES_REPO_FOLDER`).
- Whether to build the Hail module zip inside the pixeneos fork or publish it ready-made from the
  Hail repo's release workflow.
- Final privapp-permissions allowlist contents (enumerate from the manifest).
- Hail release versioning/tagging scheme consumed by `VERSION[HAIL]`.
