# Accessibility Reference — WCAG 2.1 AA

## Core Requirements

WCAG 2.1 Level AA is the baseline for all Lucitra surfaces. This means:

### Perceivable
1. **Text alternatives**: All non-text content has a text alternative (`alt`, `aria-label`, `aria-labelledby`)
2. **Captions**: Video content has captions; audio has transcripts
3. **Adaptable**: Content can be presented in different ways without losing information (semantic structure)
4. **Distinguishable**: Content is easy to see and hear (contrast, resize, spacing)

### Operable
1. **Keyboard accessible**: All functionality available via keyboard
2. **Enough time**: Users can adjust time limits (auto-dismiss toasts: minimum 5s + 1s per 120 words)
3. **No seizures**: Nothing flashes more than 3 times per second
4. **Navigable**: Users can find content and know where they are (focus order, page titles, headings)
5. **Input modalities**: Functionality can be operated through various inputs (pointer, keyboard, voice)

### Understandable
1. **Readable**: Text is readable and understandable (language declared, abbreviations explained)
2. **Predictable**: Pages operate in predictable ways (consistent navigation, no unexpected context changes)
3. **Input assistance**: Users are helped to avoid and correct mistakes (labels, errors, suggestions)

### Robust
1. **Compatible**: Content is compatible with assistive technologies (valid HTML, ARIA where needed)

## Color Contrast

| Element | Minimum Ratio | Check With |
|---------|---------------|------------|
| Normal text (<18px) | 4.5:1 | foreground vs background |
| Large text (≥18px bold or ≥24px) | 3:1 | foreground vs background |
| UI components (borders, icons) | 3:1 | component vs adjacent color |
| Focus indicators | 3:1 | indicator vs adjacent colors |
| Disabled elements | No requirement | But should be visually distinct |

### Tools
- Chrome DevTools → Accessibility panel → Contrast ratio
- axe DevTools browser extension
- Stark Figma plugin (design phase)

## ARIA Roles, States, and Properties

### Rule of Thumb
> Use semantic HTML first. Add ARIA only when HTML semantics are insufficient.

A `<button>` doesn't need `role="button"`. A `<nav>` doesn't need `role="navigation"`. ARIA supplements, it doesn't replace.

### Common Patterns by Component

#### Button
```html
<!-- Standard — no ARIA needed -->
<button type="button">Save</button>

<!-- Icon-only — needs label -->
<button type="button" aria-label="Close dialog">
  <CloseIcon aria-hidden="true" />
</button>

<!-- Toggle — needs pressed state -->
<button type="button" aria-pressed="true">Bold</button>

<!-- Loading — needs disabled + busy -->
<button type="button" disabled aria-busy="true">
  <Spinner aria-hidden="true" /> Saving...
</button>
```

#### Dialog / Modal
```html
<div role="dialog" aria-modal="true" aria-labelledby="dialog-title" aria-describedby="dialog-desc">
  <h2 id="dialog-title">Confirm Delete</h2>
  <p id="dialog-desc">This action cannot be undone.</p>
  <!-- Focus trap: Tab cycles within dialog -->
  <!-- Escape: closes dialog -->
  <!-- On close: return focus to trigger element -->
</div>
```

#### Tabs
```html
<div role="tablist" aria-label="Settings sections">
  <button role="tab" id="tab-1" aria-selected="true" aria-controls="panel-1">General</button>
  <button role="tab" id="tab-2" aria-selected="false" aria-controls="panel-2" tabindex="-1">Security</button>
</div>
<div role="tabpanel" id="panel-1" aria-labelledby="tab-1">...</div>
<div role="tabpanel" id="panel-2" aria-labelledby="tab-2" hidden>...</div>
<!-- Arrow keys navigate between tabs -->
<!-- Tab key moves into panel content -->
```

#### Combobox / Autocomplete
```html
<label for="search">Search users</label>
<input id="search" role="combobox" aria-expanded="true" aria-controls="listbox-1"
       aria-activedescendant="option-3" aria-autocomplete="list" />
<ul id="listbox-1" role="listbox">
  <li id="option-1" role="option">Alice</li>
  <li id="option-2" role="option">Bob</li>
  <li id="option-3" role="option" aria-selected="true">Carol</li>
</ul>
```

#### Table (Sortable)
```html
<table aria-label="Validation results">
  <thead>
    <tr>
      <th scope="col" aria-sort="ascending">
        <button>Score <SortIcon /></button>
      </th>
      <th scope="col" aria-sort="none">
        <button>Date <SortIcon /></button>
      </th>
    </tr>
  </thead>
  <!-- ... -->
</table>
```

