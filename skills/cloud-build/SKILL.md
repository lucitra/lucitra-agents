# Cloud Build Monitor

Monitor Google Cloud Build status for Lucitra services. Check build progress, view logs, and diagnose failures.

## Arguments

Parse for:
- **service** (optional): Service name to filter (e.g., `marketing`, `mcp-server`, `validate`). Maps to repo name.
- **--logs [build-id]**: Show logs for a specific build
- **--status**: Quick status check of recent builds
- **--watch [build-id]**: Poll a build until completion

## Service Name Map

| Short Name | Repo Name | Cloud Run Service |
|-----------|-----------|-------------------|
| `marketing` | `lucitra-marketing` | `lucitra-marketing-dev` / `lucitra-marketing` |
| `validate` | `lucitra-validate` | `lucitra-validate-dev` / `lucitra-validate` |
| `mcp-server` | `lucitra-mcp-server` | `lucitra-mcp-server-dev` / `lucitra-mcp-server-prod` |
| `platform-api` | `lucitra-platform-api` | `lucitra-platform-api-dev` / `lucitra-platform-api` |
| `studio` | `lucitra-studio` | N/A (desktop app, uses GitHub Actions) |

## Quick Status (`--status` or default)

Use the gcloud MCP tool to list recent builds:

```
gcloud builds list --limit 5 --region us-central1 --format table(id,status,source.repoSource.repoName,startTime,duration)
```

If a service name is provided, filter:

```
gcloud builds list --limit 5 --region us-central1 --filter substitutions.REPO_NAME:lucitra-{service} --format table(id,status,startTime,duration,logUrl)
```

Report a summary table:
- Build ID (short — first 8 chars)
- Status (SUCCESS/FAILURE/WORKING/QUEUED)
- Service name
- Duration
- Time ago

## View Logs (`--logs`)

```
gcloud builds log {build-id} --region us-central1
```

If the build failed, focus on the last 50 lines. Look for:
- Docker build errors
- npm/pnpm install failures
- TypeScript compilation errors
- Cloud Run deployment errors

## Watch Build (`--watch`)

Poll every 15 seconds until the build completes:

```
gcloud builds describe {build-id} --region us-central1 --format json(status,startTime,finishTime,logUrl)
```

Report status transitions: QUEUED → WORKING → SUCCESS/FAILURE

When complete, report:
- Final status
- Duration
- If failed: show last 30 lines of logs

## Diagnosing Failures

When a build fails, automatically:
1. Fetch the logs: `gcloud builds log {build-id} --region us-central1`
2. Look for common patterns:
   - `npm ERR!` / `pnpm ERR!` → dependency issue
   - `error TS` → TypeScript error
   - `COPY failed` → Docker build context issue
   - `ERROR: deploy` → Cloud Run deploy failure
   - `Revision .* is not ready` → Cloud Run health check failure
3. Suggest fixes based on the pattern

## Cloud Run Service Status

After a successful deploy, verify the service is healthy:

```
gcloud run services describe {service-name} --region us-central1 --format json(status.conditions)
```

Check if the latest revision is serving traffic:

```
gcloud run revisions list --service {service-name} --region us-central1 --limit 3 --format table(name,active,ready,deployed)
```

## Notes

- All builds are in region `us-central1`
- GCP project: `lucitra-ai`
- Cloud Build triggers fire on push to `dev` (dev services) and `main` (prod services)
- Studio desktop builds use GitHub Actions, not Cloud Build — use `/apple-signing --status` for those
