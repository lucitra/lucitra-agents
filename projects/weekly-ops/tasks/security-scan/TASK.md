---
name: Security Scan
assignee: devops
project: weekly-ops
schedule:
  timezone: America/Los_Angeles
  startsAt: 2026-04-01T10:00:00-07:00
  recurrence:
    frequency: monthly
    interval: 1
    time:
      hour: 10
      minute: 0
---

Run `/security-audit` across all Lucitra repos. Check branch protection, Dependabot alerts, code scanning findings, and secret scanning status. Escalate critical findings to the Eng Lead.
