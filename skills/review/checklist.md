# Pre-Landing Review Checklist

## Pass 1: CRITICAL (stop-ship issues)

### Secrets & Credentials
- [ ] No `.env` values, API keys, tokens, or passwords in diff
- [ ] No hardcoded secrets (grep for `sk-`, `npm_`, `ghp_`, `Bearer `, `password`)
- [ ] No `.env` files staged (check `.gitignore` coverage)

### TypeScript Safety
- [ ] No `any` types (explicit or implicit)
- [ ] No `@ts-ignore` or `@ts-nocheck` without justification comment
- [ ] No `as any` type assertions

### Infrastructure Sync
- [ ] If `cloudbuild.yaml` changed → Terraform also updated (and vice versa)
- [ ] If new GCP resource needed → defined in `lucitra-infrastructure/` first
- [ ] No manual `gcloud` commands in scripts (Terraform-first)

### Linear Compliance
- [ ] Branch name contains `luc-NNN` pattern
- [ ] Changes are non-trivial → Linear issue exists

### Security
- [ ] No SQL injection vectors (raw queries with string interpolation)
- [ ] No XSS vectors (dangerouslySetInnerHTML, unescaped user input)
- [ ] No command injection (exec/spawn with unsanitized input)
- [ ] Server components don't leak secrets to client

## Pass 2: INFORMATIONAL (note but don't block)

### Code Quality
- [ ] No `console.log` / `debugger` statements left in
- [ ] No TODO/FIXME/HACK without Linear issue reference (e.g., `// TODO: LUC-123`)
- [ ] No dead imports or unused variables
- [ ] No yalc references in package.json

### Patterns
- [ ] Docker-first: no direct `npm run dev` or `npx next dev` commands added
- [ ] App Router patterns: Server Components by default, `'use client'` only when needed
- [ ] Path alias: using `@/*` not relative paths that escape `src/`

### File Hygiene
- [ ] No large files (>500KB) being committed
- [ ] No generated files that should be in `.gitignore` (dist/, .next/, node_modules/)
- [ ] No OS files (.DS_Store, Thumbs.db)

### Commit Standards
- [ ] Conventional commit format (`feat:`, `fix:`, `chore:`, etc.)
- [ ] Co-Authored-By trailer present for AI-assisted commits

## DO NOT flag (suppressions)

- Version bumps in package.json (handled by /npm-publish)
- Lock file changes (package-lock.json, pnpm-lock.yaml)
- Submodule pointer updates
- Auto-generated files explicitly listed in .gitignore patterns
- Test files using `any` for mock data
