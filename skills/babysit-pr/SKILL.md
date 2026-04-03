---
name: babysit-pr
description: "Autonomously shepherd a PR to merge-readiness: request Copilot review, wait for it, fix any comments, resolve code scanning findings, verify CI passes, and loop until clean. Use when user says 'babysit this PR', 'get PR ready to merge', 'land this', 'watch PR #N', or after /ship creates a PR."
user_invocable: true
argument-hint: "<PR-number> [owner/repo]"
---

# Babysit PR: Autonomous PR Shepherding

Monitor a PR through external review (GitHub Copilot) and CI, automatically fixing issues until the PR is merge-ready.

## Arguments

Parse `$ARGUMENTS` for:
- **PR number** (required): e.g., `31` or `#31`
- **owner/repo** (optional): defaults to current git remote

## Babysit Checklist

- [ ] Pre-flight: uncommitted changes staged, lockfile synced
- [ ] Copilot review requested (or skipped if unavailable)
- [ ] Copilot comments resolved (fixed + replied + threads resolved)
- [ ] Code scanning comments resolved (fixed new findings, noted pre-existing)
- [ ] CI passing (with pre-existing failures noted)
- [ ] Slack notification sent (once, at completion)

## Overview

```
Pre-flight (stage changes, sync lockfile)
        ↓
Request Copilot Review
        ↓
  Wait for review
        ↓
  Comments found? ──yes──→ /resolve-review
        │                        ↓
        no                  Push fixes
        ↓                        ↓
  Code scanning comments? ──yes──→ Fix new findings, note pre-existing
        │                              ↓
        no                        Push fixes
        ↓                              ↓
  CI passing? ←────────────────────────┘
        │
       yes──→ Report: "PR ready for merge"
        │
        no──→ Diagnose + fix CI failures → loop
```

## Step 0: Pre-flight

Before anything else, ensure the working tree is clean and dependencies are in sync.

### Stage uncommitted changes

```bash
git status --short
```

If there are unstaged or untracked files that would affect CI (e.g., modified `.ts`, `.tsx`, `package.json`, `pnpm-lock.yaml`, `package-lock.json`, or other source/config files), stage and commit them ALL in a single commit:

```bash
git add -A
git commit --no-verify -m "chore: stage uncommitted changes for CI

part of LUC-XXX

Co-Authored-By: Claude <noreply@anthropic.com>"
```

### Sync lockfile

Run `pnpm install --frozen-lockfile` (or `npm ci`) to catch lockfile mismatches BEFORE pushing:

```bash
pnpm install --frozen-lockfile 2>&1
```

If it fails due to lockfile mismatch, regenerate and include it:

```bash
pnpm install
git add pnpm-lock.yaml
git commit --no-verify -m "chore: sync lockfile

part of LUC-XXX

Co-Authored-By: Claude <noreply@anthropic.com>"
```

Push all pre-flight changes before proceeding:

```bash
git push
```

## Step 1: Validate PR State

```bash
gh pr view {pr} --json state,mergeable,title,headRefName,statusCheckRollup \
  --jq '{state, mergeable, title, branch: .headRefName}'
```

- If PR is not OPEN, abort: "PR is {state}"
- Checkout the PR branch if not already on it

## Step 2: Request Copilot Review

Request GitHub Copilot as a reviewer:

```bash
gh pr edit {pr} --add-reviewer "@me/copilot"
```

If that fails (Copilot not available), try:
```bash
gh api repos/{owner}/{repo}/pulls/{pr}/requested_reviewers \
  -f "reviewers[]=copilot-pull-request-reviewer" 2>/dev/null || true
```

If Copilot review is not available for this repo, skip to Step 4 (CI check).

## Step 3: Wait for Copilot Review

Poll for review completion (max 10 attempts, 30s intervals):

```bash
gh api repos/{owner}/{repo}/pulls/{pr}/reviews \
  --jq '.[] | select(.user.login | test("copilot|github-actions")) | {state, body, submitted_at}'
```

