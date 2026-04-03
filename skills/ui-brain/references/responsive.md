# Responsive Design Reference

## Breakpoint System

Tailwind CSS default breakpoints (mobile-first — styles apply at breakpoint and above):

| Prefix | Min Width | Target |
|--------|-----------|--------|
| (none) | 0px | Mobile phones (portrait) |
| `sm` | 640px | Large phones (landscape) |
| `md` | 768px | Tablets (portrait) |
| `lg` | 1024px | Tablets (landscape), small laptops |
| `xl` | 1280px | Desktops |
| `2xl` | 1536px | Large desktops |

### How to Think About Breakpoints

Don't design for devices — design for content. Add a breakpoint when the layout breaks, not at arbitrary device widths. The Tailwind defaults cover most cases, but the content dictates when to shift layout.

### Mobile-First Methodology

**Always start with the smallest screen and add complexity upward.**

```tsx
// CORRECT: mobile-first
<div className="flex flex-col md:flex-row md:items-center gap-4 md:gap-6">
  <div className="w-full md:w-1/3">Sidebar</div>
  <div className="w-full md:w-2/3">Content</div>
</div>

// WRONG: desktop-first (overriding down)
<div className="flex flex-row items-center gap-6 max-md:flex-col max-md:items-start max-md:gap-4">
```

Why mobile-first:
1. Forces you to prioritize content
2. Progressive enhancement is more robust than graceful degradation
3. Less CSS (base = mobile, only additions at larger sizes)
4. Better performance on mobile (no overriding)

## Touch Target Sizes

| Element | Minimum Size | Recommended | Spacing |
|---------|-------------|-------------|---------|
| Buttons, links | 44×44px | 48×48px | 8px between targets |
| Icon buttons | 44×44px | 48×48px | 8px between targets |
| Checkboxes, radios | 44×44px | — | Label extends hit area |
| List items (tappable) | 48px height | 56px height | — |

```tsx
// Icon button with proper touch target
<button className="p-3 -m-3"> {/* 24px icon + 12px*2 padding = 48px */}
  <Icon size={24} />
</button>
```

## Responsive Typography

| Level | Mobile | md+ | lg+ |
|-------|--------|-----|-----|
| Display | text-3xl | text-4xl | text-5xl or text-6xl |
| H1 | text-2xl | text-3xl | text-3xl |
| H2 | text-xl | text-2xl | text-2xl |
| H3 | text-lg | text-xl | text-xl |
| Body | text-base | text-base | text-base |
| Small | text-sm | text-sm | text-sm |

```tsx
<h1 className="text-2xl md:text-3xl font-bold">Page Title</h1>
<p className="text-base leading-relaxed max-w-prose">Body text...</p>
```

### Line Length
- Optimal: 45–75 characters per line
- Tailwind: `max-w-prose` (65ch) for body text
- Never let text span full width on large screens

## Layout Patterns

### Stack to Grid
Single column on mobile, multi-column on larger screens.

```tsx
<div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
  <Card />
  <Card />
  <Card />
</div>
```

### Sidebar Collapse
Persistent sidebar on desktop, toggle overlay on mobile.

```tsx
// Desktop: sidebar + content side by side
// Mobile: sidebar as slide-over drawer
<div className="flex">
  <aside className="hidden lg:block w-64 shrink-0">
    <SidebarContent />
  </aside>
  <main className="flex-1 min-w-0">
    <button className="lg:hidden" onClick={toggleSidebar}>Menu</button>
    <Content />
  </main>
</div>

{/* Mobile drawer */}
{sidebarOpen && (
  <div className="fixed inset-0 z-50 lg:hidden">
    <div className="fixed inset-0 bg-black/50" onClick={closeSidebar} />
    <aside className="fixed inset-y-0 left-0 w-64 bg-white">
      <SidebarContent />
    </aside>
  </div>
)}
```

### Responsive Tables
Tables don't work well on mobile. Options:

**Option A: Horizontal scroll**
```tsx
<div className="overflow-x-auto -mx-4 px-4">
  <table className="min-w-[600px] w-full">...</table>
</div>
```

**Option B: Card layout on mobile**
```tsx
{/* Table on desktop, cards on mobile */}
<div className="hidden md:block">
  <Table data={data} />
</div>
<div className="md:hidden space-y-3">
  {data.map(item => <MobileCard key={item.id} data={item} />)}
</div>
```

**Option C: Priority columns (hide less important columns)**
```tsx
<table>
  <thead>
    <tr>
      <th>Name</th> {/* Always visible */}
      <th>Score</th> {/* Always visible */}
      <th className="hidden md:table-cell">Date</th>
      <th className="hidden lg:table-cell">Category</th>
    </tr>
  </thead>
</table>
```

### Responsive Navigation
```tsx
// Desktop: horizontal nav
// Mobile: hamburger menu
<nav className="flex items-center justify-between px-4 h-14">
  <Logo />
  <ul className="hidden md:flex items-center gap-6">
    <li><NavLink href="/">Home</NavLink></li>
    <li><NavLink href="/dashboard">Dashboard</NavLink></li>
  </ul>
  <button className="md:hidden" aria-label="Open menu" onClick={toggleMenu}>
    <MenuIcon />
  </button>
</nav>
```

### Content Reordering
Use `order` or flex direction changes to reorder content for different screens:

```tsx
<div className="flex flex-col-reverse md:flex-col">
  <aside>Filters</aside> {/* Below content on mobile, above on desktop */}
  <main>Results</main>
</div>
```

## Image and Video Patterns

### Responsive Images
```tsx
// Always set dimensions to prevent layout shift
<img
  src="/image.jpg"
  alt="Description"
  width={800}
  height={600}
  className="w-full h-auto rounded-lg"
  loading="lazy"
/>

// Next.js Image (preferred)
<Image
  src="/image.jpg"
  alt="Description"
  width={800}
  height={600}
  sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw"
  className="w-full h-auto"
/>
```

### Aspect Ratio Containers
```tsx
// Tailwind aspect ratio
<div className="aspect-video"> {/* 16:9 */}
  <video className="w-full h-full object-cover" />
</div>

<div className="aspect-square"> {/* 1:1 */}
  <img className="w-full h-full object-cover" />
</div>
```

### Responsive Video Embed
```tsx
<div className="aspect-video">
  <iframe
    src="https://www.youtube.com/embed/..."
    className="w-full h-full"
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
    allowFullScreen
  />
</div>
```

## Testing Checklist

1. Resize browser from 320px to 1920px — no horizontal overflow
2. Test real devices (or emulation): iPhone SE, iPhone 14, iPad, Desktop
3. Test both orientations (portrait and landscape)
4. Verify touch targets are at least 44×44px
5. Check that no content is hidden without a way to access it
6. Verify images don't overflow containers
7. Test with zoom at 200% — content should reflow, not require horizontal scroll
8. Check `prefers-reduced-motion` — animations respect user preference
