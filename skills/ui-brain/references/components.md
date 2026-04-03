# Component Patterns Reference

60+ component patterns organized by category. Each includes: when to use, anatomy, key props, accessibility, and anti-patterns.

---

## Layout Components

### Container
**When**: Constrain content width and center it horizontally.
**Anatomy**: Wrapper with `max-width` + horizontal padding + `margin: 0 auto`.
**Props**: `maxWidth` (sm/md/lg/xl/2xl/full), `padding` (responsive).
**Tailwind**: `max-w-7xl mx-auto px-4 sm:px-6 lg:px-8`
**Anti-patterns**: Nesting containers (double padding). Using fixed widths instead of max-width.

### Grid
**When**: 2D layout — rows AND columns. Cards, dashboards, form layouts.
**Anatomy**: Parent with `display: grid`, children as grid items.
**Props**: `cols` (responsive), `gap`, `alignItems`, `justifyItems`.
**Tailwind**: `grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6`
**Anti-patterns**: Using grid for single-axis layout (use flex). Fixed column counts without responsive variants.

### Stack (Flex Column)
**When**: Vertical list of elements with consistent spacing.
**Anatomy**: Flex container with `flex-direction: column` and `gap`.
**Tailwind**: `flex flex-col gap-4`
**Anti-patterns**: Using margin-bottom on children instead of gap. Mixing spacing values.

### Divider
**When**: Visual separation between sections or groups.
**Anatomy**: `<hr>` or decorative `<div>` with border.
**Tailwind**: `border-t border-zinc-200 dark:border-zinc-700`
**Accessibility**: Use `<hr>` for semantic separation, `role="separator"` for decorative.
**Anti-patterns**: Using margin for separation instead of a visible divider when groups need clear boundaries.

### AspectRatio
**When**: Maintain consistent proportions for media containers.
**Tailwind**: `aspect-video` (16:9), `aspect-square` (1:1), `aspect-[4/3]` (custom).
**Anti-patterns**: Padding-bottom hack when `aspect-ratio` CSS property is available.

---

## Navigation Components

### Navbar
**When**: Primary site/app navigation, always visible.
**Anatomy**: Logo + nav links + actions (search, profile, CTA). Fixed or sticky at top.
**Accessibility**: `<nav aria-label="Main navigation">`, `aria-current="page"` on active link.
**Responsive**: Horizontal links → hamburger menu at mobile breakpoint.
**Anti-patterns**: Using div instead of nav. No mobile menu. Active state missing.

### Sidebar
**When**: App navigation with many sections, dashboard layouts.
**Anatomy**: Vertical nav links, optional groups/headers, collapse toggle.
**Accessibility**: `<nav aria-label="Sidebar">`, proper link/button roles.
**Responsive**: Persistent on desktop, slide-over drawer on mobile.
**Anti-patterns**: Sidebar not collapsible. No indication of current section. Horizontal scroll instead of collapse.

### Breadcrumb
**When**: Show location in hierarchy, enable navigation to parent pages.
**Anatomy**: Ordered list of links separated by chevrons/slashes.
**Accessibility**: `<nav aria-label="Breadcrumb">`, `<ol>`, `aria-current="page"` on last item.
**Anti-patterns**: More than 4–5 levels deep (simplify hierarchy). Breadcrumbs that don't match actual page hierarchy.

### Tabs
**When**: Switch between related views within the same page context.
**Anatomy**: Tab list (buttons) + tab panels (content areas). Only one panel visible.
**Accessibility**: `role="tablist"`, `role="tab"`, `role="tabpanel"`, arrow key navigation, `aria-selected`.
**Anti-patterns**: Using tabs for unrelated content (use nav links). More than 7 tabs (use dropdown or different pattern). Tabs that navigate to different pages (use links).

### Pagination
**When**: Navigate through paged data (tables, search results).
**Anatomy**: Previous/Next buttons + page numbers + optional page size selector.
**Accessibility**: `<nav aria-label="Pagination">`, current page indicated.
**Anti-patterns**: No keyboard support. Page count without showing total items. Loading entire dataset then paginating client-side.

