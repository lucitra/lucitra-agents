---
name: ship
description: "Fully automated ship workflow: validate, commit, push, and open a PR to dev. Use when user says 'ship it', 'create a PR', 'open a pull request', 'send for review', or is done with a feature branch and wants to merge."
argument-hint: "[LUC-XXX] [--skip-validate]"
---

# Ship: Automated PR Workflow

You are running the `/ship` workflow. This is a **non-interactive, fully automated** workflow. Run straight through and output the PR URL at the end.

**Only stop for:**
- On `dev` or `main` branch (abort — ship from a feature branch)
- No Linear issue reference found (ask for one)
- Merge conflicts that can't be auto-resolved
- Validation failures (`make validate-*`)
- Pre-landing review finds CRITICAL issues and user chooses to fix

**Never stop for:**
- Uncommitted changes (always include them)
- Commit message approval (auto-compose from diff)
- PR description content (auto-generate)

## Arguments

Parse `$ARGUMENTS`:
- **LUC-XXX**: Linear issue reference (required — if not provided, check branch name for `luc-NNN` pattern)
- **--skip-validate**: Skip `make validate-*` step (for non-code changes like docs)

---

## Step 1: Pre-flight

1. Run `git branch --show-current`. If on `dev` or `main`:
   - If there are uncommitted changes to ship, **enter a worktree** first:
     ```
     EnterWorktree(name: "ship-{short-description}")
     ```
     Then create a feature branch inside it: `git checkout -b founder/luc-{number}-{description} origin/dev`
   - If no changes to ship, **abort**: "Nothing to ship from `{branch}`."

2. Extract Linear issue from args or branch name:
   ```bash
   git branch --show-current | grep -oiE 'luc-[0-9]+'
   ```
   If no issue found and none in args, **stop** and ask.

3. Run `git status` (never use `-uall`). Note uncommitted changes — they'll be included.

4. **Lockfile sync check:** Run `pnpm install --frozen-lockfile` to verify `pnpm-lock.yaml` is in sync with `package.json`. If it fails, run `pnpm install` to update the lockfile, then include `pnpm-lock.yaml` in the commit. This prevents CI failures from lockfile mismatches.

5. Run `git log dev..HEAD --oneline` and `git diff dev...HEAD --stat` to understand what's being shipped.

---

## Step 2: Merge origin/dev

Fetch and merge `origin/dev` into the feature branch so validation runs against the merged state:

```bash
git fetch origin dev && git merge origin/dev --no-edit
```

**If merge conflicts:** Try auto-resolve for simple cases (package-lock.json, submodule pointers). For complex conflicts, **STOP** and show them.

**If already up to date:** Continue silently.

---

## Step 3: Validate (unless --skip-validate)

Detect which service(s) changed and run the appropriate validation:

```bash
git diff dev...HEAD --name-only
```

Map changed paths to validation targets:
- `lucitra-validate/` → `make validate-validate`
- `lucitra-studio/` → `make validate-studio`
- `lucitra-marketing/` → `make validate-marketing`
- `lucitra-platform-api/` → `make validate-platform-api`
- `lucitra-mcp-server/` → `make validate-mcp-server`
- `packages/` → `make validate-packages`
- Root-level only (skills, docs, config) → skip validation

Run from the `lucitra-dev` root. If multiple services changed, run each.

**If validation fails:** Show the failures and **STOP**. Do not proceed.

**If all pass:** Continue — note the results briefly.

---

## Step 4: Pre-Landing Review

Run a lightweight review of the diff for issues tests don't catch.

1. Run `git fetch origin dev --quiet && git diff origin/dev` to get the full diff.

2. **Pass 1 (CRITICAL)** — stop-ship issues:
   - Committed secrets (.env values, API keys, tokens, passwords)
   - `any` types in TypeScript
   - Terraform/cloudbuild sync violations (one changed without the other)
   - Missing Linear issue reference in code changes
   - `console.log` / `debugger` statements left in

3. **Pass 2 (INFORMATIONAL)** — note but don't block:
   - TODO/FIXME without issue references
   - Large files (>500KB) being committed
   - Yalc references in package.json
   - Dead imports or unused variables

4. **If CRITICAL issues found:** For EACH, use AskUserQuestion with:
   - Problem description (`file:line`)
   - Recommended fix
   - Options: A) Fix now, B) Acknowledge and ship, C) False positive
   If user chose A on any: apply fixes, re-stage, and continue.

5. **If only informational issues:** Output them and continue.

---

## Step 5: Stage and Commit

1. Stage all changes:
   ```bash
   git add -A
   ```

2. Compose commit message from the diff. Use conventional commit format:
   ```
   {type}: {concise summary}

   {optional body — what and why, not how}

   {closes|fixes|part of} LUC-XXX

   Co-Authored-By: Claude <noreply@anthropic.com>
   ```

   - `closes LUC-XXX` if this PR fully resolves the issue
   - `part of LUC-XXX` if it's partial progress
   - Infer type from changes: `feat` (new), `fix` (bug), `refactor`, `chore`, `docs`, `test`

3. For large changesets (>8 files, >300 lines), split into bisectable commits:
   - Infrastructure/config first
   - Core logic second
   - UI/frontend third
   - Each commit must be independently valid

---

## Step 6: Push

```bash
git push -u origin $(git branch --show-current)
```

---

## Step 7: Create PR

Create a pull request targeting `dev`:

```bash
gh pr create --base dev --title "{type}: {summary}" --body "$(cat <<'EOF'
## Summary
{bullet points — what changed and why}

## Linear
{closes|part of} LUC-XXX

## Pre-Landing Review
{findings from Step 4, or "No issues found."}

## Validation
{which make targets ran and their results}

## Test plan
- [ ] {key things to verify}

Co-Authored-By: Claude <noreply@anthropic.com>
EOF
)"
```

**Output the PR URL.**

---

## Step 8: Babysit to Merge-Readiness

After the PR is created, automatically shepherd it through external review and CI:

1. **Request Copilot review** on the PR
2. **Poll for review completion** (30s intervals, max 10 attempts)
3. **If comments found:** Fix actionable issues, reply to all comments, resolve threads, push
4. **Verify CI passes** — if checks fail, read logs and fix (max 3 fix cycles)
5. **Report final status**: "PR ready for merge" with summary of what was resolved

See `/babysit-pr` skill for the full workflow. This step runs the same logic inline.

IMPORTANT: Never merge automatically. The final output is:
- PR URL
- Copilot review summary (N comments resolved)
- CI status (N/M checks passing)
- `gh pr merge {pr} --squash --delete-branch` command for the human

---

## Important Rules

- **Never push to dev or main directly.** Always create a PR.
- **Never skip the pre-landing review.** It catches what validation misses.
- **Never force push.** Regular `git push` only.
- **Always include Linear reference.** No commits without LUC-XXX.
- **Never merge the PR.** Leave the final merge to the human.
- **The goal is: user says `/ship`, next thing they see is a merge-ready PR.**
