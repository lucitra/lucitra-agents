---
name: retro
description: "Generate an engineering retrospective from git history analyzing velocity, commit patterns, session detection, and code quality trends. Use when user says 'retro', 'weekly review', 'how productive was I', 'engineering metrics', or wants to review recent work."
argument-hint: "[7d|14d|30d|24h] [compare]"
---

# /retro — Engineering Retrospective

Generates a retrospective analyzing commit history, work patterns, and code quality metrics. Designed for a solo founder using Claude Code as a force multiplier.

## Arguments

- `/retro` — last 7 days (default)
- `/retro 24h` — last 24 hours
- `/retro 14d` — last 14 days
- `/retro 30d` — last 30 days
- `/retro compare` — compare current window vs prior same-length window

## Instructions

### Step 0: Detect Project Context

Determine the project's conventions by reading the environment:

```bash
# Detect default branch
DEFAULT_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@')
if [ -z "$DEFAULT_BRANCH" ]; then
  DEFAULT_BRANCH=$(git branch -r | grep -E 'origin/(main|dev|master)' | head -1 | sed 's@.*origin/@@' | tr -d ' ')
fi

# Detect user timezone
USER_TZ=$(readlink /etc/localtime 2>/dev/null | sed 's@.*/zoneinfo/@@' || echo "UTC")
```

Use `origin/$DEFAULT_BRANCH` for all queries and `$USER_TZ` for timestamps throughout.

### Step 1: Gather Raw Data

Fetch origin and identify the user:
```bash
git fetch origin $DEFAULT_BRANCH --quiet
git config user.name
```

Run these git commands:

```bash
# All commits in window
git log origin/$DEFAULT_BRANCH --since="<window>" --format="%H|%aN|%ae|%ai|%s" --shortstat

# Per-commit LOC breakdown (test vs production)
git log origin/$DEFAULT_BRANCH --since="<window>" --format="COMMIT:%H|%aN" --numstat

# Timestamps for session detection
TZ=$USER_TZ git log origin/$DEFAULT_BRANCH --since="<window>" --format="%at|%aN|%ai|%s" | sort -n

# Hotspot analysis
git log origin/$DEFAULT_BRANCH --since="<window>" --format="" --name-only | grep -v '^$' | sort | uniq -c | sort -rn

# Per-author commit counts
git shortlog origin/$DEFAULT_BRANCH --since="<window>" -sn --no-merges
```

### Step 2: Metrics Summary

| Metric | Value |
|--------|-------|
| Commits to $DEFAULT_BRANCH | N |
| Contributors | N |
| Total insertions | N |
| Total deletions | N |
| Net LOC | N |
| Test LOC ratio | N% |
| Active days | N |
| Sessions detected | N |
| AI-assisted commits | N% |

**AI-assisted:** Count commits with `Co-Authored-By:` trailers mentioning `anthropic.com`, `noreply`, or other AI tool signatures.

### Step 3: Commit Time Distribution

Show hourly histogram in the user's timezone:

```
Hour  Commits  ████████████████
 09:    5      █████
 10:    8      ████████
 ...
```

Call out peak hours, dead zones, and late-night patterns.

### Step 4: Work Session Detection

Detect sessions using **45-minute gap** threshold between consecutive commits.

Classify:
- **Deep sessions** (50+ min) — focused implementation
- **Medium sessions** (20-50 min) — feature work
- **Micro sessions** (<20 min) — quick fixes

Calculate: total active time, average session length, LOC per hour.

### Step 5: Commit Type Breakdown

Categorize by conventional commit prefix:

```
feat:     20  (40%)  ████████████████████
fix:      15  (30%)  ███████████████
chore:    10  (20%)  ██████████
refactor:  5  (10%)  █████
```

Flag if fix ratio exceeds 50% — signals "ship fast, fix fast" pattern.

### Step 6: Hotspot Analysis

Top 10 most-changed files. Flag:
- Files changed 5+ times (churn hotspots)
- Test files vs production files in the hotspot list
- Submodule pointer changes (indicate cross-repo work)

### Step 7: Directory Distribution

Map commits to top-level directories or services:

```bash
git log origin/$DEFAULT_BRANCH --since="<window>" --format="" --name-only | grep -v '^$' | cut -d'/' -f1 | sort | uniq -c | sort -rn
```

Display as a bar chart. If the project has a service/package structure, group by service.

### Step 8: Focus Score

Calculate percentage of commits touching the single most-changed top-level directory. Higher = deeper focused work. Lower = scattered context-switching.

**Ship of the week:** Auto-identify the single highest-impact PR or commit. Highlight what it was and why it matters.

### Step 9: Streak Tracking

```bash
TZ=$USER_TZ git log origin/$DEFAULT_BRANCH --format="%ad" --date=format:"%Y-%m-%d" | sort -u
```

Count consecutive days with at least 1 commit, going back from today.

### Step 10: Load History & Compare

Check for prior retros:
```bash
ls -t .claude/retros/*.json 2>/dev/null
```

If prior retros exist, load the most recent and show deltas:
```
                Last      Now       Delta
Commits:        32   →    47        ↑47%
Test ratio:     22%  →    41%       ↑19pp
Sessions:       10   →    14        ↑4
LOC/hour:       200  →    350       ↑75%
Fix ratio:      54%  →    30%       ↓24pp (improving)
```

### Step 11: Save Snapshot

```bash
mkdir -p .claude/retros
```

Save JSON snapshot to `.claude/retros/{date}.json` with all metrics for future comparison.

### Step 12: Write the Narrative

**Tweetable summary** (first line):
```
Week of Mar 7: 47 commits, 3.2k LOC, 38% tests, peak: 10pm | Streak: 12d
```

Then structured sections:
1. **Summary Table** (from Step 2)
2. **Trends vs Last Retro** (if prior data exists)
3. **Time & Session Patterns** — when you're most productive, session quality
4. **Shipping Velocity** — commit types, PR sizes, fix chains
5. **Code Quality Signals** — test ratio, hotspots
6. **Focus & Highlights** — focus score, ship of the week
7. **Directory Distribution** — where effort went across the repo
8. **Top 3 Wins** — highest-impact things shipped
9. **3 Things to Improve** — specific, actionable, anchored in data
10. **3 Habits for Next Week** — small, practical (<5 min to adopt)

---

## Compare Mode

When `/retro compare`:
1. Compute metrics for current window
2. Compute metrics for prior same-length window using `--since` and `--until`
3. Side-by-side table with deltas and arrows
4. Narrative on biggest improvements and regressions

---

## Tone

- Encouraging but candid
- Specific and concrete — anchor in actual commits
- Skip generic praise — say exactly what was good
- Frame improvements as leveling up
- ~2000-3000 words
- Output directly to conversation (only JSON snapshot written to file)

## Important Rules

- Use `origin/$DEFAULT_BRANCH` for all queries (not local branch which may be stale)
- Convert timestamps to the user's local timezone
- If zero commits in window, say so and suggest a different window
- Track AI-assisted commits as a metric, not a judgment
- Never read CLAUDE.md or source code — this skill is self-contained