### Steps / Stepper
**When**: Multi-step wizard or progress indicator.
**Anatomy**: Numbered/labeled steps with status (complete, current, upcoming).
**Accessibility**: `aria-current="step"` on active step, progress communicated.
**Anti-patterns**: No way to go back. Steps that don't indicate completion status.

---

## Data Entry Components

### Input
**When**: Collect single-line text (names, emails, numbers, search).
**Anatomy**: Label + input field + optional help text + optional error message.
**Accessibility**: `<label htmlFor>` pairing, `aria-describedby` for help/error, `aria-invalid` for errors.
**Props**: `type` (text/email/number/password/search/tel/url), `placeholder`, `disabled`, `required`.
**States**: Default, focused, filled, error, disabled, readonly.
**Anti-patterns**: Placeholder as label. No visible label. No error state. Input without associated label.

### Textarea
**When**: Collect multi-line text (comments, descriptions, messages).
**Anatomy**: Same as Input but multi-line. Optional character count.
**Accessibility**: Same as Input. Add `aria-describedby` for character limit.
**Anti-patterns**: Fixed height that hides content. No resize handle. Missing character count for limited fields.

### Select
**When**: Choose from a predefined list (≤10 options where all should be visible).
**Anatomy**: Label + select trigger + dropdown list + optional search.
**Accessibility**: Native `<select>` preferred. Custom: `role="combobox"` + `role="listbox"`.
**Anti-patterns**: Custom select without keyboard support. Select for yes/no (use checkbox/switch). Select with >50 items without search.

### Checkbox
**When**: Toggle one or more independent options. Agree to terms. Multi-select.
**Anatomy**: Checkbox input + label. Optional indeterminate state for parent checkbox.
**Accessibility**: `<input type="checkbox">` + `<label>`. Group with `<fieldset>` + `<legend>`.
**Anti-patterns**: Using checkboxes for mutually exclusive options (use radio). Checkbox without label.

### Radio Group
**When**: Choose exactly one option from a visible set (2–5 options).
**Anatomy**: Fieldset + legend + radio buttons with labels.
**Accessibility**: `<fieldset>` + `<legend>` + `<input type="radio">` with same `name`. Arrow key navigation.
**Anti-patterns**: Radio for toggle (use switch). Radio for >5 options (use select). Single radio button alone.

### Switch / Toggle
**When**: Toggle a setting with immediate effect (no form submission needed).
**Anatomy**: Label + switch control. Visual on/off state.
**Accessibility**: `role="switch"` + `aria-checked`. Needs associated label.
**Anti-patterns**: Switch in a form that requires submission (use checkbox). No label explaining what it controls.

### Slider
**When**: Select a value from a continuous range (volume, price, rating).
**Anatomy**: Track + thumb + optional value label + optional min/max labels.
**Accessibility**: `role="slider"` + `aria-valuenow` + `aria-valuemin` + `aria-valuemax` + `aria-label`.
**Anti-patterns**: Slider for precise values (use number input). Slider without displaying current value.

### DatePicker
**When**: Select a date or date range.
**Anatomy**: Input trigger + calendar dropdown. Optional presets (today, this week, etc.).
**Accessibility**: Full keyboard navigation, arrow keys for calendar, Enter to select, Escape to close.
**Anti-patterns**: No keyboard support. No manual text entry fallback. Showing all months without navigation.

### FileUpload
**When**: Upload files (images, documents, data).
**Anatomy**: Drop zone + browse button + file list with progress + validation feedback.
**Accessibility**: `<input type="file">` with label. Drag-and-drop as enhancement, not requirement.
**Anti-patterns**: No file type validation. No size limits. No upload progress. No way to remove files before submission.

### Form
**When**: Collect and validate a group of related inputs.
**Anatomy**: `<form>` + field groups + validation + submit/cancel buttons.
**Accessibility**: Labels on all fields, error summary, inline errors linked with `aria-describedby`, focus management on submit.
**Anti-patterns**: No client-side validation. Error messages not associated with fields. Submit button far from form. No loading state during submission.

---

## Data Display Components

