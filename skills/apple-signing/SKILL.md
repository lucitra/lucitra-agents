# Apple Signing & Release for Tauri Desktop Apps

Set up Apple code signing secrets and publish signed releases. Two modes:
- **Setup**: Configure signing secrets (one-time)
- **Release**: Bump version, tag, and publish (frequent)

## Arguments

Parse for:
- **repo** (optional): `owner/repo` — defaults to current git remote
- **--verify**: Only verify existing secrets, don't set new ones
- **--release [version]**: Skip setup, go straight to release. Version is optional (auto-increments minor if omitted)
- **--status**: Check status of the latest release workflow run

## Quick Release Workflow (`--release`)

Use this for frequent publishes. Assumes signing secrets are already configured.

### Step 1: Verify secrets exist

```bash
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)
APPLE_COUNT=$(gh secret list -R $REPO | grep -c APPLE)
TAURI_COUNT=$(gh secret list -R $REPO | grep -c TAURI_SIGNING)

if [ "$APPLE_COUNT" -lt 6 ] || [ "$TAURI_COUNT" -lt 2 ]; then
  echo "Missing signing secrets. Run /apple-signing (without --release) first."
  exit 1
fi
```

### Step 2: Determine version

If version provided as argument, use it. Otherwise auto-increment:

```bash
# Read current version from tauri.conf.json
CURRENT=$(jq -r .version apps/studio/src-tauri/tauri.conf.json)
# Increment minor: 0.2.0 → 0.3.0
NEXT=$(echo $CURRENT | awk -F. '{printf "%d.%d.%d", $1, $2+1, 0}')
```

Confirm with user: "Releasing v{NEXT} (current: v{CURRENT}). Proceed?"

### Step 3: Bump version in all files

Update version in these files (all must match):
- `apps/studio/src-tauri/tauri.conf.json` → `"version": "X.Y.Z"`
- `apps/studio/package.json` → `"version": "X.Y.Z"`
- `apps/studio/src-tauri/Cargo.toml` → `version = "X.Y.Z"`
- `package.json` (root) → `"version": "X.Y.Z"`

### Step 4: Commit, push, and tag

```bash
git add apps/studio/package.json apps/studio/src-tauri/Cargo.toml \
  apps/studio/src-tauri/tauri.conf.json package.json

git commit -m "chore: bump version to v${NEXT} for release

Co-Authored-By: Claude <noreply@anthropic.com>
part of LUC-383"

git push origin dev
git tag "v${NEXT}"
git push origin "v${NEXT}"
```

### Step 5: Monitor release workflow

```bash
# Wait a moment for the workflow to start
sleep 5
gh run list -R $REPO -w "Release Desktop App" --limit 1
```

Report the workflow URL so the user can watch it. Estimated build time: ~15-20 minutes.

### Step 6: Verify publication

After the workflow completes:

```bash
gh release list -R $REPO --limit 3
# The new version should appear as "Latest" (not Draft)
```

Confirm the `latest.json` asset is present — this is what the in-app updater reads.

## Release Status (`--status`)

Check the latest release workflow run:

```bash
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)
gh run list -R $REPO -w "Release Desktop App" --limit 3
# For the latest run:
gh run view <run-id> -R $REPO
```

Report: status (in_progress/completed/failed), which platform builds succeeded, and whether the release was published.

## Setup Workflow (first-time only)

### Phase 1: Auto-detect what we can

```bash
# Detect repo
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)

# Find signing identity
IDENTITY=$(security find-identity -v -p codesigning | grep "Developer ID Application" | head -1)
# Extract: full identity string, team ID
```

If no Developer ID Application certificate found, tell the user:
1. Open Xcode → Settings → Accounts → Manage Certificates
2. Click + → Developer ID Application
3. Re-run this skill

### Phase 2: Check existing secrets

```bash
gh secret list -R $REPO | grep -iE 'APPLE|TAURI'
```

Report which secrets exist and which are missing. If all 6 Apple secrets + 2 Tauri secrets exist, skip to verification.

### Phase 3: Prompt for manual steps

**Only two things require the user to do manually:**

#### Manual Step 1: Export .p12 certificate
Tell the user:
> Open **Keychain Access** → search "Developer ID Application" → right-click the certificate → **Export Items...** → save as `lucitra-signing.p12` on Desktop → set a password.
>
> Tell me the password you set and I'll handle the rest.

