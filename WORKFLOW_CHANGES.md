# GitHub Actions Workflow Changes

## Problem

The original GitHub Actions workflows were checking out the `schnatterer/rooted-graphene` repository, which contains the original unmodified `rooted-ota.sh` script. This meant that all the privileged apps integration (BCR, MSD, AlterInstaller, bindhosts, AppManager) was not being included in the builds because the workflows were using the original script instead of your modified version.

## Solution

Updated the workflows to checkout **your local repository** instead of the upstream `schnatterer/rooted-graphene` repository.

## Files Modified

### 1. `.github/workflows/release-single.yaml`

**Before:**
```yaml
- name: Checkout rooted-graphene script repository
  uses: actions/checkout@v4
  with:
    repository: schnatterer/rooted-graphene
    ref: ${{ github.event.inputs.rooted-graphene-version || inputs.rooted-graphene-version || 'main' }} 
    fetch-depth: 1
```

**After:**
```yaml
- name: Checkout OTA repository (this repo with rooted-ota.sh)
  uses: actions/checkout@v4
  with:
    fetch-depth: 1
```

### 2. `.github/workflows/create-release.yaml`

**Before:**
```yaml
- name: Checkout rooted-graphene script repository
  uses: actions/checkout@v5
  with:
    repository: schnatterer/rooted-graphene
    ref: ${{ github.event.inputs.rooted-graphene-version || inputs.rooted-graphene-version || 'main' }}
    fetch-depth: 1
```

**After:**
```yaml
- name: Checkout OTA repository (this repo with rooted-ota.sh)
  uses: actions/checkout@v5
  with:
    fetch-depth: 1
```

## What Changed

1. **Removed `repository:` parameter** - No longer checks out `schnatterer/rooted-graphene`, now uses the default (your repository)
2. **Removed `ref:` parameter** - No longer needs version selection, uses your current branch/commit
3. **Updated step names** - Changed from "rooted-graphene script repository" to "OTA repository (this repo with rooted-ota.sh)" for clarity
4. **Kept `PAGES_REPO_FOLDER: 'ota'`** - The second checkout (for GitHub Pages) still uses the `ota` folder path as expected

## Impact

### Before These Changes
- ✗ Builds used the original script from `schnatterer/rooted-graphene`
- ✗ Only custota and oemunlockonboot were included
- ✗ New privileged apps (BCR, MSD, AlterInstaller, bindhosts, AppManager) were NOT included

### After These Changes
- ✓ Builds use YOUR modified script from this repository
- ✓ custota and oemunlockonboot are still included
- ✓ All 5 new privileged apps are now included in every build
- ✓ Both rootless and magisk builds get all the apps

## Verification

To verify the workflows are using your repository:

```bash
# Check that no references to schnatterer/rooted-graphene remain
grep -r "schnatterer/rooted-graphene" .github/workflows/
# Should return: (no output)

# Check that your script is being used
grep -r "Checkout OTA repository" .github/workflows/
# Should show the updated checkout steps
```

## Next Build

The next time you run the GitHub Actions workflow (manually or via scheduled run):
1. It will checkout YOUR repository
2. Use YOUR modified `rooted-ota.sh` with all the privileged apps
3. Build OTAs that include all 5 new privileged apps
4. Upload them to releases with all apps working

## No Manual Intervention Needed

The workflows will automatically:
- Download all APKs and modules
- Verify signatures (for chenxiaolong apps)
- Create Magisk module structures
- Include all modules in both rootless and magisk builds
- Upload to GitHub releases
- Update the OTA server on GitHub Pages

Everything is automated and ready to go!

