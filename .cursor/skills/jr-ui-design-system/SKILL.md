---
name: jr-ui-design-system
description: UI/UX design system for React + Tailwind CSS projects with glassmorphism, multi-theme support, premium sliders, and micro-interactions. Use when building new UI components, creating buttons, sliders, panels, forms, toasts, toolbars, or when the user asks about UI design, styling, theming, or layout patterns.
---

# JR UI Design System

Professional-grade UI design system for React + TypeScript + Tailwind CSS, optimized for dark-themed tool/editor interfaces.

**Stack**: React 18+, TypeScript, Tailwind CSS 3.4+, Zustand, Lucide React, `clsx` + `tailwind-merge`

```typescript
import { clsx, type ClassValue } from "clsx";
import { twMerge } from "tailwind-merge";
export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

For complete CSS code, theme definitions, Tailwind config, and component API details, see [reference.md](reference.md).

---

## 1. Design Tokens (CSS Variables)

All components reference CSS variables (not hardcoded colors). Override per `[data-theme]`.

| Category | Key Variables |
|----------|--------------|
| Surface | `--color-bg-primary` (#0f172a), `--color-bg-panel` (#1e293b), `--color-bg-input` (rgba(0,0,0,0.4)) |
| Border | `--color-border-default` (rgba(255,255,255,0.1)), `--color-border-hover` (rgba(255,255,255,0.2)) |
| Text | `--color-text-primary` (#f8fafc), `--color-text-secondary` (#94a3b8), `--color-text-muted` (#475569) |
| Accent | `--color-accent` (#3b82f6), `--color-accent-glow` (#60a5fa), `--color-accent-secondary` (#a855f7) |
| Status | `--color-success` (#22c55e), `--color-warning` (#f59e0b), `--color-error` (#ef4444), `--color-info` (#3b82f6) |
| Slider | `--slider-color` (defaults to `--color-accent`) |

**Spacing**: 4px base system (4, 8, 12, 16, 24, 32px).
**Radius**: `sm`=4px, `md`=6px, `lg`=8px, `xl`=12px, `full`=9999px.
**Font**: `'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif`. Sizes: xs=10, sm=12, base=14, lg=16, xl=18.

---

## 2. Theme System

Use CSS variables + `data-theme` attribute (not prop drilling).

- 5 themes: `dark`, `light`, `cyberpunk`, `nord`, `dracula`
- `ThemeProvider` wraps app, sets `data-theme` on `<html>`
- Each theme overrides CSS variables on `[data-theme="themeName"]`
- Components read variables automatically - no theme prop needed

---

## 3. Button Design

### Variants

| Variant | Style |
|---------|-------|
| **ghost** (default) | `bg-white/5 border border-white/5 hover:bg-white/10 hover:border-white/20 active:scale-90 backdrop-blur-sm transition-all duration-150` |
| **solid** | `bg-[var(--color-accent)] text-white hover:brightness-110 active:scale-90 shadow-[var(--shadow-glow-sm)]` |
| **outline** | `bg-transparent border border-[var(--color-border-default)] hover:border-[var(--color-accent)]/50 hover:text-[var(--color-accent)] active:scale-90` |
| **icon** | Same as ghost but square, `p-0 rounded-lg` for toolbar use |

### Active State (toggle buttons)

```
bg-gradient-to-br from-blue-500 via-blue-600 to-indigo-600
text-white border border-blue-400/40
shadow: multi-layer neon glow (see reference.md)
```

### Sizes

| Size | Height | Padding | Icon | Font |
|------|--------|---------|------|------|
| `sm` | 28px | px-2 | 14px | text-xs |
| `md` | 36px | px-3 | 16px | text-sm |
| `lg` | 44px | px-4 | 20px | text-base |

### Rules

- ALL buttons MUST have `focus-visible:ring-2 focus-visible:ring-[var(--color-accent)]/50`
- ALL buttons MUST have `disabled:opacity-40 disabled:pointer-events-none`
- Icon-only buttons MUST have `aria-label`
- Press feedback: `active:scale-90` with `transition-all duration-150`

---

## 4. Slider Design

### Variants

| Variant | Track | Thumb | Use Case |
|---------|-------|-------|----------|
| `default` | 6px height | 14x14px circle | Volume, opacity, general |
| `compact` | 4px height | 10x10px circle | Inline, space-tight |
| `timeline` | 4px height | 4x32px bar | Playback scrubber |

### Key Behaviors

- **Color**: driven by CSS variable `--slider-color`, customizable per instance
- **Progress fill**: `linear-gradient` on track shows filled portion (always enabled)
- **Hover**: thumb scales to 1.2x with glow intensification
- **Vertical**: supported via `writing-mode: vertical-lr; direction: rtl`
- **Focus**: `focus-visible` outline ring
- **ARIA**: `role="slider"` with `aria-valuemin`, `aria-valuemax`, `aria-valuenow`, `aria-label`

---

## 5. Panel & Layout

### Architecture