### Table
**When**: Display structured, comparable data with multiple attributes per item.
**Anatomy**: thead (column headers) + tbody (data rows) + optional tfoot. Sort controls, filter bar, pagination.
**Accessibility**: `<th scope="col">` for headers, `aria-sort` for sortable columns, proper caption/label.
**Props**: `data`, `columns`, `sortable`, `selectable`, `pagination`.
**Responsive**: Horizontal scroll wrapper, priority columns, or card view on mobile.
**Anti-patterns**: Table for layout (never). No headers. Massive tables without pagination. No empty/loading state.

### Card
**When**: Display a self-contained unit of content (product, article, user profile, metric).
**Anatomy**: Container + optional image/media + header + body + footer/actions.
**Accessibility**: If clickable, entire card should be the link target. Use `article` element.
**Anti-patterns**: Cards with too many actions. Cards of inconsistent sizes in a grid. Card without clear hierarchy.

### Badge
**When**: Small status indicator or count. Tags, labels, notification counts.
**Anatomy**: Inline element with background color and text.
**Variants**: Status (success/warning/error/info), count (notification number), label (category tag).
**Accessibility**: Convey meaning beyond color — include text or `aria-label`.
**Anti-patterns**: Too many badges creating visual noise. Using badge for long text (use tag/chip).

### Avatar
**When**: Represent a user or entity with an image, initials, or icon.
**Anatomy**: Circular image/initials + optional status dot + optional fallback.
**Sizes**: xs (24px), sm (32px), md (40px), lg (48px), xl (64px).
**Accessibility**: `alt="User Name"` on image. Decorative avatars: `alt=""`.
**Anti-patterns**: Avatar without fallback for missing images. Tiny avatars (<24px) that can't be distinguished.

### Tag / Chip
**When**: Display metadata, filter criteria, or selected items. Often removable.
**Anatomy**: Label + optional icon + optional remove button.
**Accessibility**: Remove button needs `aria-label="Remove {tag name}"`.
**Anti-patterns**: Tags that look clickable but aren't. Too many tags overflowing without handling.

### Tooltip
**When**: Provide supplementary information on hover/focus. NOT for essential information.
**Anatomy**: Trigger element + positioned tooltip text.
**Accessibility**: `aria-describedby` linking trigger to tooltip. Show on focus, not just hover. Escape dismisses.
**Anti-patterns**: Essential content in tooltip (use visible text). Tooltip on mobile (hard to hover). Tooltip that covers the element it describes.

### Popover
**When**: Rich content on click/focus — forms, detailed info, mini-UIs.
**Anatomy**: Trigger button + positioned panel with content. Click outside or Escape to close.
**Accessibility**: `aria-haspopup`, `aria-expanded`, focus management into/out of popover.
**Anti-patterns**: Using popover for simple text (use tooltip). Popover that's too large (use modal/drawer).

### Accordion
**When**: Collapse/expand sections to manage information density. FAQs, settings groups.
**Anatomy**: Multiple panels with header (trigger) + collapsible body.
**Accessibility**: Header is `<button>` with `aria-expanded` + `aria-controls`. Optional: only one open at a time.
**Anti-patterns**: Hiding essential content in collapsed sections. Accordion with single item (use details/summary).

### Timeline
**When**: Display chronological events or history. Activity logs, order status, version history.
**Anatomy**: Vertical line + timestamped events with icons/status indicators.
**Accessibility**: Use `<ol>` or `<ul>` for the list. Timestamp and event description visible.
**Anti-patterns**: Timeline without dates/times. Horizontal timeline on small screens (use vertical).

### Stat / Metric
**When**: Display a key number with label and optional trend. Dashboard KPIs.
**Anatomy**: Label (what) + value (number) + optional trend (change) + optional icon.
**Accessibility**: Label clearly describes the metric. Trend includes direction (up/down) not just color.
**Anti-patterns**: Stat without context (what does the number mean?). Too many decimals. Trend arrow with no sr-only direction text.

### List
**When**: Display items in sequence. Simple text items, settings, menu items.
**Anatomy**: `<ul>` or `<ol>` + `<li>` items. Optional icons, descriptions, actions.
**Accessibility**: Semantic list markup. If interactive, items need proper roles.
**Anti-patterns**: Using `<div>` instead of `<ul>`/`<ol>`. List without visual bullets/separators that looks like paragraphs.

