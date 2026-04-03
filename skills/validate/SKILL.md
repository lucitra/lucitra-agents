---
name: validate
description: "Run validation checks locally before pushing. Use when user says 'validate', 'run checks', 'pre-push checks', 'does it pass CI', or before shipping code. Auto-discovers make targets or package.json scripts."
user_invocable: true
argument-hint: "<service|all>"
---

# Run Validation Pipeline

Run the same checks as CI locally to catch failures before pushing.

## Arguments

Parse `$ARGUMENTS` as a service name, script name, or `all` (default: `all`).

## Steps

### 1. Discover Validation Targets

Check what validation system the project uses, in order:

**a) Makefile with validate targets:**
```bash
# Check if Makefile exists and has validate targets
grep -E '^validate' Makefile 2>/dev/null | sed 's/:.*//'
```
If found, use `make validate` (all) or `make validate-<service>`.

**b) package.json scripts:**
```bash
# Check for test/lint/typecheck scripts
node -e "const p=require('./package.json'); console.log(Object.keys(p.scripts||{}).filter(s => /^(test|lint|check|typecheck|validate|build)/.test(s)).join('\n'))"
```
If found, map common scripts: `test`, `lint`, `typecheck`, `build`.

**c) Monorepo with turbo/nx:**
```bash
# Check for turbo.json or nx.json
ls turbo.json nx.json 2>/dev/null
```
If found, use `npx turbo run lint test typecheck` or equivalent.

### 2. Run Validation

Execute the appropriate target:
- If `$ARGUMENTS` is `all` or empty: run the full validation suite
- If `$ARGUMENTS` is a service name: run validation for that specific service
- If the target doesn't exist: run `make help` or list available scripts and suggest the closest match

### 3. Report Results

- If all checks pass: confirm ready to push
- If any fail: show the specific failures and suggest fixes

## Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| `make: *** No rule to make target 'validate-X'` | Service name not recognized | Run `make help` to list valid targets |
| Docker daemon not running | Docker Desktop not started | Start Docker Desktop, wait for it to be ready |
| `EACCES` or permission denied | Node modules or Docker volume issue | Run `make clean-{service}` then retry |
| TypeScript errors in `node_modules` | Stale dependencies | Reinstall dependencies and retry |
| Timeout during build | Large service or slow network | Retry; check Docker resource allocation (CPU/RAM) |
