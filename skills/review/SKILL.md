---
name: review
description: "Pre-landing code review with 2-pass checklist (critical stop-ship issues, then informational). Use when user says 'review my code', 'check before merge', 'pre-landing review', or before shipping a PR. Also invoked automatically by /ship and /implement."
context: fork
argument-hint: "[LUC-XXX]"
---

# Pre-Landing Review

You are running the `/review` workflow. Analyze the current branch's diff against dev for structural issues that tests don't catch.

## Arguments

Parse `$ARGUMENTS` as an optional Linear issue ID (e.g., `LUC-301`) to include in commit message if fixes are made.

---

## Step 1: Check branch

1. Run `git branch --show-current` to get the current branch.
2. If on `dev` or `main`, output: **"Nothing to review — you're on dev."** and stop.
3. Run `git fetch origin dev --quiet && git diff origin/dev --stat` to check if there's a diff. If no diff, output the same message and stop.

---

## Step 2: Read the checklist

Read `.claude/skills/review/checklist.md`.

**If the file cannot be read, STOP and report the error.** Do not proceed without the checklist.

---

## Step 3: Get the diff

Fetch the latest dev to avoid false positives from a stale local:

```bash
git fetch origin dev --quiet
```

Run `git diff origin/dev` to get the full diff. This includes both committed and uncommitted changes.

Also run `git diff origin/dev --name-only` to get the list of changed files for targeted analysis.

---

## Step 4: Two-pass review

Apply the checklist against the diff in two passes:

### Pass 1 (CRITICAL) — stop-ship issues:
- Secrets & Credentials
- TypeScript Safety (`any` types)
- Infrastructure Sync (Terraform ↔ cloudbuild)
- Linear Compliance
- Security (injection vectors, leaked secrets)

### Pass 2 (INFORMATIONAL) — note but don't block:
- Code Quality (console.log, dead code, TODOs)
- Patterns (Docker-first, App Router, path aliases)
- File Hygiene (large files, generated files)
- Commit Standards

**Respect the suppressions** in the checklist — do NOT flag items listed in the "DO NOT flag" section.

---

## Step 5: Output findings

**Always output ALL findings** — both critical and informational.

Format:

```
Pre-Landing Review: N issues (X critical, Y informational)

## CRITICAL
1. [file:line] Description of issue
   Fix: recommended action

## INFORMATIONAL
1. [file:line] Description of issue
```

### If CRITICAL issues found:

For EACH critical issue, use AskUserQuestion with:
- The problem (`file:line` + description)
- Your recommended fix
- Options:
  - **A) Fix it now** (recommended)
  - **B) Acknowledge and ship anyway**
  - **C) False positive — skip**

After all critical questions are answered:
- If user chose **A** on any: apply the fixes, stage only the fixed files by name, and commit:
  ```bash
  git add <fixed-files>
  git commit -m "fix: apply pre-landing review fixes

  part of LUC-XXX

  Co-Authored-By: Claude <noreply@anthropic.com>"
  ```
- If only **B/C** chosen: no action needed.

### If only informational issues found:

Output findings. No further action needed.

### If no issues found:

Output: `Pre-Landing Review: No issues found.`

---

## Important Rules

- **Read the FULL diff before commenting.** Do not flag issues already addressed in the diff.
- **Read-only by default.** Only modify files if user explicitly chooses "Fix it now."
- **Be terse.** One line problem, one line fix. No preamble.
- **Only flag real problems.** Skip anything that's fine.
- **Never commit, push, or create PRs.** That's `/ship`'s job.