**While no Copilot review exists:** Wait 30 seconds and re-poll.

**When review arrives:**
- If `APPROVED`: Skip to Step 4
- If `COMMENTED` or `CHANGES_REQUESTED`: Continue to Step 3b

### Step 3b: Resolve Copilot Comments

Run the `/resolve-review` workflow:

1. Fetch all unresolved comments on this PR
2. For each comment:
   - Read the referenced file
   - Categorize: actionable / acknowledged / false-positive
   - Fix actionable issues
3. Batch ALL fixes into a single commit (never multiple commits per cycle). Always use `--no-verify` during babysit — the CI pipeline itself is the verification:
   ```
   fix: address Copilot review feedback

   part of LUC-XXX

   Co-Authored-By: Claude <noreply@anthropic.com>
   ```
4. Push
5. Reply to every comment with what was done
6. Resolve all threads via GraphQL

After resolving, wait 30s for CI to re-trigger, then continue to Step 3c.

## Step 3c: Resolve Code Scanning Comments

After Copilot review (or if Copilot is unavailable), check for `github-advanced-security[bot]` code scanning comments. These are CodeQL / GHAS findings that appear as PR review comments.

### Fetch code scanning comments

```bash
gh api repos/{owner}/{repo}/pulls/{pr}/comments \
  --jq '.[] | select(.user.login == "github-advanced-security[bot]") | {id, path: .path, line: .line, body: .body[:150]}'
```

### Classify each finding

For each comment, determine if it's on files changed in this PR or pre-existing:

```bash
# Get the list of files changed in this PR
CHANGED_FILES=$(gh pr diff {pr} --name-only)
```

- **New finding on changed file**: Fix the code issue (same as Copilot comments)
- **Pre-existing finding on unchanged file**: Reply "Pre-existing finding — not introduced in this PR. Tracked for separate fix." and move on
- **False positive / by-design**: Reply with justification (e.g., "By design — OAuth protocol requires code as query parameter")

### Fix and reply

1. Fix actionable findings on our code
2. Batch ALL fixes into a single commit
3. Reply to every comment explaining what was done
4. Push

### Resolve code scanning threads

Code scanning threads from `github-advanced-security[bot]` **cannot be resolved** via the `resolveReviewThread` GraphQL mutation or the code-scanning dismiss API. GitHub returns "The thread is not a conversation and cannot be resolved". These threads can only be resolved through the GitHub web UI or auto-close when the PR merges.

**Workaround:** Dismiss the underlying alerts via the code scanning API to prevent them from re-appearing on future PRs:

```bash
# Dismiss pre-existing code scanning alerts
gh api -X PATCH "repos/{owner}/{repo}/code-scanning/alerts/{alert_number}" \
  -f state=dismissed \
  -f dismissed_reason="won't fix" \
  -f dismissed_comment="Pre-existing finding — tracked for separate fix."
```

This does NOT resolve the PR threads, but prevents alert noise. Reply to each thread and note them in the babysit report as unresolvable via API.

After addressing code scanning comments, continue to Step 4.

## Step 4: Verify CI

### Establish baseline (pre-existing failure detection)

Before checking CI on this PR, get the base branch's CI status to identify pre-existing failures:

```bash
# Get the merge base commit
MERGE_BASE=$(gh pr view {pr} --json baseRefName --jq '.baseRefName')
BASE_SHA=$(git merge-base HEAD origin/$MERGE_BASE)

# Check if any failing checks on this PR were also failing on the base branch
gh api "repos/{owner}/{repo}/commits/$BASE_SHA/check-runs" \
  --jq '.check_runs[] | select(.conclusion == "failure") | .name'
```

Any check that was already failing on the base branch is tagged as **pre-existing** and does NOT count as a blocker. In reports and Slack notifications, clearly distinguish these:
`5/6 passing | 1 pre-existing failure (SonarCloud)`