```
+----------+-------------------------------+----------------+
| Left     | Center                        | Right Panel    |
| Toolbar  | +---------------------------+ | (resizable     |
| (48px    | | Top Toolbar               | |  280-600px)    |
|  icons)  | +---------------------------+ | +------------+ |
|          | | Main Viewport             | | | Tab Bar    | |
|          | | (aspect ratio container)  | | +------------+ |
|          | +---------------------------+ | | Tab Content| |
|          | | Bottom Panel (collapsible)| | |            | |
|          | +---------------------------+ | +------------+ |
+----------+-------------------------------+----------------+
```

### Glass Panel

```
backdrop-blur-lg + semi-transparent bg + 1px border(white/10) + shadow-glass
```

### Resizable Panels

- Right panel: 280–600px width, drag handle on left edge
- Bottom panel: 200–500px height, drag handle on top edge, collapse toggle
- Implement via `useResizablePanel` hook (mousedown → mousemove → mouseup)

### Left Toolbar

- `w-12`, vertical icon stack, fixed position, `z-40`
- Each button: `icon` variant, 36x36px

### Right Tab Panel

- Tab bar: icons + labels, `role="tablist"`
- Active tab: accent underline or breathing glow icon
- Content: scrollable with `custom-scrollbar`

---

## 6. Form Components

| Component | Behavior |
|-----------|----------|
| **NumberInput** | Local state while focused, commit on blur/Enter, hover-reveal step buttons (ChevronUp/Down), supports min/max/step |
| **DeferredInput** | Allows intermediate states (`"0."`, `"-"`, empty), commit on blur/Enter, optional `numeric` mode |

Shared style:
```
bg-[var(--color-bg-input)] border border-[var(--color-border-input)]
rounded-md text-sm px-2 h-7
focus:border-[var(--color-accent)]/50 focus:ring-1 focus:ring-[var(--color-accent)]/30
```

---

## 7. Feedback Components

### Toast

4 types: `info` (blue), `success` (emerald), `warning` (amber), `error` (red).
Each has `bg-{color}-900/90`, `border-{color}-500/50`, icon in `text-{color}-400`.
Position: fixed top-4 right-4, stacked gap-2. Animation: slide-in-right 0.3s.
State: Zustand store with `addToast` / `dismissToast`, auto-removal timer.
ARIA: `role="alert"`, `aria-live="polite"`.

### ProgressBar

States: active (accent gradient), completed (emerald), pending (white/20), inactive (none).
Supports marker overlays (absolute-positioned dots for cues/triggers).

### Tooltip

```
bg-[var(--color-bg-primary)]/95 backdrop-blur-sm text-xs px-2 py-1 rounded-md shadow-lg
```
300ms hover delay. Auto-position based on available space.

---

## 8. Animation

| Element | Animation | Duration |
|---------|-----------|----------|
| Button press | `scale(0.9)` | 150ms |
| Slider hover | thumb `scale(1.2)` + glow | 150ms |
| Panel appear | fadeIn or slideUp | 300-400ms |
| Tab icon active | breathing glow | 3s infinite |
| Toast enter | slide-in-right | 300ms |

**Reduced motion**: wrap ALL animations in `@media (prefers-reduced-motion: reduce)` with `animation-duration: 0.01ms !important`.

---

## 9. Scrollbar

- Width: 6px, transparent track, `rgba(255,255,255,0.1)` thumb, rounded
- Hover: thumb brightens to `rgba(255,255,255,0.2)`
- `.scrollbar-hide`: completely hidden but functional

---

## 10. Accessibility Checklist

- [ ] All interactive elements have `:focus-visible` ring (`2px solid var(--color-accent)`)
- [ ] Icon-only buttons have `aria-label`
- [ ] Sliders have `role="slider"` + `aria-valuemin/max/now` + `aria-label`
- [ ] Tab bars use `role="tablist/tab/tabpanel"` + `aria-selected`
- [ ] Toasts use `role="alert"` + `aria-live="polite"`
- [ ] `prefers-reduced-motion` disables all animations
- [ ] Color contrast meets WCAG AA (4.5:1 text, 3:1 large text/UI)
- [ ] Keyboard navigation: Tab for focus, Enter/Space for activation, Arrows for sliders/tabs, Escape to close

---

## Quick Reference

| Pattern | Classes |
|---------|---------|
| Glass bg | `bg-[var(--color-bg-panel)] backdrop-blur-lg border border-[var(--color-border-default)]` |
| Ghost btn | `bg-white/5 border border-white/5 hover:bg-white/10 active:scale-90 transition-all duration-150` |
| Active toggle | `bg-gradient-to-br from-blue-500 via-blue-600 to-indigo-600 text-white` |
| Input | `bg-[var(--color-bg-input)] border border-[var(--color-border-input)] rounded-md text-sm focus:border-[var(--color-accent)]/50` |
| Section label | `text-xs font-medium text-[var(--color-text-muted)] uppercase tracking-wider` |
| Card | `bg-[var(--color-bg-card)] rounded-lg border border-[var(--color-border-default)] p-3` |
| Divider | `border-t border-[var(--color-border-default)]` |
