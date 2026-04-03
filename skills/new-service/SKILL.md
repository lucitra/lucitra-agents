---
name: new-service
description: "Scaffold a new Lucitra service with Dockerfile, cloudbuild.yaml, TypeScript config, and make targets. Use when user says 'create a new service', 'scaffold a new app', 'add a new microservice', or 'bootstrap a new project'."
user_invocable: true
argument-hint: "<service-name>"
---

# Scaffold a New Service

Create a new Lucitra service with all required boilerplate.

## Arguments

Parse `$ARGUMENTS` as a service name (e.g., `lucitra-analytics`).

## Steps

1. **Load context**:
   - `.claude/context/dev-standards.md` — Project structure, npm scripts, Docker requirements
   - `.claude/context/infrastructure.md` — Terraform requirements
   - `.claude/context/deployment.md` — CI/CD setup

2. **Validate naming**: Service must follow `lucitra-[name]` convention.

3. **Create repository structure**:
   ```
   lucitra-{name}/
   ├── src/                    # Application source code
   │   └── app/                # (Next.js) or index.ts (API)
   ├── test/                   # Tests
   ├── public/                 # (frontend only) Static assets
   ├── Dockerfile              # Multi-stage Docker build
   ├── docker-compose.yml      # Local development
   ├── cloudbuild.yaml         # CI/CD pipeline
   ├── package.json            # With standardized npm scripts
   ├── tsconfig.json           # @/* path alias
   ├── CLAUDE.md               # Service-specific instructions
   └── README.md               # Service documentation
   ```

4. **Configure package.json** with standardized npm scripts (dev, build, lint, type-check, docker:*, etc.)

5. **Set up TypeScript** with `@/*` → `./src/*` path alias.

6. **Create Dockerfile** with multi-stage build for optimized containers.

7. **Create cloudbuild.yaml** for Cloud Build triggers (dev + main branches).

8. **Terraform reminder**: All GCP infrastructure must be defined in `lucitra-infrastructure/` BEFORE creating Cloud resources:
   - Cloud Run service (dev + prod)
   - Cloud Build triggers
   - Artifact Registry repository
   - Secret Manager secrets (if needed)
   - Cloud SQL database/user (if needed)

9. **Add to docker-compose.yml** in `lucitra-dev` root for local development.

10. **Add make targets** in `lucitra-dev/Makefile` for the new service.

11. **Ask user** to create the GitHub repo at `lucitra/{service-name}` and add as submodule.

## Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| `make` target not found after adding | Makefile not updated | Add targets to `lucitra-dev/Makefile` under the service section |
| Docker build fails | Missing `COPY` or wrong `WORKDIR` | Verify Dockerfile paths match `src/` structure |
| `@/*` imports not resolving | tsconfig paths misconfigured | Ensure `"paths": {"@/*": ["./src/*"]}` in `tsconfig.json` |
| Submodule not tracking | Git config issue | Run `git submodule add https://github.com/lucitra/{name}.git` from root |
