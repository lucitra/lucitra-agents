# Lucitra Agents

An [Agent Companies](https://agentcompanies.io/specification) package for AI-powered software engineering.

## How It Works

Lucitra Agents operates as a **pipeline** for feature work with **parallel ops** for maintenance:

```
CEO (plan/prioritize)
  ↓ assigns stories
Eng Lead (implement + review)
  ↓ branch ready
QA Engineer (test + validate)
  ↓ approved
Release Manager (ship + babysit PR)
  ↓ merged

DevOps runs in parallel:
  CI/CD, security, dependency triage
```

## Org Chart

| Agent | Title | Reports To | Skills |
|-------|-------|-----------|--------|
| CEO | Chief Executive Officer | — | paperclip, create-plan, retro |
| Eng Lead | Engineering Lead | CEO | implement, review, ship, validate, create-plan |
| QA Engineer | QA Engineer | Eng Lead | qa, ui-brain, ui-brain-lookup, review |
| Release Manager | Release Manager | Eng Lead | npm-publish, ship, babysit-pr, resolve-review |
| DevOps | DevOps Engineer | Eng Lead | validate, security-audit, triage-dependabot, cloud-build |

## Skills (19)

All skills are bundled in `skills/` and can be imported individually or as part of the company.

### Generic (reusable across any project)

| Skill | Description |
|-------|-------------|
| `babysit-pr` | Shepherd PRs to merge-readiness through CI and review |
| `create-plan` | PRD -> Architecture -> Stories implementation planning |
| `npm-publish` | Version bump, build, publish npm packages |
| `qa` | Systematic QA testing with screenshots and reports |
| `resolve-review` | Fix PR review comments and resolve threads |
| `retro` | Engineering retrospective from git history |
| `ui-brain` | UI design patterns, accessibility, responsive design |
| `ui-brain-lookup` | Quick component pattern reference cards |
| `validate` | Run CI checks locally before pushing |

### Lucitra-specific

| Skill | Description |
|-------|-------------|
| `apple-signing` | Apple code signing and release for Tauri desktop apps |
| `cloud-build` | GCP Cloud Build monitoring |
| `deploy` | Lucitra service deployment |
| `implement` | Pick up Linear story end-to-end |
| `prime` | Load service context before working |
| `review` | Pre-landing code review with Lucitra checklist |
| `security-audit` | Security audit across Lucitra repos |
| `ship` | Automated PR creation to dev branch |
| `sunset` | Decommission services and infrastructure |
| `triage-dependabot` | Triage Dependabot PRs across repos |

## Recurring Tasks

| Task | Agent | Schedule |
|------|-------|----------|
| Weekly Retro | CEO | Monday 9am PT |
| Triage Dependencies | DevOps | Wednesday 9am PT |
| Security Scan | DevOps | 1st of month 10am PT |

## Usage

### Import the whole company into Paperclip

```bash
paperclip company import --from github.com/lucitra/lucitra-agents
```

### Import a single skill

```bash
paperclip skill import lucitra/lucitra-agents/retro
```

### Use skills with Claude Code directly

```bash
# Copy a skill to your global skills
cp -r skills/retro ~/.claude/skills/retro
```

## References

- [Agent Companies Specification](https://agentcompanies.io/specification)
- [Agent Skills Specification](https://agentskills.io/specification)
- [Paperclip](https://github.com/paperclipai/paperclip)
