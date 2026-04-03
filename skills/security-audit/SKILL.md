# Security Audit

Systematically review branch protections, Dependabot, code scanning, secret scanning, and push protection across all Lucitra repositories. Target: 100% coverage.

## Usage

```
/security-audit              # Full audit of all repos
/security-audit --fix        # Audit and fix issues automatically
/security-audit <repo>       # Audit a specific repo
```

## Step 1: Inventory All Active Repos

```bash
gh repo list lucitra --limit 50 --json name,isArchived,visibility,defaultBranchRef \
  --jq '.[] | select(.isArchived == false) | {name, visibility, default: .defaultBranchRef.name}'
```

## Step 2: Branch Protection Audit

For each active repo, check both `dev` and `main` branches:

```bash
# Check if branch protection exists
gh api repos/lucitra/{repo}/branches/{branch}/protection --jq '{
  force_push: .allow_force_pushes.enabled,
  deletions: .allow_deletions.enabled,
  pr_required: (.required_pull_request_reviews != null),
  status_checks: (.required_status_checks != null)
}'
```

**Required protections:**
- [ ] No force pushes (`allow_force_pushes: false`)
- [ ] No deletions (`allow_deletions: false`)
- [ ] PRs required (even if 0 approvals)
- [ ] Status checks required (if CI exists for the repo)

**If `--fix`**: Apply missing protections:
```bash
gh api repos/lucitra/{repo}/branches/{branch}/protection --method PUT --input - <<EOF
{
  "required_status_checks": null,
  "enforce_admins": false,
  "required_pull_request_reviews": {"required_approving_review_count": 0},
  "restrictions": null,
  "allow_force_pushes": false,
  "allow_deletions": false
}
EOF
```

## Step 3: Dependabot Audit

Check if Dependabot alerts and security updates are enabled:

```bash
# Check Dependabot alerts
gh api repos/lucitra/{repo}/vulnerability-alerts --method GET 2>&1
# 204 = enabled, 404 = disabled

# Enable if disabled (--fix mode)
gh api repos/lucitra/{repo}/vulnerability-alerts --method PUT 2>&1

# Check Dependabot security updates (auto-fix PRs)
gh api repos/lucitra/{repo}/automated-security-fixes --method GET 2>&1
# 200 with enabled:true = active

# Enable security updates (--fix mode)
gh api repos/lucitra/{repo}/automated-security-fixes --method PUT

# Check if dependabot.yml exists
gh api repos/lucitra/{repo}/contents/.github/dependabot.yml 2>&1
```

**Note:** Dependabot security updates require vulnerability alerts to be enabled first. Enable alerts before enabling security updates. Repos with no package ecosystem (e.g., empty repos, pure documentation) will show security updates as "not supported" — this is expected.

## Step 4: Code Scanning Audit

Check if code scanning (CodeQL) is enabled:

```bash
# Check for code scanning alerts endpoint
gh api repos/lucitra/{repo}/code-scanning/alerts --jq 'length' 2>&1
# If 403/404: not enabled

# Check for CodeQL workflow
gh api repos/lucitra/{repo}/contents/.github/workflows --jq '.[].name' 2>&1 | grep -i codeql
```

**If `--fix`**: Enable code scanning in two steps:

**Step 4a: Enable code_security on private repos first**

Private repos require `code_security` (GitHub Advanced Security) enabled before code scanning can be configured. Without this, the `code-scanning/default-setup` API returns 403.

```bash
# Check current status
gh api repos/lucitra/{repo} --jq '.security_and_analysis.code_security.status'

# Enable code_security (required for private repos)
gh api repos/lucitra/{repo} --method PATCH --input - <<EOF
{
  "security_and_analysis": {
    "code_security": {"status": "enabled"}
  }
}
EOF
```

**Note:** The org-level setting `advanced_security_enabled_for_new_repositories` only applies to newly created repos. Existing private repos must have `code_security` enabled individually.

**Step 4b: Auto-detect languages and enable default setup**

```bash
# Detect languages in the repo
gh api repos/lucitra/{repo}/languages --jq 'keys'

# Map to CodeQL-supported languages:
#   JavaScript/TypeScript → "javascript-typescript"
#   Python → "python"
#   Ruby → "ruby"
#   Go → "go"
#   Java/Kotlin → "java-kotlin"
#   C/C++ → "c-cpp"
#   C# → "csharp"
#   Swift → "swift"
#   Rust → "actions" (via extended suite, not default)

# Enable with detected languages
gh api repos/lucitra/{repo}/code-scanning/default-setup --method PATCH \
  -f state=configured -f 'languages[]=javascript-typescript'
```

**Common errors:**
- `403 Advanced Security must be enabled` → Run Step 4a first
- `422 no_supported_languages` → Repo has no CodeQL-compatible code (e.g., pure Terraform, shell scripts)
- `409 already configured` → Already enabled, skip

## Step 5: Secret Scanning Audit

```bash
# Check secret scanning status
gh api repos/lucitra/{repo} --jq '{
  secret_scanning: .security_and_analysis.secret_scanning.status,
  push_protection: .security_and_analysis.secret_scanning_push_protection.status
}'

# Enable if disabled (--fix mode)
gh api repos/lucitra/{repo} --method PATCH --input - <<EOF
{
  "security_and_analysis": {
    "secret_scanning": {"status": "enabled"},
    "secret_scanning_push_protection": {"status": "enabled"}
  }
}
EOF
```

## Step 6: Report

Output a summary table:

```
Security Audit Report — Lucitra Organization
============================================

Branch Protection:
  ✅ lucitra-studio     dev: protected  main: protected
  ✅ lucitra-dev        dev: protected  main: protected
  ❌ lucitra-utils      main: NOT PROTECTED
  ...

Dependabot:          26/35 enabled (74%)  → target: 35/35
Code Scanning:       22/35 enabled (62%)  → target: 35/35
Secret Scanning:     22/35 enabled (62%)  → target: 35/35
Push Protection:     22/35 enabled (62%)  → target: 35/35

Issues Found: N
Auto-fixed (--fix): M
Manual Action Required: K
```

## Step 7: Fix Remaining Issues (--fix mode)

For each repo missing coverage, apply fixes in this order (dependencies matter):

1. Enable Dependabot alerts (`vulnerability-alerts --method PUT`)
2. Enable Dependabot security updates (`automated-security-fixes --method PUT`)
3. Enable secret scanning + push protection
4. Enable `code_security` on private repos (required before code scanning)
5. Enable code scanning default setup with auto-detected languages
6. Add branch protection if missing

**Important:** Do NOT delete or disable existing branch protections. If protections need updating, use PUT to overwrite with the correct settings. See memory: "Never disable branch protection".

## Scheduling

Run via `/loop 7d /security-audit` for weekly automated checks.
Or use `/schedule` to set up a cron trigger.
