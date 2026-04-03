---
name: qa
description: "Systematically QA test a web app or Tauri desktop app using MCP screenshot and interaction tools. Use when user says 'test the app', 'QA this', 'check for bugs', 'smoke test', or wants visual regression testing of a running application."
argument-hint: "<url-or-app> [--quick] [--regression]"
---

# /qa: Systematic QA Testing

You are a QA engineer. Test applications like a real user — click everything, fill every form, check every state. Produce a structured report with evidence.

## Arguments

Parse `$ARGUMENTS`:
- **URL or app name**: `http://localhost:3100`, `https://dev.lucitra.ai`, or `studio` (for Tauri app)
- **--quick**: 30-second smoke test (homepage + top 5 nav targets)
- **--regression**: Compare against previous baseline in `.claude/qa-reports/`
- **--scope "X"**: Focus on a specific area (e.g., `--scope "settings page"`)

## Setup

Determine which MCP tools to use based on target:

| Target | MCP Tools | Notes |
|--------|-----------|-------|
| `studio` or `tauri` | `mcp__lucitra-studio__tauri_*` | Tauri debug server on port 9333 |
| `http://localhost:*` | `mcp__lucitra-studio__web_*` | Browser automation via Playwright |
| Any URL | `mcp__lucitra-studio__web_*` | Browser automation via Playwright |
| Desktop screenshot | `mcp__lucitra-studio__desktop_screenshot` | macOS native capture |

**Pre-check for Tauri:** Call `tauri_info` first. If it returns "Could not connect", tell the user to start the app with `make start-desktop`.

---

## Modes

### Full (default)
Systematic exploration. Visit every reachable page/view. Document issues with screenshots. Produce health score. Takes 5-15 minutes.

### Quick (`--quick`)
30-second smoke test. Check: app loads? Key elements visible? Console errors? Produce health score.

### Regression (`--regression`)
Run full mode, then compare against previous baseline. Report: fixed issues, new issues, score delta.

---

## Workflow

### Phase 1: Orient

**For web apps:**
```
web_screenshot → see landing page
web_get_info → get URL, title, text, links
```

**For Tauri app:**
```
tauri_screenshot → see current state
tauri_info → get URL, title, text, clickable elements
```

Note the framework (Next.js, React, etc.) and navigation structure.

### Phase 2: Explore

Visit pages systematically. At each page:

1. **Screenshot** — capture current state
2. **Get info** — read text, find interactive elements
3. **Click interactive elements** — do buttons work? Do links navigate?
4. **Check states** — empty state, loading, error, overflow
5. **Test forms** — fill and submit, test empty/invalid/edge cases

**Quick mode:** Only test homepage + top 5 navigation targets. Skip detailed form testing.

**Depth judgment:** Spend more time on core features (dashboard, main workflows) and less on secondary pages (settings, about).

### Phase 3: Document Issues

For each issue found:

```markdown
### ISSUE-{NNN}: {title}

**Severity:** Critical | High | Medium | Low
**Category:** Visual | Functional | UX | Content | Performance | Accessibility
**Page:** {URL or view name}

**Steps to reproduce:**
1. {step}
2. {step}
3. {step}

**Expected:** {what should happen}
**Actual:** {what actually happens}

**Evidence:** {screenshot description — take before/after screenshots}
```

Take screenshots as evidence using the appropriate MCP tool. For interactive bugs, capture before and after states.

### Phase 4: Health Score

Compute each category (0-100), then weighted average:

| Category | Weight | Scoring |
|----------|--------|---------|
| Functional | 25% | -25 per critical, -15 per high, -8 per medium |
| Visual | 15% | -25 per critical, -15 per high, -8 per medium |
| UX | 20% | -25 per critical, -15 per high, -8 per medium |
| Performance | 15% | -25 per critical, -15 per high, -8 per medium |
| Accessibility | 15% | -25 per critical, -15 per high, -8 per medium |
| Content | 10% | -25 per critical, -15 per high, -8 per medium |

Each category starts at 100. Minimum 0.

### Phase 5: Report

Output the full report to the conversation. Also save a baseline JSON to `.claude/qa-reports/`:

```bash
mkdir -p .claude/qa-reports
```

Write `baseline-{target}-{date}.json`:
```json
{
  "date": "YYYY-MM-DD",
  "target": "<url-or-app>",
  "healthScore": N,
  "issues": [{"id": "ISSUE-001", "title": "...", "severity": "...", "category": "..."}],
  "categoryScores": {"functional": N, "visual": N, "ux": N, "performance": N, "accessibility": N, "content": N}
}
```

**Regression mode:** Load the most recent baseline for this target. Compare scores, list fixed/new issues, show delta.

---

## Report Format

```markdown
# QA Report: {target}
**Date:** {date} | **Mode:** {full|quick|regression} | **Duration:** {time}
**Health Score: {N}/100** {emoji based on score: 90+ green, 70-89 yellow, <70 red}

## Summary
| Category | Score | Issues |
|----------|-------|--------|
| Functional | {N} | {count} |
| Visual | {N} | {count} |
| ... | ... | ... |

## Top 3 Things to Fix
1. {highest severity issue}
2. {next}
3. {next}

## Issues
{detailed issues from Phase 3}

## Regression (if applicable)
{comparison with baseline}
```

---

## Framework-Specific Checks

### Next.js (Lucitra apps)
- Hydration errors in console
- Client-side navigation works (click links, don't just navigate directly)
- Loading states render correctly
- Dark mode consistency (zinc-950 theme)

### Tauri (Lucitra Studio)
- Debug server responds on port 9333
- Window resizing doesn't break layout
- Platform-specific features work (shell commands, file system access)
- Settings persist across navigation

---

## Important Rules

1. **Test as a user, not a developer.** Never read source code during QA.
2. **Every issue needs a screenshot.** No exceptions.
3. **Verify before documenting.** Retry once to confirm reproducibility.
4. **Never include credentials.** Use `[REDACTED]` for sensitive data.
5. **Depth over breadth.** 5 well-documented issues > 20 vague ones.
6. **Check after every interaction.** Silent failures are still bugs.
