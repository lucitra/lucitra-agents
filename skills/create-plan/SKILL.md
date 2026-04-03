---
name: create-plan
description: "Design an implementation plan following PRD, Architecture, Stories workflow. Use when user says 'plan a feature', 'create a PRD', 'design architecture', 'write stories', 'new feature proposal', or needs to go from idea to implementation plan."
user_invocable: true
argument-hint: "<feature-name>"
---

# Create an Implementation Plan

Design a structured implementation plan following a PRD → Architecture → Stories methodology.

## Arguments

Parse `$ARGUMENTS` as a feature name or description.

## Steps

1. **Determine workflow type** based on the request:
   - New feature → Full Flow (PRD → Architecture → Stories → Implementation)
   - Bug fix / config change → Quick Fix (Stories → Implementation)
   - Refactor / performance → Tech Debt (Architecture → Stories → Implementation)
   - Research / exploration → Discovery (PRD only)

2. **Check for existing artifacts**:
   - Search `docs/prds/` for existing PRD (or project-specific docs directory)
   - Search `docs/architecture/` for existing architecture doc
   - Search the issue tracker (Linear, GitHub Issues, etc.) for related issues

3. **Load workflow context**: If the project has a workflow doc (e.g., `WORKFLOW.md`, `.claude/context/workflow.md`), read it for templates and phase gates. Otherwise, use the default templates below.

4. **Create the appropriate documents**:
   - **PRD** at `docs/prds/{feature-name}.md` containing:
     - Problem statement
     - User stories / jobs to be done
     - Scope (in/out)
     - Success metrics
     - Open questions
   - **Architecture** at `docs/architecture/{feature-name}.md` (after PRD approval) containing:
     - Technical approach
     - Component/service design
     - Data model changes
     - API contracts
     - Dependencies and risks
   - **Stories** as issue tracker tickets (after Architecture approval) containing:
     - User story format
     - Acceptance criteria
     - Estimated complexity

5. **Present plan to user** with:
   - Recommended workflow type
   - Scope summary
   - Proposed stories with acceptance criteria
   - Dependencies and risks

6. **Gate**: Wait for explicit user approval at each phase before proceeding.

Remember: Refuse to implement features without proper documentation chain (unless Quick Fix path).

## Troubleshooting

| Issue | Resolution |
|-------|------------|
| No existing PRD found but feature seems familiar | Search issue tracker for related issues; the PRD may live in a different naming convention |
| User wants to skip PRD phase | Only allowed for bug fixes and config changes (Quick Fix path). For new features, PRD is mandatory |
| Architecture doc references services that don't exist yet | Flag as a dependency; scaffold before implementation |
| Issue tracker search returns too many results | Filter by team, status, or label to narrow down |