### Check status

```bash
gh pr checks {pr} --watch --fail-fast 2>&1
```

If `--watch` is not available, poll manually:

```bash
gh pr checks {pr}
```

### Handle CI Failures

For each failing check, first cross-reference against the baseline. If the check was already failing on the base branch, mark it as pre-existing and skip. Otherwise, diagnose:

| Check | Typical Fix |
|-------|-------------|
| **test-and-lint** | Read failure logs, fix test/lint errors |
| **SonarCloud** | Usually quality gate — check if pre-existing |
| **CodeQL / Analyze** | Security findings — fix if actionable |
| **Cloud Build** | Build errors — read logs and fix |
| **check-desktop-app** | Tauri build — check Rust/cargo issues |

```bash
# Get failure logs for a specific job
gh api "repos/{owner}/{repo}/actions/jobs/{job_id}/logs" 2>/dev/null | tail -50
```

Fix failures, batch ALL fixes into a single commit per cycle with `--no-verify`, push, and re-poll. **Max 3 fix-and-retry cycles** to avoid infinite loops.

```bash
git add -A
git commit --no-verify -m "fix: resolve CI failures (cycle N/3)

part of LUC-XXX

Co-Authored-By: Claude <noreply@anthropic.com>"
git push
```

If a check is consistently failing and was identified as pre-existing in the baseline check, note it and move on.

## Step 5: Final Verification

Once Copilot review is resolved and CI is green (or only pre-existing failures remain):

```bash
gh pr view {pr} --json reviewDecision,statusCheckRollup,mergeable \
  --jq '{reviewDecision, mergeable, checks: [.statusCheckRollup[] | {name: .name, status: .status, conclusion: .conclusion}]}'
```

## Step 6: Notify Slack

Send exactly **ONE** notification at the end of the babysit workflow. Do NOT send intermediate notifications during fix cycles.

- **Success**: Rich "ready to merge" notification with per-check CI status (see below)
- **Failure** (max retries exhausted): Send a failure notification instead:
  ```
  :x: PR #{pr} needs attention
  {what failed and what was tried}
  ```

If the babysit hits max retries and gives up, send the failure variant and include which checks/issues remain unresolved.

### 6a: Gather PR metadata and CI status

```bash
# PR metadata
PR_DATA=$(gh pr view {pr} --json url,title,additions,deletions,changedFiles,commits,headRefName \
  --jq '{url, title, additions, deletions, files: .changedFiles, commits: (.commits | length), branch: .headRefName}')
PR_URL=$(echo "$PR_DATA" | jq -r '.url')
PR_TITLE=$(echo "$PR_DATA" | jq -r '.title')
ADDITIONS=$(echo "$PR_DATA" | jq -r '.additions')
DELETIONS=$(echo "$PR_DATA" | jq -r '.deletions')
FILES=$(echo "$PR_DATA" | jq -r '.files')
BRANCH=$(echo "$PR_DATA" | jq -r '.branch')

# Per-check CI status — build a line per check with emoji + name + duration
CI_LINES=$(gh pr checks {pr} 2>&1 | while IFS=$'\t' read -r name status duration url; do
  case "$status" in
    pass)    echo ":white_check_mark: ${name} (${duration})" ;;
    fail)    echo ":x: ${name} (${duration})" ;;
    skipping) ;; # omit skipped checks
    pending) echo ":hourglass_flowing_sand: ${name}" ;;
    *)       echo ":grey_question: ${name} — ${status}" ;;
  esac
done)

# Review summary — count resolved threads and how they were handled
REVIEW_SUMMARY="{N}/{N} comments resolved ({X} fixed, {Y} acknowledged)"
# Fill these from the resolve-review step's actual counts
```

### 6b: Find webhook URL

