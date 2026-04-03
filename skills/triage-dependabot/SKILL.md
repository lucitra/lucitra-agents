---
name: triage-dependabot
description: "Triage open Dependabot PRs across Lucitra repos: close superseded/breaking, merge safe bumps, rebase stale ones, and report status. Use when user says 'check dependabot', 'triage deps', 'merge security updates', or run via /loop for automated maintenance."
user_invocable: true
argument-hint: "[repo-name] [--dry-run]"
---

# Triage Dependabot PRs

Automatically triage open Dependabot PRs: close superseded ones, merge safe bumps, rebase stale ones, and send a Slack summary.

## Arguments

Parse `$ARGUMENTS`:
- **repo-name** (optional): specific repo to check (e.g., `lucitra-studio`). Default: check all Lucitra repos.
- **--dry-run**: report what would be done without taking action

## Repos to Check

```bash
REPOS="lucitra/lucitra-studio lucitra/lucitra-validate lucitra/lucitra-marketing lucitra/lucitra-platform-api lucitra/lucitra-mcp-server lucitra/lucitra-infrastructure"
```

If a specific repo was passed, filter to just that one.

## Step 1: Gather Open Dependabot PRs

For each repo:

```bash
gh pr list --repo {repo} --author "dependabot[bot]" --state open \
  --json number,title,mergeable,createdAt,headRefName,statusCheckRollup,labels \
  --jq '.[] | {number, title, mergeable, created: .createdAt, branch: .headRefName, labels: [.labels[].name], checks: [.statusCheckRollup[]? | {name: .name, conclusion: .conclusion}]}'
```

If no open PRs, skip that repo.

## Step 2: Categorize Each PR

For each PR, determine the action:

### Close (superseded)
- PR bumps a dependency to version X, but the repo already has version >= X (check the current file)
- PR has merge conflicts AND is a minor/patch bump that dependabot can re-create
- PR is a major version bump that requires manual migration (e.g., ESLint 8→9, React 18→19)

```bash
gh pr close {number} --repo {repo} --comment "Closed: {reason}. Dependabot will re-create if needed."
```

### Merge (safe)
- PR is mergeable (no conflicts)
- All CI checks pass (or only pre-existing failures)
- It's a patch/minor version bump (not major)
- It's a security update (has `security` label)

```bash
gh pr merge {number} --repo {repo} --squash --auto
```

### Rebase (stale)
- PR has merge conflicts from recent changes
- The underlying bump is still needed

```bash
gh pr comment {number} --repo {repo} --body "@dependabot rebase"
```

### Skip (needs review)
- Major version bumps that might be safe but need human judgment
- PRs with failing CI that isn't pre-existing
- Report these for human review

## Step 3: Auto-merge Safe PRs

For PRs categorized as "Merge":

1. Check if CI is passing:
   ```bash
   gh pr checks {number} --repo {repo}
   ```

2. If all green (or only pre-existing failures), enable auto-merge:
   ```bash
   gh pr merge {number} --repo {repo} --squash --auto --delete-branch
   ```

3. If CI is pending, enable auto-merge to trigger when checks pass:
   ```bash
   gh pr merge {number} --repo {repo} --squash --auto
   ```

## Step 4: Report via Slack

Send a summary to Slack:

```bash
WEBHOOK_URL=$(cat .claude/hooks/notify-slack.sh 2>/dev/null | grep 'WEBHOOK_URL=' | head -1 | cut -d'"' -f2)
if [ -z "$WEBHOOK_URL" ]; then
  PARENT_ROOT=$(cd .. && git rev-parse --show-toplevel 2>/dev/null)
  [ -n "$PARENT_ROOT" ] && WEBHOOK_URL=$(cat "$PARENT_ROOT/.claude/hooks/notify-slack.sh" 2>/dev/null | grep 'WEBHOOK_URL=' | head -1 | cut -d'"' -f2)
fi
```

Message format — each PR must be a clickable link using `<url|#number title>` Slack mrkdwn syntax:
```
:package: Dependabot Triage Summary

:white_check_mark: Merged:
• repo: <https://github.com/owner/repo/pull/N|#N title>

:no_entry_sign: Closed:
• repo: <https://github.com/owner/repo/pull/N|#N title> — {reason}

:arrows_counterclockwise: Rebased:
• repo: <https://github.com/owner/repo/pull/N|#N title>

:eyes: Needs review:
• repo: <https://github.com/owner/repo/pull/N|#N title> — {why}
```

Build PR URLs from `https://github.com/{owner}/{repo}/pull/{number}`. Every PR reference in the Slack message MUST be a link.

## Step 5: Output Summary

Report to conversation what was done.

## Guardrails

- **Never merge major version bumps** without human approval
- **Never merge if CI is failing** (unless pre-existing)
- **Always squash merge** dependabot PRs
- **Security updates get priority** — merge even if other checks are pending
- In `--dry-run` mode, only report — take no action