---

## Feedback Components

### Alert
**When**: Persistent, important messages that need attention. Errors, warnings, success confirmations.
**Anatomy**: Icon + title (optional) + description + optional action + optional dismiss.
**Variants**: `info` (blue), `success` (green), `warning` (yellow), `error` (red).
**Accessibility**: `role="alert"` for urgent. `role="status"` for non-urgent. Don't use for toast-like ephemeral messages.
**Anti-patterns**: Alert without icon (color-only). Too many alerts stacked. Using alert for trivial information.

### Toast / Notification
**When**: Ephemeral feedback for user actions. "Saved successfully", "Link copied".
**Anatomy**: Auto-dismissing message + optional action (undo) + optional close button.
**Timing**: Minimum 5 seconds visible. Add 1s per 120 words of content.
**Accessibility**: `role="status"` + `aria-live="polite"`. Must not block content. Must be dismissible.
**Anti-patterns**: Toasts for errors (use inline alert). Multiple simultaneous toasts. Toast with too much content. No keyboard dismiss.

### Modal / Dialog
**When**: Focused task that requires attention. Confirmations, short forms, critical decisions.
**Anatomy**: Overlay + centered dialog with title + body + actions (confirm/cancel).
**Accessibility**: `role="dialog"` + `aria-modal="true"` + `aria-labelledby` + focus trap + Escape to close + return focus on close.
**Anti-patterns**: Modal for content that should be a page. Modal inside modal. Long forms in modals (>5 fields). No way to dismiss.

### Drawer / Sheet
**When**: Contextual panel sliding from edge. Detail view, filters, settings, mobile navigation.
**Anatomy**: Overlay + side panel (left, right, bottom) with content area.
**Accessibility**: Same as modal — focus trap, Escape, return focus. `aria-label` on the drawer.
**Anti-patterns**: Drawer for critical actions (use modal). Full-screen drawer (use a page). No close mechanism.

### Progress Bar
**When**: Show progress of a determinate operation (upload, installation, multi-step process).
**Anatomy**: Track + fill bar + optional label + optional percentage.
**Accessibility**: `role="progressbar"` + `aria-valuenow` + `aria-valuemin` + `aria-valuemax` + `aria-label`.
**Anti-patterns**: Progress bar for indeterminate operations (use spinner). Progress that jumps backward. No text alternative.

### Skeleton
**When**: Placeholder while content loads. Shows the expected shape of the UI.
**Anatomy**: Gray animated blocks matching the shape of text lines, images, avatars, buttons.
**Accessibility**: `aria-busy="true"` on container. `aria-hidden="true"` on skeleton elements.
**Anti-patterns**: Skeleton that doesn't match actual content layout. Skeleton for already-loaded content. Indefinite skeleton (add timeout + error state).

### Spinner
**When**: Indeterminate loading — you don't know how long it'll take.
**Anatomy**: Rotating indicator + optional label.
**Accessibility**: `role="progressbar"` (indeterminate) or `aria-label="Loading"`. `aria-busy="true"` on affected area.
**Anti-patterns**: Spinner for known-duration operations (use progress bar). Spinner without label. Full-page spinner blocking all interaction.

### Empty State
**When**: No data to display — first use, zero results, cleared content.
**Anatomy**: Illustration/icon + message explaining why it's empty + CTA to create/add content.
**Accessibility**: Message is programmatically accessible, CTA is a real button/link.
**Anti-patterns**: Blank white space with no explanation. Empty state without action. Generic "No data" without context.

---

## Action Components

### Button
**When**: Trigger an action (submit, save, delete, open). NOT for navigation (use links).
**Anatomy**: Label text + optional leading/trailing icon.
**Variants**: Primary (filled), Secondary (outlined), Ghost (text-only), Destructive (red).
**Sizes**: sm (32px height), md (40px), lg (48px).
**States**: Default, hover, active, focus-visible, disabled, loading.
**Accessibility**: Native `<button>`. Icon-only: `aria-label`. Loading: `disabled` + `aria-busy="true"`.
**Anti-patterns**: Button styled as link. Link styled as button. Multiple primary buttons in one view. Disabled without explanation. Button without type attribute (defaults to submit in forms).

