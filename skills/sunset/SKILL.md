---
name: sunset
description: "Sunset a service, submodule, or repo: remove submodule from lucitra-dev, disable GCP infrastructure via Terraform, archive the GitHub repo, and clean up local directories. Use when user says 'sunset X', 'remove X', 'archive X', or 'decommission X'."
argument-hint: "<service-or-repo-name> [--dry-run] [--skip-terraform] [--skip-archive]"
---

# Sunset: Decommission a Service or Repo

You are running the `/sunset` workflow. This systematically removes a service/repo from the Lucitra ecosystem.

**Arguments:**
- `$ARGUMENTS` — the service or repo name (e.g., `lucitra-validate`, `platform-api`)
- `--dry-run` — show what would happen without making changes
- `--skip-terraform` — skip GCP infrastructure changes
- `--skip-archive` — skip GitHub repo archiving

## Phase 1: Identify scope

1. Parse `$ARGUMENTS` to determine the target service/repo name
2. Check if it exists as:
   - A git submodule in `.gitmodules`
   - A local directory in `lucitra-dev/`
   - A GCP service in `lucitra-infrastructure/services.tf` or `main.tf`
   - A GitHub repo under `lucitra/`
3. Present a summary of what will be affected:
   - Submodule entry in `.gitmodules` and `.git/config`
   - Local directory on disk
   - Cloud Run services (dev/prod) — check `services.tf` module blocks
   - Cloud Build triggers (dev/prod/PR) — check `services.tf` trigger modules
   - Cloud Deploy pipelines — check `cloud_deploy_enabled` locals block
   - Artifact Registry repositories — check `main.tf` (keep these, they're cheap)
   - Secret Manager secrets — check `main.tf`
   - Custom domain mappings — check `custom-domains.tf`
   - IAM bindings — check `iap.tf` for IAP invoker resources AND `iap_service_accounts` list
   - Output blocks — check `services-outputs.tf`
   - Cross-service references — grep for `module.<service_name>` across all `.tf` files
   - GitHub repo
4. If `--dry-run`, stop here and show the summary

## Phase 2: Remove submodule (if applicable)

1. Run `git submodule deinit -f <path>` (requires sandbox disabled for .git/config writes)
2. Run `git rm -f <path>`
3. Remove the entry from `.git/modules/<path>` (where `<path>` matches the submodule path) if it exists:
   ```
   rm -rf .git/modules/<path>
   ```
4. Verify `.gitmodules` no longer contains the entry

## Phase 3: Remove local directories (if applicable)

1. Remove any non-submodule local directories related to the service
2. Check for references in:
   - `Makefile` — remove related targets
   - `docker-compose*.yml` — remove service definitions
   - `.mcp.json` — remove MCP server entries
   - `CLAUDE.md` — update references

## Phase 4: Disable GCP infrastructure (if applicable, unless --skip-terraform)

### Critical: Cloud Run v2 deletion_protection

In our standard Lucitra Terraform modules, Cloud Run v2 services are typically created with `deletion_protection = true`. When that flag is enabled you CANNOT destroy the service resource in a single `terraform apply`.

Before proceeding, confirm the current `deletion_protection` setting in the relevant `google_cloud_run_v2_service` resource or module. If `deletion_protection` is `true`, use the following sequence:

1. **Comment out the module blocks entirely** using `/* ... */` block comments
2. **Delete the Cloud Run services from GCP directly** using gcloud:
   ```
   gcloud run services delete <service-name> --region=us-central1 --quiet
   ```
3. **Identify the exact Terraform state addresses** for the Cloud Run services:
   ```
   terraform state list | grep <service-name>
   # e.g.
   # module.<service-name>_dev.google_cloud_run_v2_service.service
   # module.<service-name>_prod[0].google_cloud_run_v2_service.service
   ```
4. **Remove the services from Terraform state** (since we deleted them outside Terraform), using the addresses from the previous step:
   ```
   terraform state rm 'module.<service-name>_dev.google_cloud_run_v2_service.service'
   terraform state rm 'module.<service-name>_prod[0].google_cloud_run_v2_service.service'
   ```
5. Then `terraform apply` will cleanly destroy the remaining resources (triggers, IAM, service accounts)

**Do NOT try to set deletion_protection=false and destroy in the same apply** — Terraform resolves the destroy before the update, so it always fails.

### Terraform changes checklist

For each sunset service, update ALL of these files:

1. **`services.tf` — `prod_services_enabled` locals block:**
   - Set the service flag to `false` with a sunset comment

2. **`services.tf` — `cloud_deploy_enabled` locals block:**
   - Set the service flag to `false` with a sunset comment

3. **`services.tf` — Module blocks:**
   - Comment out ALL module blocks for the service: `<name>_dev`, `<name>_prod`, `<name>_trigger_dev`, `<name>_trigger_prod`, `<name>_trigger_pr`, `<name>_deploy`
   - Use `/* ... */` block comments with `# Sunset <date>` above each

4. **`services-outputs.tf`:**
   - Comment out output blocks referencing the sunset modules
   - Remove entries from the `all_service_urls` summary map

5. **`iap.tf`:**
   - Comment out or remove `iap_invoker_<service>_dev` resource blocks
   - Remove service account entries from `iap_service_accounts` list
   - Comment out any backend service / NEG resources for the service

6. **`custom-domains.tf`:**
   - Comment out any domain mappings for this service

7. **`main.tf`:**
   - Leave artifact registries (cheap storage, preserves image history)
   - Comment out dedicated secret modules if the service had them

8. **Cross-service references:**
   - Grep for `module.<service_name>` across ALL `.tf` files
   - Fix any references in other active services (e.g., env vars pointing to sunset service URLs)

### Commit and apply

1. Commit Terraform changes inside the `lucitra-infrastructure` submodule
2. Push the submodule: `cd lucitra-infrastructure && git push`
3. Tell the user to run:
   ```
   gcloud run services delete <service>-dev --region=us-central1 --quiet
   # (repeat for each service)
   terraform state rm 'module.<name>_dev.google_cloud_run_v2_service.service'
   terraform state rm 'module.<name>_prod[0].google_cloud_run_v2_service.service'
   # (repeat for each service)
   terraform apply
   ```

## Phase 5: Archive GitHub repo (unless --skip-archive)

1. Use `gh repo archive lucitra/<repo-name> --yes` to archive the repo
2. May need `unset http_proxy https_proxy` if TLS errors occur (proxy config issue)
3. This makes the repo read-only but preserves all history

## Phase 6: Clean up references

1. Update `CLAUDE.md` repository layout section if the service was listed
2. Remove from any `docker-compose*.yml` files
3. Remove from `Makefile` if there are service-specific targets
4. Remove from `.mcp.json` if applicable

## Phase 7: Summary

Output a checklist of everything that was done:

```
## Sunset Summary: <service-name>

### Completed
- [ ] Submodule removed from .gitmodules
- [ ] Local directory removed
- [ ] Terraform modules commented out (services.tf, services-outputs.tf, iap.tf)
- [ ] Cross-service references fixed
- [ ] GitHub repo archived
- [ ] References cleaned up (Makefile, docker-compose, CLAUDE.md)

### Manual steps required
- [ ] Delete Cloud Run services via gcloud (deletion_protection blocks terraform destroy)
- [ ] Remove Cloud Run services from Terraform state
- [ ] Run `terraform apply` to destroy remaining resources (triggers, IAM, service accounts)
- [ ] Cancel any active Stytch OAuth configs pointing to this service
- [ ] Remove DNS records if custom domains were used
```

## Rules

- **Always ask for confirmation** before executing destructive steps (unless --dry-run already shown)
- **Never try to terraform destroy Cloud Run v2 services directly** — delete via gcloud, then state rm
- **Never delete GitHub repos** — archive only
- **Keep artifact registries** — cheap storage, preserves image history
- **Commit Terraform changes inside the submodule** — push separately, then update parent ref
- **Commit changes** to a feature branch, not directly to dev
- **One service at a time** — if multiple services requested, run sequentially
- **Check for ALL dangling references** — grep `module.<name>` across every `.tf` file before applying
- **Fix iap.tf service account lists** — not just resource blocks, also the `iap_service_accounts` local
- **gh CLI may need proxy unset** — `unset http_proxy https_proxy` before `gh` commands
