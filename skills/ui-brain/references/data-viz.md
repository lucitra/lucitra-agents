# Data Visualization Reference

## Chart Selection Guide

| Data Relationship | Chart Type | When to Use |
|-------------------|-----------|-------------|
| Trend over time | **Line** | Continuous data, multiple series, time-based x-axis |
| Comparison | **Bar** (vertical) | Comparing categories, discrete values |
| Ranking | **Bar** (horizontal) | Sorted values, long category labels |
| Part of whole | **Pie** / **Donut** | ≤6 segments, percentages that sum to 100% |
| Distribution | **Area** (stacked) | Volume over time, composition changes |
| Correlation | **Scatter** | Two numeric variables, pattern detection |
| Multi-dimensional | **Radar** | 3–8 metrics per entity, comparison profiles |
| Density / Intensity | **Heatmap** | Two categorical axes + intensity value |
| Flow / Progress | **Funnel** | Step-by-step conversion rates |

### Rules
- Never use pie charts with >6 segments — use horizontal bar instead
- Never use 3D charts — they distort perception
- Always start y-axis at 0 for bar charts (line charts can truncate if labeled)
- Use consistent colors across related charts
- Sort bar charts by value (not alphabetically) unless there's a natural order

## Recharts Patterns

Recharts is used in the Validate Dashboard. All examples use the composable API.

### Line Chart
```tsx
import { LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip, ResponsiveContainer } from 'recharts';

<ResponsiveContainer width="100%" height={300}>
  <LineChart data={data} margin={{ top: 5, right: 20, left: 0, bottom: 5 }}>
    <CartesianGrid strokeDasharray="3 3" stroke="var(--border-default)" />
    <XAxis
      dataKey="date"
      tick={{ fontSize: 12 }}
      tickFormatter={(val) => formatDate(val)}
    />
    <YAxis tick={{ fontSize: 12 }} />
    <Tooltip
      contentStyle={{
        backgroundColor: 'var(--bg-surface-elevated)',
        border: '1px solid var(--border-default)',
        borderRadius: 8,
      }}
    />
    <Line
      type="monotone"
      dataKey="score"
      stroke="var(--accent)"
      strokeWidth={2}
      dot={false}
      activeDot={{ r: 4 }}
    />
  </LineChart>
</ResponsiveContainer>
```

### Bar Chart
```tsx
<ResponsiveContainer width="100%" height={300}>
  <BarChart data={data} margin={{ top: 5, right: 20, left: 0, bottom: 5 }}>
    <CartesianGrid strokeDasharray="3 3" vertical={false} />
    <XAxis dataKey="category" tick={{ fontSize: 12 }} />
    <YAxis tick={{ fontSize: 12 }} />
    <Tooltip />
    <Bar dataKey="value" fill="var(--accent)" radius={[4, 4, 0, 0]} />
  </BarChart>
</ResponsiveContainer>
```

### Area Chart (Stacked)
```tsx
<ResponsiveContainer width="100%" height={300}>
  <AreaChart data={data}>
    <CartesianGrid strokeDasharray="3 3" />
    <XAxis dataKey="date" />
    <YAxis />
    <Tooltip />
    <Area type="monotone" dataKey="passed" stackId="1" fill="#22c55e" fillOpacity={0.6} stroke="#22c55e" />
    <Area type="monotone" dataKey="warned" stackId="1" fill="#eab308" fillOpacity={0.6} stroke="#eab308" />
    <Area type="monotone" dataKey="failed" stackId="1" fill="#ef4444" fillOpacity={0.6} stroke="#ef4444" />
  </AreaChart>
</ResponsiveContainer>
```

### Key Recharts Patterns
- **Always wrap in `<ResponsiveContainer>`** — never set fixed dimensions on chart
- Use `margin` on chart component to prevent axis labels from clipping
- Use `dot={false}` on Line for clean appearance, `activeDot` for hover
- Use `radius={[4, 4, 0, 0]}` on Bar for rounded top corners
- Custom tooltip: `<Tooltip content={<CustomTooltip />}` for branded styling
- Use `tickFormatter` for date/number formatting, not raw data values

## Color Palettes for Data

### Categorical (Colorblind-Safe)
For distinguishing different series/categories:
```
Blue:    #2563eb
Orange:  #ea580c
Green:   #16a34a
Purple:  #9333ea
Teal:    #0d9488
Pink:    #db2777
```

