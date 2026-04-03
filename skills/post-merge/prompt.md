# Post-Merge: Release Babysitting

After a PR is merged to dev, shepherd the changes through CI/CD, update submodule pointers, close Linear issues, and verify the release.

## Arguments

Parse for:
- **PR number** (optional): e.g., `54` — if provided, uses the PR to identify what was merged
- **service** (optional): e.g., `studio`, `marketing`, `platform-api` — which service was changed

If no arguments, detect from the most recent merge commit on dev.

## Step 1: Identify What Merged

```bash
# Get the latest merge or squash commit on dev
git log --oneline -5 dev
```

From the commit message, extract:
- Linear issue references (LUC-XXX)
- Service affected (from file paths)
- PR number (from commit message or `gh pr list --state merged`)

## Step 2: Update Submodule Pointer

If the change was in a submodule (lucitra-studio, lucitra-marketing, etc.):

```bash
cd /Users/ibraheem/Projects/lucitra-dev
git add {submodule}
git commit -m "chore: update {submodule} submodule — {summary}

{linear references}

Co-Authored-By: Claude <noreply@anthropic.com>"
git push origin dev
```

Skip if already done.

## Step 3: Monitor CI/CD

### For Cloud Run services (marketing, platform-api, mcp-server):
```bash
# Check Cloud Build status
gcloud builds list --filter="source.repoSource.branchName=dev" --limit=3 --format="table(id,status,startTime,source.repoSource.repoName)"
```

Poll until build completes. If failed, read logs:
```bash
gcloud builds log {build_id}
```

### For Studio (Tauri desktop app):
No auto-deploy — the build is local. Note this in the report.

### For npm packages:
Check if a version bump + publish is needed:
```bash
# Check if package.json version changed
git diff HEAD~1 -- packages/*/package.json
```

If version changed, suggest running `/npm-publish`.

## Step 4: Verify Deployment

### For Cloud Run services:
```bash
# Check service status
gcloud run services describe {service} --region=us-central1 --format="value(status.url,status.conditions[0].status)"

# Quick smoke test
curl -s -o /dev/null -w "%{http_code}" {service_url}/health
```

### For Studio:
Suggest running the app locally with `pnpm tauri:dev` and using `/qa` for visual testing.

## Step 5: Close Linear Issues

For each Linear issue referenced with closing keywords (closes, fixes):
```bash
# Verify issue status — magic words should auto-close on merge
# If not closed, check Linear webhook integration
```

For issues with `part of` references, leave them open (they span multiple PRs).

## Step 6: Clean Up

- Delete the merged feature branch (if not auto-deleted):
  ```bash
  git push origin --delete {branch}
  ```
- Clean up local worktrees for the branch:
  ```bash
  git worktree list | grep {branch}
  git worktree remove {path}
  ```
- Remove orphaned worktrees:
  ```bash
  git worktree prune
  ```

## Step 7: Report

Output summary:
- What merged (PR, issues, summary)
- Submodule pointer updated?
- CI/CD status
- Deployment verified?
- Issues closed?
- Branches cleaned up?
- Next steps (if any)

## Integration with /ship

The `/ship` skill should mention `/post-merge` in its output:
> "After human merges, run `/post-merge` to babysit the release."

The `/babysit-pr` Slack notification should include:
> "After merge, run `/post-merge 54` to verify deployment."

## Service Map

| Service | Deploy Target | Build System | Health Check |
|---------|--------------|--------------|--------------|
| lucitra-studio | Local desktop | Tauri CLI | `/qa` visual |
| lucitra-marketing | Cloud Run (prod) | Cloud Build | `lucitra.ai/health` |
| lucitra-platform-api | Cloud Run (dev/prod) | Cloud Build | `api-dev.lucitra.ai/health` |
| lucitra-mcp-server | Cloud Run (dev/prod) | Cloud Build | `mcp-dev.lucitra.ai/health` |
| packages/* | npm registry | pnpm build | `npm view @lucitra/{pkg}` |
