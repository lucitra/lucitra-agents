---
name: Triage Dependencies
assignee: devops
project: weekly-ops
schedule:
  timezone: America/Los_Angeles
  startsAt: 2026-04-09T09:00:00-07:00
  recurrence:
    frequency: weekly
    interval: 1
    weekdays:
      - wednesday
    time:
      hour: 9
      minute: 0
---

Run `/triage-dependabot` across all Lucitra repos. Close superseded PRs, merge safe patch bumps, rebase stale ones, and report any breaking changes that need Eng Lead attention.