### IconButton
**When**: Action with icon only — toolbar buttons, close buttons, collapse toggles.
**Anatomy**: Icon + invisible label.
**Accessibility**: **Must have `aria-label`**. Minimum 44×44px touch target.
**Anti-patterns**: Icon button without label. Tiny icon buttons. Ambiguous icons without tooltip.

### ButtonGroup
**When**: Group of related actions (segmented control, toolbar section).
**Anatomy**: Adjacent buttons with connected borders.
**Accessibility**: `role="group"` + `aria-label`. For toggle groups: `role="radiogroup"`.
**Anti-patterns**: Too many buttons (>5) in a group. Mixing primary and destructive in one group.

### Dropdown Menu
**When**: List of actions triggered from a single button. Context menus, "more" menus.
**Anatomy**: Trigger button + positioned menu list + menu items (optionally grouped with separators).
**Accessibility**: `aria-haspopup="menu"` + `aria-expanded` on trigger. `role="menu"` + `role="menuitem"`. Arrow key navigation. Enter/Space to activate.
**Anti-patterns**: Dropdown for navigation (use links/nav). Dropdown with >10 items without grouping. Nested dropdowns.

### FAB (Floating Action Button)
**When**: Primary action that should always be accessible. Mobile "compose" or "add" action.
**Anatomy**: Circular button + icon, fixed position (usually bottom-right).
**Accessibility**: `aria-label` required. Should not cover content.
**Anti-patterns**: Multiple FABs on one screen. FAB for secondary actions. FAB on desktop (use regular buttons).

---

## Media Components

### Image
**When**: Display photographs, illustrations, diagrams.
**Anatomy**: `<img>` or Next.js `<Image>` with alt text, dimensions, and lazy loading.
**Accessibility**: Descriptive `alt` text. Decorative images: `alt=""`. Complex images: `aria-describedby` linking to longer description.
**Anti-patterns**: Missing alt text. Missing width/height (causes layout shift). Loading all images eagerly. Serving unoptimized images.

### Video
**When**: Display video content (demos, tutorials, product showcases).
**Anatomy**: Video element + controls + optional poster image + captions track.
**Accessibility**: Captions required. Controls must be keyboard accessible. No autoplay with audio.
**Anti-patterns**: Autoplay with sound. Video without controls. Missing captions. Inline autoplay video that wastes bandwidth on mobile.

### Carousel / Slider
**When**: Browse through multiple items (images, testimonials, products) in constrained space.
**Anatomy**: Visible item(s) + prev/next controls + pagination dots.
**Accessibility**: `aria-roledescription="carousel"` + `aria-label`. Controls: `aria-label="Previous/Next slide"`. Auto-advance: pause on hover/focus, respect prefers-reduced-motion.
**Anti-patterns**: Auto-advancing without pause control. No keyboard controls. Carousel as the only way to access content. More than 8–10 slides.

### Gallery
**When**: Display a collection of images. Portfolio, product images, photo gallery.
**Anatomy**: Grid of thumbnails + lightbox/expanded view + navigation.
**Accessibility**: Each image needs alt text. Lightbox follows modal accessibility pattern.
**Anti-patterns**: Gallery without lightbox (small images hard to see). Missing loading states for images.

---

## Chart Components

See `references/data-viz.md` for detailed chart patterns. Summary:

### Line Chart
**When**: Show trends over time with continuous data. Multiple series comparison.

### Bar Chart
**When**: Compare discrete categories. Vertical for few categories, horizontal for many or long labels.

### Area Chart
**When**: Show volume/composition over time. Stacked areas for part-of-whole comparison.

### Pie / Donut Chart
**When**: Show part-of-whole for ≤6 segments. Donut preferred (center for summary label).

### Radar Chart
**When**: Compare 3–8 metrics across entities. Validation score profiles.

### Heatmap
**When**: Show intensity across two categorical dimensions. Schedule views, correlation matrices.

**Accessibility for all charts**: Provide data table alternative, use aria-label on chart container, colorblind-safe palettes, tooltips for exact values.
