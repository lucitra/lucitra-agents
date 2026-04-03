---
name: docs-monitor
description: "Fetch latest Claude Code documentation, compare against our skills setup, and report new features or breaking changes. Use when user says 'check for Claude updates', 'audit our skills', 'what changed in Claude docs', or wants to keep skills current."
disable-model-invocation: true
argument-hint: "[skills|hooks|agents|all]"
---

# Claude Code Documentation Monitor

Fetch the latest Claude Code documentation pages, compare against our current skills setup, and report what's new or changed.

## Arguments

Parse `$ARGUMENTS` as the topic to check (default: `all`):
- `skills` — just the skills docs
- `hooks` — hooks docs
- `agents` — subagents docs
- `plugins` — plugins docs
- `all` — check all pages

## Documentation Pages

| Topic | URL |
|-------|-----|
| Skills | https://code.claude.com/docs/en/skills |
| Hooks | https://code.claude.com/docs/en/hooks |
| Sub-agents | https://code.claude.com/docs/en/sub-agents |
| Plugins | https://code.claude.com/docs/en/plugins |
| Settings | https://code.claude.com/docs/en/settings |
| Permissions | https://code.claude.com/docs/en/permissions |
| Memory | https://code.claude.com/docs/en/memory |
| Index | https://code.claude.com/docs/llms.txt |

## Steps

1. **Fetch the docs index** at `https://code.claude.com/docs/llms.txt` to discover all available pages and check for new sections.

2. **Fetch the relevant page(s)** based on the argument. Use WebFetch to retrieve each page.

3. **Compare against our current setup**:
   - Read our skills in `.claude/skills/` to understand what we currently use
   - Read `.claude/README.md` for our documented workflow
   - Check for new frontmatter fields, features, or patterns we're not using

4. **Report findings** in this format:

   ### New Features
   - Features in the docs that we're not using yet
   - Include the specific frontmatter field or pattern

   ### Breaking Changes
   - Anything deprecated or changed that affects our setup

   ### Recommendations
   - Specific changes to make to our skills, ranked by impact

   ### Our Skills Audit
   - For each of our skills, note if any improvements are available based on new docs

5. **If changes are warranted**, propose specific edits to our skill files. Ask for confirmation before making changes.

## Snapshot

Save a summary of what was found to `.claude/skills/docs-monitor/last-check.md` with the date, so future runs can diff against it.

## Usage

Run manually when you want to check for updates:
```
/docs-monitor skills
/docs-monitor all
```

Or set up periodic monitoring:
```
/loop 24h /docs-monitor all
```