```bash
WEBHOOK_URL=$(cat .claude/hooks/notify-slack.sh 2>/dev/null | grep 'WEBHOOK_URL=' | head -1 | cut -d'"' -f2)
# Fallback: check parent repo (for submodule setups like lucitra-dev/lucitra-studio)
if [ -z "$WEBHOOK_URL" ]; then
  PARENT_ROOT=$(cd .. && git rev-parse --show-toplevel 2>/dev/null)
  if [ -n "$PARENT_ROOT" ]; then
    WEBHOOK_URL=$(cat "$PARENT_ROOT/.claude/hooks/notify-slack.sh" 2>/dev/null | grep 'WEBHOOK_URL=' | head -1 | cut -d'"' -f2)
  fi
fi
```

### 6c: Build and send notification

Build the Slack message with these blocks:
1. **Header**: `:white_check_mark: PR #{pr} ready for merge` (or `:warning:` if pre-existing failures)
2. **Title section**: PR title as link
3. **CI Status section**: One line per check with emoji status and duration. Include ALL non-skipped checks. Format:
   - `:white_check_mark: check-name (duration)` for passing
   - `:x: check-name (duration)` for failing
   - `:hourglass_flowing_sand: check-name` for pending
   - If a failing check is pre-existing, append `— pre-existing`
4. **Review section**: Copilot review summary (N fixed, N acknowledged) if review happened
5. **Stats context**: Branch, changes (+N/-N), files changed
6. **Action buttons**: "Merge PR" (primary, links to `{pr_url}/merge`) and "View PR"
7. **Footer context**: `_Babysit complete — tap Merge to ship_`

```bash
if [ -n "$WEBHOOK_URL" ]; then
  MERGE_URL="${PR_URL}/merge"

  # Use python3 to safely build JSON with all the gathered data.
  # Pass CI_LINES, REVIEW_SUMMARY, and PR metadata as env vars.
  # The python script should:
  # 1. Split CI_LINES on newlines into individual status lines
  # 2. Join them with \n for the mrkdwn text block
  # 3. Build the full blocks array as shown above

  curl -s -X POST "$WEBHOOK_URL" \
    -H 'Content-Type: application/json' \
    -d "$(CI_LINES="$CI_LINES" REVIEW_SUMMARY="$REVIEW_SUMMARY" \
      PR_URL="$PR_URL" MERGE_URL="$MERGE_URL" PR_TITLE="$PR_TITLE" \
      ADDITIONS="$ADDITIONS" DELETIONS="$DELETIONS" FILES="$FILES" BRANCH="$BRANCH" \
      python3 -c "
import json, os

pr_url = os.environ.get('PR_URL', '')
merge_url = os.environ.get('MERGE_URL', '')
pr_title = os.environ.get('PR_TITLE', '')
additions = os.environ.get('ADDITIONS', '0')
deletions = os.environ.get('DELETIONS', '0')
files = os.environ.get('FILES', '0')
branch = os.environ.get('BRANCH', '')
ci_lines = os.environ.get('CI_LINES', '').strip()
review_summary = os.environ.get('REVIEW_SUMMARY', '')

has_failure = ':x:' in ci_lines
header_emoji = ':warning:' if has_failure else ':white_check_mark:'

blocks = [
    {'type': 'section', 'text': {'type': 'mrkdwn', 'text': f'{header_emoji} *PR ready for merge*\n*<{pr_url}|{pr_title}>*'}},
    {'type': 'section', 'text': {'type': 'mrkdwn', 'text': f'*CI Status:*\n{ci_lines}'}},
]

if review_summary:
    blocks.append({'type': 'section', 'text': {'type': 'mrkdwn', 'text': f'*Review:* {review_summary}'}})

blocks.append({'type': 'actions', 'elements': [
    {'type': 'button', 'text': {'type': 'plain_text', 'text': 'Merge PR'}, 'url': merge_url, 'style': 'primary'},
    {'type': 'button', 'text': {'type': 'plain_text', 'text': 'View PR'}, 'url': pr_url},
]})

blocks.append({'type': 'context', 'elements': [
    {'type': 'mrkdwn', 'text': f'\`{branch}\` | +{additions} / -{deletions} across {files} files | _Babysit complete — tap Merge to ship_'}
]})

print(json.dumps({'blocks': blocks, 'text': f'PR ready for merge: {pr_title}'}))
" 2>/dev/null)" > /dev/null 2>&1
fi
```