### Sequential (Single Hue)
For representing magnitude/intensity:
```
Light → Dark: #dbeafe → #93c5fd → #3b82f6 → #1d4ed8 → #1e3a8a
```

### Diverging (Two Hues)
For representing deviation from center (e.g., above/below threshold):
```
Red → Neutral → Green: #ef4444 → #f87171 → #d4d4d8 → #4ade80 → #22c55e
```

### Semantic Score Colors
Used consistently across Lucitra for validation scores:
```
Pass/Good:    #22c55e (green-500)
Warning:      #eab308 (yellow-500)
Fail/Bad:     #ef4444 (red-500)
Neutral/NA:   #a1a1aa (zinc-400)
```

### Rules
- Never rely on color alone — add labels, patterns, or icons
- Test with colorblind simulation (Chrome DevTools → Rendering → Vision)
- Maximum 6–8 colors in one chart before it becomes unreadable
- Use opacity/lightness variations of one color for ordered data

## Three.js Integration Patterns

Used in Studio for 3D scene preview/validation visualization.

### Core Setup
```tsx
import { Canvas } from '@react-three/fiber';
import { OrbitControls, Environment, Grid } from '@react-three/drei';

<Canvas camera={{ position: [5, 5, 5], fov: 50 }}>
  <ambientLight intensity={0.5} />
  <directionalLight position={[10, 10, 5]} intensity={1} />
  <OrbitControls makeDefault />
  <Grid args={[10, 10]} cellSize={1} cellColor="#6b7280" />
  <Environment preset="studio" />
  {children}
</Canvas>
```

### Performance Rules
- Use `instancedMesh` for >100 identical geometries
- Dispose of geometries and materials on unmount
- Use `useFrame` sparingly — avoid heavy computation per frame
- Enable frustum culling (default in three.js)
- Use LOD (Level of Detail) for complex scenes

## Dashboard Layout Patterns

### KPI Row + Detail Grid
```tsx
<div className="space-y-6">
  {/* KPI summary row */}
  <div className="grid grid-cols-2 md:grid-cols-4 gap-4">
    <StatCard label="Total Runs" value={1234} trend="+12%" />
    <StatCard label="Pass Rate" value="94.2%" trend="+3.1%" />
    <StatCard label="Avg Score" value={87.5} trend="-1.2%" />
    <StatCard label="Active Jobs" value={7} />
  </div>

  {/* Chart + table grid */}
  <div className="grid grid-cols-1 lg:grid-cols-2 gap-6">
    <Card>
      <CardHeader>Score Trend</CardHeader>
      <CardBody><LineChart /></CardBody>
    </Card>
    <Card>
      <CardHeader>Category Breakdown</CardHeader>
      <CardBody><BarChart /></CardBody>
    </Card>
  </div>

  {/* Full-width detail table */}
  <Card>
    <CardHeader>Recent Validations</CardHeader>
    <CardBody><DataTable /></CardBody>
  </Card>
</div>
```

### Stat Card Pattern
```tsx
function StatCard({ label, value, trend, trendDirection }) {
  return (
    <div className="p-4 rounded-lg border bg-surface">
      <p className="text-sm text-muted">{label}</p>
      <p className="text-2xl font-semibold mt-1">{value}</p>
      {trend && (
        <p className={cn("text-sm mt-1", trendDirection === 'up' ? 'text-green-500' : 'text-red-500')}>
          {trend}
        </p>
      )}
    </div>
  );
}
```

## Animation and Transition Guidelines

### Chart Transitions
- Entry animation: 300–500ms ease-out
- Data update: 200–300ms ease-in-out
- Hover effects: 150ms ease

### Recharts Animation
```tsx
<Line animationDuration={500} animationEasing="ease-out" />
<Bar animationDuration={400} animationBegin={0} />
```

### Reduced Motion
```tsx
const prefersReducedMotion = window.matchMedia('(prefers-reduced-motion: reduce)').matches;

<Line animationDuration={prefersReducedMotion ? 0 : 500} />
```

### Rules
- Always respect `prefers-reduced-motion`
- Don't animate on initial render if data is already available
- Use animation to draw attention, not to decorate
- Tooltip/hover responses should feel instant (<150ms)
