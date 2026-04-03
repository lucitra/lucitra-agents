---
date: 2026-03-12
pages_checked: skills, hooks, sub-agents, plugins, settings, permissions, memory, llms.txt index
---

# Docs Monitor — Last Check

## Pages tracked (8)
- skills, hooks, sub-agents, plugins, settings, permissions, memory, llms.txt

## Key findings
- No custom sub-agents defined (.claude/agents/ does not exist)
- No .claude/rules/ directory (path-specific rules not in use)
- Skills use `user_invocable` (deprecated field name) — should use default (true) or `disable-model-invocation`
- Not using: context:fork, agent field, !`command` injection, $ARGUMENTS[N], ${CLAUDE_SKILL_DIR}, hooks in skills, model override
- Not using: /batch, /debug bundled skills
- Docs now cover: agent teams, plugins reference, plugin marketplaces, LSP servers, scheduled tasks, remote control, output styles

## Skills audit
- 11 skills total, 3 with disable-model-invocation (deploy, npm-publish, docs-monitor)
- ui-brain missing frontmatter (no --- block)
- allowed-tools format uses JSON array syntax — docs show comma-separated string