#### Alert / Status Messages
```html
<!-- Assertive: interrupts screen reader -->
<div role="alert">Error: Invalid email address</div>

<!-- Polite: waits for pause -->
<div role="status" aria-live="polite">3 results found</div>

<!-- Log: appended content (chat, activity feed) -->
<div role="log" aria-live="polite" aria-relevant="additions">...</div>
```

#### Tooltip
```html
<button aria-describedby="tooltip-1">
  <InfoIcon aria-hidden="true" />
</button>
<div id="tooltip-1" role="tooltip">Additional information here</div>
<!-- Show on hover AND focus -->
<!-- Escape dismisses -->
<!-- Content must be accessible without tooltip (supplementary only) -->
```

#### Navigation
```html
<nav aria-label="Main navigation">
  <ul>
    <li><a href="/" aria-current="page">Home</a></li>
    <li><a href="/dashboard">Dashboard</a></li>
  </ul>
</nav>
<!-- Use aria-current="page" on active link -->
<!-- Multiple <nav> elements need distinct aria-label -->
```

#### Progress
```html
<!-- Determinate -->
<div role="progressbar" aria-valuenow="65" aria-valuemin="0" aria-valuemax="100"
     aria-label="Upload progress">
  <div style="width: 65%"></div>
</div>

<!-- Indeterminate -->
<div role="progressbar" aria-label="Loading results">
  <Spinner />
</div>
```

## Keyboard Navigation Patterns

### Global
| Key | Action |
|-----|--------|
| `Tab` | Move to next focusable element |
| `Shift+Tab` | Move to previous focusable element |
| `Enter` / `Space` | Activate focused element |
| `Escape` | Close overlay (modal, dropdown, tooltip) |

### Composite Widgets (Arrow Key Navigation)
These use **roving tabindex**: one item has `tabindex="0"`, rest have `tabindex="-1"`. Arrow keys move focus between items.

| Widget | Arrow Keys | Wrap? |
|--------|-----------|-------|
| `tablist` | Left/Right (horizontal) or Up/Down (vertical) | Yes |
| `menu` | Up/Down | Yes |
| `listbox` | Up/Down | Optional |
| `radiogroup` | Up/Down or Left/Right | Yes |
| `toolbar` | Left/Right | Optional |
| `grid` / `treegrid` | All arrows | No |

### Focus Trap Pattern (Modals, Drawers)
```
1. On open: move focus to first focusable element (or dialog itself)
2. Tab cycles within the trapped container only
3. Shift+Tab cycles backward within container
4. Escape closes and returns focus to trigger element
5. Click outside (optional): closes and returns focus
```

### Skip Links
```html
<a href="#main-content" class="sr-only focus:not-sr-only focus:absolute focus:top-4 focus:left-4 focus:z-50 focus:bg-white focus:p-4">
  Skip to main content
</a>
```

## Screen Reader Testing Checklist

1. **Page title**: Descriptive and unique per page
2. **Heading hierarchy**: h1 → h2 → h3, no skipped levels, one h1 per page
3. **Landmarks**: `main`, `nav`, `header`, `footer`, `aside` all present and labeled
4. **Images**: All `<img>` have `alt` text (decorative images: `alt=""`)
5. **Links**: Link text is descriptive (not "click here" or "read more")
6. **Forms**: All inputs have associated labels, errors announced
7. **Dynamic content**: Live regions announce updates (`aria-live`)
8. **Focus management**: Focus moves logically, no focus traps outside modals
9. **Tables**: Headers use `<th>` with `scope`, complex tables use `id`/`headers`
10. **Custom widgets**: ARIA roles and states match expected patterns

## Common Violations and Fixes

| Violation | Fix |
|-----------|-----|
| `<div onClick>` for buttons | Use `<button>` or add `role="button"` + `tabindex="0"` + keyboard handler |
| Image without alt | Add `alt="description"` or `alt=""` for decorative |
| Input without label | Add `<label htmlFor>` or `aria-label` |
| Low contrast text | Increase contrast to 4.5:1 minimum (3:1 for large text) |
| Focus not visible | Add `focus-visible:ring-2 focus-visible:ring-offset-2` |
| Modal without focus trap | Implement focus trap on open, return focus on close |
| Auto-playing carousel | Add pause control, respect `prefers-reduced-motion` |
| Color-only status | Add icon or text alongside color indicator |
| Missing page language | Add `lang="en"` to `<html>` |
| Timeout without warning | Warn before timeout, allow extension, or remove time limit |
| Custom select without ARIA | Use `role="combobox"` + `role="listbox"` + `aria-expanded` |
| Toast disappears too fast | Minimum 5 seconds, add close button, use `role="status"` |
