---
name: resolve-review
description: "Fetch and resolve PR review comments by fixing code issues, replying to every comment, and resolving threads via GitHub API. Use when user says 'resolve review comments', 'fix PR feedback', 'address review', or 'resolve PR #N'. Works with GitHub code scanning alerts and human review comments."
user_invocable: true
argument-hint: "<PR-number> [owner/repo]"
---

# Resolve PR Review Comments

Fetch unresolved review comments on a GitHub PR, fix actionable issues, reply to every comment, and resolve all threads.

## Arguments

Parse `$ARGUMENTS` for:
- **PR number** (required): e.g., `31` or `#31`
- **owner/repo** (optional): e.g., `lucitra/lucitra-studio`. Default: detect from current git remote.

## Step 1: Identify the PR

```bash
# If owner/repo not provided, detect from git remote
gh api repos/{owner}/{repo}/pulls/{pr} --jq '{title, head: .head.ref, state}'
```

Checkout the PR branch if not already on it:
```bash
git checkout {branch}
git pull origin {branch}
```

## Step 2: Fetch Unresolved Comments

Fetch all PR review comments:
```bash
gh api repos/{owner}/{repo}/pulls/{pr}/comments \
  --jq '.[] | {id: .id, path: .path, line: (.line // .original_line), body: .body, in_reply_to_id: .in_reply_to_id}'
```

Fetch thread resolution status via GraphQL:
```bash
gh api graphql -f query='{
  repository(owner: "{owner}", name: "{repo}") {
    pullRequest(number: {pr}) {
      reviewThreads(first: 50) {
        nodes {
          id
          isResolved
          comments(first: 1) {
            nodes { databaseId body }
          }
        }
      }
    }
  }
}'
```

Filter to only **unresolved** threads. If all threads are resolved, report "No unresolved comments" and stop.

## Step 3: Categorize Each Comment

For each unresolved comment, read the referenced file and categorize:

| Category | Criteria | Action |
|----------|----------|--------|
| **Actionable** | Real code issue (bug, security, missing permissions, etc.) | Fix it |
| **Acknowledged** | Valid concern but intentional design choice | Reply explaining why |
| **False positive** | Doesn't apply, already fixed, or scanner error | Reply explaining |

Read `.github/copilot-instructions.md` if it exists — code scanning comments may flag things that are project conventions.

## Step 4: Fix Actionable Issues

- Read each referenced file at the flagged line
- Make the minimal fix that addresses the comment
- Group all fixes into a **single commit**

> **Batch ALL fixes into a single commit — do not create separate commits per comment.** Use `--no-verify` for the commit since CI will validate after push.

CRITICAL: Follow project commit conventions:
```bash
git add {files}
git commit -m "fix: address PR review comments

part of LUC-XXX

Co-Authored-By: Claude <noreply@anthropic.com>"
git push
```

## Step 5: Reply to Every Comment

Reply to **every** comment, even false positives. Use the comment's `id` as `in_reply_to`.

> For comments with identical fixes (e.g., multiple "add aria-label" comments), you can use a single reply pattern but **MUST** reply to each comment individually. Run replies in parallel using multiple `gh api` calls in a single bash command when possible.

```bash
# For fixed issues:
gh api repos/{owner}/{repo}/pulls/{pr}/comments \
  -f body="Fixed in {commit_sha} — {brief description of fix}" \
  -F in_reply_to={comment_id}

# For acknowledged/false-positive:
gh api repos/{owner}/{repo}/pulls/{pr}/comments \
  -f body="{explanation}" \
  -F in_reply_to={comment_id}
```

## Step 6: Resolve All Threads

Resolve each thread via GraphQL mutation:
```bash
gh api graphql -f query='mutation {
  resolveReviewThread(input: {threadId: "{thread_id}"}) {
    thread { isResolved }
  }
}'
```

## Step 7: Verify and Report

Re-fetch threads to confirm all resolved:
```bash
gh api graphql -f query='{
  repository(owner: "{owner}", name: "{repo}") {
    pullRequest(number: {pr}) {
      reviewThreads(first: 50) {
        nodes { isResolved }
      }
    }
  }
}'
```

Report summary:
- N comments resolved (X fixed, Y acknowledged, Z false positives)
- Commit SHA if fixes were pushed
- Any comments that could not be resolved (and why)

## Common Issues

### "Resource not accessible by integration"
The `gh` CLI token may lack permissions. Run `gh auth status` and ensure the token has `repo` scope.

### GraphQL thread ID not found
Comment `databaseId` from REST API must be matched to GraphQL `reviewThreads` by comparing `comments.nodes[].databaseId`. If a comment has no thread (e.g., PR description comment), skip thread resolution.

### Code scanning alerts vs human comments
Code scanning alerts (from CodeQL, SonarCloud, etc.) appear as PR comments but may auto-resolve when the underlying issue is fixed. Fix the code first, push, then check if the alert auto-resolved before manually resolving.
