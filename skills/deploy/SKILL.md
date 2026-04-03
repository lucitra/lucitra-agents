---
name: deploy
description: "Deploy a service to dev or prod with pre-flight validation, git push to trigger Cloud Build, and post-deploy monitoring. Use when user says 'deploy', 'push to dev', 'ship to production', or 'release to staging'."
disable-model-invocation: true
argument-hint: "<service> <dev|prod>"
---

# Deploy a Service

Pre-flight checks, push to trigger Cloud Build, and monitor.

## Arguments

Parse `$ARGUMENTS` as `[service] [dev|prod]` (e.g., `marketing dev`, `validate prod`). Default environment is `dev`.

## Steps

1. **Parse arguments**: Extract service name and target environment (default: `dev`).

2. **Pre-flight checks**:
   - Verify on correct branch (`dev` for dev deploys, `main` for prod)
   - Run `git status` — ensure working tree is clean
   - Run validation: `make validate-{service}` or `npm run ci:validate`
   - Check `cloudbuild.yaml` exists in the service directory

3. **Load deployment context**: Read `.claude/context/deployment.md` for branching strategy.

4. **For dev deployment**:
   ```bash
   cd {service-directory}
   git push origin dev
   ```

5. **For prod deployment**:
   - Confirm with user: "This will deploy to production. Are you sure?"
   - Verify changes have been tested on dev first
   ```bash
   cd {service-directory}
   git push origin main
   ```

6. **Monitor**: Check Cloud Build status after push.
   ```bash
   gcloud builds list --limit=5 --filter="source.repoSource.repoName={service}"
   ```

7. **Infrastructure sync reminder**: If `cloudbuild.yaml` was modified, remind to update Terraform in `lucitra-infrastructure/`.

## Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| `fatal: not on branch` | Detached HEAD state | Run `git checkout dev` or `git checkout main` |
| Cloud Build fails after push | Build config or dependency issue | Run `gcloud builds log <build-id>` to see full logs |
| `ERROR: (gcloud.builds.list) PERMISSION_DENIED` | Missing GCP auth | Run `gcloud auth login` and `gcloud config set project lucitra-prod` |
| Validation passes locally but fails in Cloud Build | Environment differences | Check Docker base image version matches `cloudbuild.yaml` |
| Push rejected | Branch protection or stale ref | Pull latest with `git pull --rebase origin {branch}` |
