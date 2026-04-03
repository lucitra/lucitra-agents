---
name: implement
description: "Pick up a Linear story and implement it end-to-end: load context, create branch, code the solution, validate, review, and open a PR. Use when user says 'implement LUC-XXX', 'pick up this story', 'work on this ticket', or 'build this feature'."
user_invocable: true
argument-hint: "<LUC-XXX>"
---

# Implement a Linear Story

Pick up a Linear story, load relevant context, implement it end-to-end, then review and ship a PR.

## Arguments

Parse `$ARGUMENTS` as a Linear issue ID (e.g., `LUC-301`) or description to search for.

## Steps

### Phase 1: Understand

1. **Find the story**: Look up `$ARGUMENTS` in Linear. If no issue ID given, search Linear for matching issues.

2. **Load context**: Read these files based on what the story touches:
   - Always: `.claude/context/product.md`, `.claude/reference/linear-keywords.md`
   - If touching infra: `.claude/context/infrastructure.md`
   - If touching auth: `.claude/context/auth-oauth.md`
   - If touching deployment: `.claude/context/deployment.md`
   - Read the service's own `CLAUDE.md` if it exists

3. **Understand the story**: Read the full issue description, acceptance criteria, and linked PRD/architecture docs.

4. **Update Linear**: Set issue status to `In Progress`.

### Phase 2: Implement (in worktree)

5. **Enter worktree for isolation**: Use the `EnterWorktree` tool so this implementation is fully isolated from any other in-progress work:
   ```
   EnterWorktree(name: "luc-{number}-{short-description}")
   ```
   This creates a fresh worktree in `.claude/worktrees/` with its own branch based on HEAD. Multiple `/implement` sessions can run in parallel without conflicts.

6. **Set up branch**: Inside the worktree, create the feature branch from `dev`:
   ```bash
   git fetch origin dev && git checkout -b founder/luc-{number}-{short-description} origin/dev
   ```

7. **Implement**: Follow the acceptance criteria. Use Docker for local development. Follow standards in `.claude/context/dev-standards.md`.

8. **Validate**: Run `make validate-{service}` to catch issues before committing.

### Phase 3: Review & Ship

9. **Pre-landing review**: Run the review checklist from `.claude/skills/review/checklist.md` against the diff.
   - Check the full diff: `git diff dev`
   - **Pass 1 (CRITICAL)**: Secrets, `any` types, Terraform sync, security
   - **Pass 2 (INFORMATIONAL)**: Console.log, dead code, TODOs, patterns
   - If critical issues found, fix them before proceeding
   - If only informational issues, note them for the PR description

10. **Stage and commit**: Use conventional commit format with Linear magic words:
    ```
    {type}: {description}

    closes LUC-{number}

    Co-Authored-By: Claude <noreply@anthropic.com>
    ```
    For large changesets, split into bisectable commits (infra first, logic second, UI third).

11. **Push and create PR**:
    ```bash
    git push -u origin $(git branch --show-current)
    gh pr create --base dev --title "{type}: {summary}" --body "..."
    ```
    PR body must include: Summary, Linear reference (`closes LUC-XXX`), review findings, validation results, test plan.

12. **Output the PR URL.**

### Phase 4: Babysit to Merge-Readiness

13. **Babysit the PR** using the `/babysit-pr` workflow (pre-flight, Copilot review, CI, Slack notification). See that skill for full details.

### Phase 5: Cleanup

14. **Exit the worktree**: After the PR is created and babysit is complete, exit and keep the worktree (the branch is on the remote now):
    ```
    ExitWorktree(action: "keep")
    ```
    The worktree can be removed after the PR is merged. If the implementation was abandoned, use `ExitWorktree(action: "remove")` instead.

## Important Rules

- **Never push directly to dev or main.** Always create a PR.
- **Never commit without running the review checklist.** It catches what validation misses.
- **Never skip validation.** If `make validate-*` fails, fix the issues first.
- **Always include Linear reference.** No commits without `LUC-XXX`.
- **Never merge the PR.** Leave the final merge to the human.
- **Ask for user confirmation** before creating the commit and PR.