Wait for the user to confirm and provide the password.

#### Manual Step 2: App-specific password
Tell the user:
> Go to https://appleid.apple.com/account/manage → App-Specific Passwords → click **+** → name it "Tauri Notarization" → copy the password.
>
> Paste it here.

Wait for the user to provide the password.

Also ask:
> What's your Apple ID email address?

### Phase 4: Set all secrets automatically

Once the user provides the .p12 password, app-specific password, and Apple ID email:

```bash
# Auto-detected values
SIGNING_IDENTITY="Developer ID Application: Name (TEAM_ID)"  # from Phase 1
TEAM_ID="XXXXXXXXXX"  # extracted from identity

# Certificate (base64)
base64 -i ~/Desktop/lucitra-signing.p12 | gh secret set APPLE_CERTIFICATE -R $REPO

# User-provided values
gh secret set APPLE_CERTIFICATE_PASSWORD -R $REPO -b "$P12_PASSWORD"
gh secret set APPLE_SIGNING_IDENTITY -R $REPO -b "$SIGNING_IDENTITY"
gh secret set APPLE_ID -R $REPO -b "$APPLE_ID_EMAIL"
gh secret set APPLE_PASSWORD -R $REPO -b "$APP_SPECIFIC_PASSWORD"
gh secret set APPLE_TEAM_ID -R $REPO -b "$TEAM_ID"
```

### Phase 5: Verify

```bash
# Confirm all 6 secrets are set
gh secret list -R $REPO | grep -c APPLE
# Should be 6

# Also check Tauri signing keys
gh secret list -R $REPO | grep TAURI_SIGNING_PRIVATE_KEY
```

Report:
- Which secrets are set (with timestamps)
- Whether Tauri signing key is also configured
- Next step: "Run `/apple-signing --release` to publish your first release"

### Phase 6: Cleanup reminder

Remind the user:
> Delete `~/Desktop/lucitra-signing.p12` — the certificate is now in GitHub Secrets and shouldn't stay on disk unencrypted.

## Required Secrets Summary

| Secret | Source | Auto? |
|--------|--------|-------|
| `APPLE_CERTIFICATE` | base64 of exported .p12 | Auto (after manual export) |
| `APPLE_CERTIFICATE_PASSWORD` | User sets during .p12 export | User provides |
| `APPLE_SIGNING_IDENTITY` | `security find-identity` | Auto |
| `APPLE_ID` | User's Apple Developer email | User provides |
| `APPLE_PASSWORD` | App-specific password from appleid.apple.com | User provides |
| `APPLE_TEAM_ID` | Extracted from signing identity | Auto |
| `TAURI_SIGNING_PRIVATE_KEY` | `pnpm tauri signer generate` | Auto (if missing) |
| `TAURI_SIGNING_PRIVATE_KEY_PASSWORD` | Generated with key | Auto (if missing) |

## Version Files Reference

All four files must have matching versions for a clean release:

| File | Field | Format |
|------|-------|--------|
| `apps/studio/src-tauri/tauri.conf.json` | `"version"` | `"0.2.0"` |
| `apps/studio/package.json` | `"version"` | `"0.2.0"` |
| `apps/studio/src-tauri/Cargo.toml` | `version` | `"0.2.0"` |
| `package.json` (root) | `"version"` | `"0.2.0"` |

## Troubleshooting

| Issue | Fix |
|-------|-----|
| No "Developer ID Application" cert | Need Apple Developer Program membership ($99/yr). Enroll at developer.apple.com |
| `security find-identity` shows 0 identities | Create cert in Xcode → Settings → Accounts → Manage Certificates |
| Release build fails with "certificate not found" | The .p12 may not include the private key. Re-export from Keychain with the key |
| Notarization fails with "invalid credentials" | App-specific password may have expired. Generate a new one at appleid.apple.com |
| Release build fails on universal binary | Add `--target universal-apple-darwin` to tauri-action args |
| v0.1.1 draft stuck | Delete the draft: `gh release delete v0.1.1 -R $REPO --yes` then re-tag |
| Updater not finding release | Check `latest.json` exists in release assets and release is not Draft |
| Workflow not triggered | Tags must match `v*.*.*` pattern. Check: `git tag -l 'v*'` |