The "Merge PR" button opens GitHub's merge page directly — one tap to merge from phone.

### 6d: Failure notification (when max retries exhausted)

If the babysit gave up after max fix cycles, send a failure notification instead of the success one:

```bash
if [ -n "$WEBHOOK_URL" ]; then
  FAILURE_DETAIL="Tried $CYCLE_COUNT fix cycles. Remaining issues: $REMAINING_ISSUES"

  curl -s -X POST "$WEBHOOK_URL" \
    -H 'Content-Type: application/json' \
    -d "$(PR_URL="$PR_URL" PR_TITLE="$PR_TITLE" FAILURE_DETAIL="$FAILURE_DETAIL" \
      CI_LINES="$CI_LINES" BRANCH="$BRANCH" \
      python3 -c "
import json, os

pr_url = os.environ.get('PR_URL', '')
pr_title = os.environ.get('PR_TITLE', '')
failure_detail = os.environ.get('FAILURE_DETAIL', '')
ci_lines = os.environ.get('CI_LINES', '').strip()
branch = os.environ.get('BRANCH', '')

blocks = [
    {'type': 'section', 'text': {'type': 'mrkdwn', 'text': f':x: *PR needs attention*\n*<{pr_url}|{pr_title}>*'}},
    {'type': 'section', 'text': {'type': 'mrkdwn', 'text': f'*What happened:*\n{failure_detail}'}},
    {'type': 'section', 'text': {'type': 'mrkdwn', 'text': f'*CI Status:*\n{ci_lines}'}},
    {'type': 'actions', 'elements': [
        {'type': 'button', 'text': {'type': 'plain_text', 'text': 'View PR'}, 'url': pr_url},
    ]},
    {'type': 'context', 'elements': [
        {'type': 'mrkdwn', 'text': f'\`{branch}\` | _Babysit gave up — manual intervention needed_'}
    ]},
]

print(json.dumps({'blocks': blocks, 'text': f'PR needs attention: {pr_title}'}))
" 2>/dev/null)" > /dev/null 2>&1
fi
```

If no webhook is configured, skip silently.

## Step 7: Report

Output a summary to the conversation:

```
## PR #{pr} — Ready for Merge

**Copilot Review:** {N comments resolved (X fixed, Y acknowledged, Z false-positive)}
**Code Scanning:** {N findings addressed (X fixed, Y pre-existing, Z by-design)}
**CI Status:** {N/M checks passing}
**Pre-existing failures:** {list any, or "None"}

→ Merge with: `gh pr merge {pr} --squash --delete-branch`
→ After merge: `/post-merge {pr}` to verify deployment + update submodules
```

IMPORTANT: Do NOT merge the PR automatically. Always leave the final merge to the human.

## Guardrails

- **Max 3 fix cycles**: If issues keep appearing after 3 rounds of fixes, stop and report
- **Never force-push**: Only regular `git push`
- **Never merge**: Report readiness, let the human merge
- **Pre-existing failures**: If a check was failing before this PR, note it but don't block
- **Timeout**: If waiting >10 minutes for Copilot review with no response, skip and move to CI check

## Common Issues

### Copilot review not available
Not all repos have Copilot code review enabled. If the reviewer request fails, skip to CI verification — the workflow still adds value by monitoring and fixing CI failures.

### Rate limiting on gh api
If you get 403/429 errors, increase poll intervals to 60s. Don't retry more than 10 times total.

### Circular fixes
If fixing one Copilot comment introduces a new one, prioritize the original fix. After 2 cycles on the same file, reply acknowledging the tradeoff and move on.
