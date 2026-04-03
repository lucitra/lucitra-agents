---
name: ui-brain-lookup
description: "Quick lookup for a specific UI component pattern including anatomy, props, and accessibility. Use when user asks 'how do I build a Table', 'Modal best practices', 'Button accessibility', or needs a component reference card."
user_invocable: true
argument-hint: "<component-name>"
allowed-tools: ["Read", "Grep"]
---

# UI Brain — Quick Component Lookup

Fast lookup for a specific component pattern from the UI Design Brain.

## Arguments

Parse `$ARGUMENTS` as a component name (e.g., "Table", "Modal", "Button", "Toast", "Tabs").

## Steps

1. **Search the component reference**: Use Grep to find the component pattern for `$ARGUMENTS` in the ui-brain references. Check both global (`~/.claude/skills/ui-brain/references/components.md`) and project-local (`.claude/skills/ui-brain/references/components.md`) locations.

2. **Return the full pattern** including:
   - When to use
   - Anatomy
   - Key props
   - Accessibility requirements
   - Anti-patterns to avoid

3. **Add accessibility details**: Search the accessibility reference for ARIA patterns specific to this component type.

4. **Add project context** if available — check if the project has a surface conventions or design system file (e.g., `.claude/skills/ui-brain/references/*-surfaces.md` or a `design-system.md`) for project-specific conventions.

## Output Format

Return a concise, actionable reference card:

```
## [Component Name]

**Use when**: ...
**Anatomy**: ...
**Accessibility**: ...
**Anti-patterns**: ...
**Project note**: ... (if applicable)
```
