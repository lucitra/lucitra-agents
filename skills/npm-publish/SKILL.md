---
name: npm-publish
description: "Version bump, build, and publish npm packages to registry, then commit, tag, and push to GitHub. Use when user says 'publish package', 'release npm', 'bump version', or 'publish all packages'."
argument-hint: "<package-name|all> <patch|minor|major>"
---

# /npm-publish — Publish npm Packages

Handles the full npm package release lifecycle: version bump → build → publish → commit → tag → push.

## Arguments

Parse `$ARGUMENTS` as:
- **First arg**: Package name (short name or `all` for every publishable package)
- **Second arg**: Semver bump type — `patch` (default), `minor`, or `major`

Examples:
- `/npm-publish my-lib patch` → bumps 0.1.2 → 0.1.3
- `/npm-publish utils minor` → bumps 0.1.1 → 0.2.0
- `/npm-publish all patch` → publishes all packages

## Package Discovery

Discover publishable packages by checking common monorepo layouts:

1. Look for `packages/*/package.json`
2. If not found, check `libs/*/package.json`, `modules/*/package.json`
3. If a single-package repo, use the root `package.json`

To resolve a package name:
1. Look for `packages/<name>/package.json`
2. If not found, try common prefixes from the npm scope (e.g., `packages/<scope>-<name>/package.json`)
3. If `all`, find every `packages/*/package.json` that has a scoped `"name"` (e.g., `@org/...`)

Detect the npm scope from existing packages: `node -e "const p=require('./packages/' + require('fs').readdirSync('./packages')[0] + '/package.json'); console.log(p.name.split('/')[0])"` or from the root `package.json`.

## Steps

### 1. Pre-flight Checks

For each target package:

- Read `packages/<name>/package.json` to get current version and package name
- Run `git status` inside the package directory to check for uncommitted changes
- If there are uncommitted changes, **stop and ask the user** whether to include them or stash them first
- Verify the package has `"prepublishOnly": "npm run build"` in scripts (if not, warn that build won't happen automatically)

### 2. Version Bump

- Calculate the new version using semver rules:
  - `patch`: 0.1.2 → 0.1.3
  - `minor`: 0.1.2 → 0.2.0
  - `major`: 0.1.2 → 1.0.0
- Update `"version"` in `package.json`
- If the package has a hardcoded version string in source files (e.g., `version: '0.1.0'` in `McpServer()`), update those too — but prefer the `pkg.version` pattern (reading from package.json at runtime) so this isn't needed
- Tell the user what version bump you're making and confirm before proceeding

### 3. Install & Build

```bash
cd packages/<name> && npm install && npm run build
```

- If `npm install` fails with pnpm traversal errors ("Cannot read properties of null"), retry with `--install-links`

### 4. Publish

Authentication: Check for npm auth via `~/.npmrc` or project `.npmrc`:

```bash
# Extract NPM_TOKEN from ~/.npmrc (handles both hardcoded and env-var styles)
export NPM_TOKEN=$(grep '//registry.npmjs.org/:_authToken=' ~/.npmrc | tail -1 | sed 's/.*_authToken=//')
cd packages/<name> && npm publish --access public
```

- If `NPM_TOKEN` is empty and `~/.npmrc` has no token, ask the user to run `npm config set //registry.npmjs.org/:_authToken=<token>`
- Wait for confirmation that publish succeeded (look for `+ @scope/<name>@<version>`)
- If publish fails, stop and diagnose

### 5. Git Commit & Tag

After successful publish:

```bash
cd packages/<name>
git add package.json package-lock.json
git commit -m "release: <package-name>@<new-version>

Co-Authored-By: Claude <noreply@anthropic.com>"
git tag v<new-version>
```

- Use the package's git directory (it may be a submodule with its own repo)
- The commit message uses `release:` conventional commit type
- Tag format: `v<version>` (e.g., `v0.1.3`)

### 6. Push

```bash
cd packages/<name>
git push origin $(git branch --show-current) --tags
```

- Push both the commit and the tag in one command
- Push to whatever branch is currently checked out

### 7. Summary

Print a summary table:

```
| Package        | Old     | New     | npm                    | Git   |
|----------------|---------|---------|------------------------|-------|
| @scope/my-lib  | 0.1.2   | 0.1.3   | ✓ published            | ✓ pushed + tagged |
```

## Multi-Package Mode (`all`)

When publishing all packages:
- Discover all publishable packages with scoped names
- Show the list and ask for confirmation before proceeding
- Process each package sequentially (npm auth tokens may expire between publishes)
- If any package fails, continue with the rest and report failures at the end

## Error Recovery

- **npm cache EPERM**: Tell user to run `sudo chown -R $(whoami) ~/.npm`
- **pnpm traversal crash**: Ensure `.npmrc` with `install-strategy=nested` exists in the package directory
- **OTP/401/403**: `NPM_TOKEN` is missing or expired — check `npm whoami` with the token exported, or generate a new granular token at npmjs.com
- **Version already exists on npm**: The version was already published — skip or bump again
- **Build failure**: Stop and show the error — don't publish broken code

## Notes

- Never publish packages that contain `.env` files or secrets
- The `prepublishOnly` script handles building, so a separate build step is only needed if you want to verify before publishing
- Always check `npm view <package-name>` after publish to confirm it's live
