# JR UI Design System - Reference

Complete CSS code, theme definitions, Tailwind config, and component implementation details.

---

## CSS Variables (Full Definition)

```css
:root {
  /* Surface */
  --color-bg-primary: #0f172a;
  --color-bg-panel: #1e293b;
  --color-bg-card: rgba(0, 0, 0, 0.3);
  --color-bg-input: rgba(0, 0, 0, 0.4);
  --color-bg-hover: rgba(255, 255, 255, 0.1);

  /* Border */
  --color-border-default: rgba(255, 255, 255, 0.1);
  --color-border-hover: rgba(255, 255, 255, 0.2);
  --color-border-input: rgba(255, 255, 255, 0.1);

  /* Text */
  --color-text-primary: #f8fafc;
  --color-text-secondary: #94a3b8;
  --color-text-muted: #475569;

  /* Accent */
  --color-accent: #3b82f6;
  --color-accent-glow: #60a5fa;
  --color-accent-secondary: #a855f7;
  --color-accent-secondary-glow: #c084fc;

  /* Status */
  --color-success: #22c55e;
  --color-warning: #f59e0b;
  --color-error: #ef4444;
  --color-info: #3b82f6;

  /* Slider */
  --slider-color: var(--color-accent);

  /* Shadows */
  --shadow-glass: 0 8px 32px rgba(0, 0, 0, 0.3);
  --shadow-glow-sm: 0 0 10px var(--color-accent);
  --shadow-glow-md: 0 0 20px var(--color-accent);
  --shadow-glow-lg: 0 0 10px var(--color-accent), 0 0 40px var(--color-accent);
  --shadow-active-button: 0 0 0 3px rgba(59,130,246,0.15),
                          0 0 20px -2px rgba(59,130,246,0.4),
                          0 4px 12px -2px rgba(99,102,241,0.3),
                          inset 0 1px 0 rgba(255,255,255,0.15);
}
```

---

## Theme Overrides

```css
[data-theme="dark"] {
  --color-bg-primary: #0f172a;
  --color-bg-panel: #1e293b;
  --color-bg-card: rgba(0, 0, 0, 0.3);
  --color-bg-input: rgba(0, 0, 0, 0.4);
  --color-bg-hover: rgba(255, 255, 255, 0.1);
  --color-border-default: rgba(255, 255, 255, 0.1);
  --color-border-hover: rgba(255, 255, 255, 0.2);
  --color-text-primary: #f8fafc;
  --color-text-secondary: #94a3b8;
  --color-text-muted: #475569;
  --color-accent: #3b82f6;
  --color-accent-glow: #60a5fa;
}

[data-theme="light"] {
  --color-bg-primary: #f8fafc;
  --color-bg-panel: rgba(255, 255, 255, 0.9);
  --color-bg-card: #ffffff;
  --color-bg-input: #f1f5f9;
  --color-bg-hover: rgba(0, 0, 0, 0.05);
  --color-border-default: #e2e8f0;
  --color-border-hover: #cbd5e1;
  --color-text-primary: #1e293b;
  --color-text-secondary: #64748b;
  --color-text-muted: #94a3b8;
  --color-accent: #3b82f6;
  --color-accent-glow: #60a5fa;
}

[data-theme="cyberpunk"] {
  --color-bg-primary: #050510;
  --color-bg-panel: rgba(10, 10, 26, 0.8);
  --color-bg-card: rgba(10, 10, 26, 0.4);
  --color-bg-input: rgba(10, 10, 26, 0.6);
  --color-bg-hover: rgba(59, 130, 246, 0.2);
  --color-border-default: rgba(168, 85, 247, 0.3);
  --color-border-hover: rgba(59, 130, 246, 0.4);
  --color-text-primary: #60a5fa;
  --color-text-secondary: rgba(168, 85, 247, 0.7);
  --color-text-muted: rgba(168, 85, 247, 0.6);
  --color-accent: #3b82f6;
  --color-accent-glow: #22d3ee;
  --color-accent-secondary: #a855f7;
}

[data-theme="nord"] {
  --color-bg-primary: #2e3440;
  --color-bg-panel: rgba(59, 66, 82, 0.9);
  --color-bg-card: rgba(59, 66, 82, 0.6);
  --color-bg-input: rgba(67, 76, 94, 0.6);
  --color-bg-hover: #434c5e;
  --color-border-default: #4c566a;
  --color-border-hover: rgba(136, 192, 208, 0.5);
  --color-text-primary: #eceff4;
  --color-text-secondary: rgba(216, 222, 233, 0.7);
  --color-text-muted: rgba(216, 222, 233, 0.5);
  --color-accent: #88c0d0;
  --color-accent-glow: #88c0d0;
}

[data-theme="dracula"] {
  --color-bg-primary: #282a36;
  --color-bg-panel: rgba(68, 71, 90, 0.9);
  --color-bg-card: rgba(68, 71, 90, 0.4);
  --color-bg-input: rgba(68, 71, 90, 0.6);
  --color-bg-hover: rgba(98, 114, 164, 0.5);
  --color-border-default: rgba(189, 147, 249, 0.3);
  --color-border-hover: rgba(255, 121, 198, 0.5);
  --color-text-primary: #f8f8f2;
  --color-text-secondary: rgba(189, 147, 249, 0.7);
  --color-text-muted: #6272a4;
  --color-accent: #bd93f9;
  --color-accent-glow: #ff79c6;
}
```

---

## ThemeProvider Implementation

```typescript
import { createContext, useContext, useState, useEffect, type ReactNode } from 'react';

export type ThemeMode = 'dark' | 'light' | 'cyberpunk' | 'nord' | 'dracula';

interface ThemeContextValue {
  mode: ThemeMode;
  setMode: (mode: ThemeMode) => void;
}

const ThemeContext = createContext<ThemeContextValue>({
  mode: 'dark',
  setMode: () => {},
});

export function ThemeProvider({
  children,
  defaultMode = 'dark',
}: {
  children: ReactNode;
  defaultMode?: ThemeMode;
}) {
  const [mode, setMode] = useState<ThemeMode>(() => {
    const saved = localStorage.getItem('theme-mode');
    return (saved as ThemeMode) || defaultMode;
  });

  useEffect(() => {
    document.documentElement.setAttribute('data-theme', mode);
    localStorage.setItem('theme-mode', mode);
  }, [mode]);

  return (
    <ThemeContext.Provider value={{ mode, setMode }}>
      {children}
    </ThemeContext.Provider>
  );
}

export function useTheme() {
  return useContext(ThemeContext);
}
```

---

## Button Component

```typescript
import { forwardRef, type ButtonHTMLAttributes, type ReactNode } from 'react';
import { cn } from '@/lib/utils';

interface ButtonProps extends ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'ghost' | 'solid' | 'outline' | 'icon';
  size?: 'sm' | 'md' | 'lg';
  active?: boolean;
  children?: ReactNode;
}

const variantClasses = {
  ghost: `bg-white/5 border border-white/5 text-[var(--color-text-secondary)]
          hover:bg-white/10 hover:border-white/20 hover:text-[var(--color-text-primary)]
          active:scale-90 active:bg-white/20 active:border-white/20
          backdrop-blur-sm`,
  solid: `bg-[var(--color-accent)] text-white border border-[var(--color-accent)]/40
          hover:brightness-110
          active:scale-90
          shadow-[var(--shadow-glow-sm)]`,
  outline: `bg-transparent border border-[var(--color-border-default)] text-[var(--color-text-secondary)]
            hover:border-[var(--color-accent)]/50 hover:text-[var(--color-accent)]
            active:scale-90`,
  icon: `p-0 rounded-lg bg-white/5 border border-white/5 text-[var(--color-text-secondary)]
         hover:bg-white/10 hover:border-white/10 hover:text-[var(--color-text-primary)]
         active:scale-90 active:bg-white/20
         backdrop-blur-sm`,
};

const activeClass = `bg-gradient-to-br from-blue-500 via-blue-600 to-indigo-600
  text-white border border-blue-400/40
  shadow-[0_0_0_3px_rgba(59,130,246,0.15),0_0_20px_-2px_rgba(59,130,246,0.4),0_4px_12px_-2px_rgba(99,102,241,0.3),inset_0_1px_0_rgba(255,255,255,0.15)]`;

const sizeClasses = {
  sm: 'h-7 px-2 text-xs gap-1',
  md: 'h-9 px-3 text-sm gap-1.5',
  lg: 'h-11 px-4 text-base gap-2',
};

const iconSizeClasses = {
  sm: 'h-7 w-7',
  md: 'h-9 w-9',
  lg: 'h-11 w-11',
};

export const Button = forwardRef<HTMLButtonElement, ButtonProps>(
  ({ variant = 'ghost', size = 'md', active, className, children, ...props }, ref) => (
    <button
      ref={ref}
      className={cn(
        'inline-flex items-center justify-center rounded-md font-medium',
        'transition-all duration-150',
        'focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-[var(--color-accent)]/50',
        'disabled:opacity-40 disabled:pointer-events-none',
        active ? activeClass : variantClasses[variant],
        variant === 'icon' ? iconSizeClasses[size] : sizeClasses[size],
        className,
      )}
      {...props}
    >
      {children}
    </button>
  ),
);
```

---

## Slider Component

```typescript
import { type InputHTMLAttributes } from 'react';
import { cn } from '@/lib/utils';

interface SliderProps extends Omit<InputHTMLAttributes<HTMLInputElement>, 'onChange'> {
  value: number;
  onChange: (value: number) => void;
  min?: number;
  max?: number;
  step?: number;
  variant?: 'default' | 'compact' | 'timeline';
  color?: string;
  orientation?: 'horizontal' | 'vertical';
  label?: string;
  showValue?: boolean;
}

const trackHeight = { default: 'h-1.5', compact: 'h-1', timeline: 'h-1' };
const thumbClass = {
  default: '[&::-webkit-slider-thumb]:w-3.5 [&::-webkit-slider-thumb]:h-3.5 [&::-webkit-slider-thumb]:rounded-full',
  compact: '[&::-webkit-slider-thumb]:w-2.5 [&::-webkit-slider-thumb]:h-2.5 [&::-webkit-slider-thumb]:rounded-full',
  timeline: '[&::-webkit-slider-thumb]:w-1 [&::-webkit-slider-thumb]:h-8 [&::-webkit-slider-thumb]:rounded-sm',
};

export function Slider({
  value,
  onChange,
  min = 0,
  max = 100,
  step = 1,
  variant = 'default',
  color,
  orientation = 'horizontal',
  label,
  showValue,
  className,
  ...props
}: SliderProps) {
  const pct = ((value - min) / (max - min)) * 100;
  const sliderColor = color || 'var(--color-accent)';

  return (
    <div className={cn('flex items-center gap-2', className)}>
      {label && <span className="text-xs text-[var(--color-text-secondary)] shrink-0">{label}</span>}
      <input
        type="range"
        min={min}
        max={max}
        step={step}
        value={value}
        onChange={(e) => onChange(Number(e.target.value))}
        role="slider"
        aria-valuemin={min}
        aria-valuemax={max}
        aria-valuenow={value}
        aria-label={label}
        data-orientation={orientation}
        className={cn(
          'jr-slider w-full appearance-none rounded-full outline-none border border-white/5',
          'cursor-pointer',
          trackHeight[variant],
          thumbClass[variant],
          '[&::-webkit-slider-thumb]:appearance-none [&::-webkit-slider-thumb]:cursor-pointer',
          '[&::-webkit-slider-thumb]:transition-all [&::-webkit-slider-thumb]:duration-150',
          'focus-visible:outline-2 focus-visible:outline-[var(--color-accent)] focus-visible:outline-offset-2',
        )}
        style={{
          '--slider-color': sliderColor,
          background: `linear-gradient(to right, ${sliderColor} 0%, ${sliderColor} ${pct}%, rgba(255,255,255,0.1) ${pct}%, rgba(255,255,255,0.1) 100%)`,
          ...(orientation === 'vertical' ? { writingMode: 'vertical-lr' as const, direction: 'rtl' as const } : {}),
        } as React.CSSProperties}
        {...props}
      />
      {showValue && <span className="text-xs text-[var(--color-text-secondary)] tabular-nums w-8 text-right shrink-0">{value}</span>}
    </div>
  );
}
```

### Slider CSS (add to global stylesheet)

```css
.jr-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  border-radius: 50%;
  cursor: pointer;
  background-color: var(--slider-color);
  box-shadow: 0 0 10px var(--slider-color);
  transition: transform 0.15s ease, box-shadow 0.15s ease;
}

.jr-slider::-webkit-slider-thumb:hover {
  transform: scale(1.2);
  box-shadow: 0 0 15px var(--slider-color);
}

.jr-slider::-moz-range-thumb {
  appearance: none;
  border-radius: 50%;
  border: none;
  cursor: pointer;
  background-color: var(--slider-color);
  box-shadow: 0 0 10px var(--slider-color);
  transition: transform 0.15s ease, box-shadow 0.15s ease;
}

.jr-slider::-moz-range-thumb:hover {
  transform: scale(1.2);
  box-shadow: 0 0 15px var(--slider-color);
}

.jr-slider[data-orientation="vertical"] {
  writing-mode: vertical-lr;
  direction: rtl;
  width: 8px;
  height: 96px;
}
```

---

## NumberInput Component

```typescript
import { useState, useRef, useEffect, type ChangeEvent, type KeyboardEvent } from 'react';
import { ChevronUp, ChevronDown } from 'lucide-react';
import { cn } from '@/lib/utils';

interface NumberInputProps {
  value: number;
  onChange: (value: number) => void;
  min?: number;
  max?: number;
  step?: number;
  className?: string;
  label?: string;
}

export function NumberInput({ value, onChange, min, max, step = 1, className, label }: NumberInputProps) {
  const [localValue, setLocalValue] = useState(String(value));
  const [isFocused, setIsFocused] = useState(false);
  const [isHovered, setIsHovered] = useState(false);

  useEffect(() => {
    if (!isFocused) setLocalValue(String(value));
  }, [value, isFocused]);

  const clamp = (v: number) => {
    if (min !== undefined) v = Math.max(min, v);
    if (max !== undefined) v = Math.min(max, v);
    return v;
  };

  const commit = () => {
    const parsed = parseFloat(localValue);
    if (!isNaN(parsed)) onChange(clamp(parsed));
    else setLocalValue(String(value));
  };

  const nudge = (dir: 1 | -1) => onChange(clamp(value + step * dir));

  return (
    <div
      className="relative inline-flex items-center"
      onMouseEnter={() => setIsHovered(true)}
      onMouseLeave={() => setIsHovered(false)}
    >
      {label && <span className="text-xs text-[var(--color-text-secondary)] mr-2">{label}</span>}
      <input
        type="text"
        inputMode="decimal"
        value={localValue}
        onChange={(e: ChangeEvent<HTMLInputElement>) => setLocalValue(e.target.value)}
        onFocus={() => setIsFocused(true)}
        onBlur={() => { setIsFocused(false); commit(); }}
        onKeyDown={(e: KeyboardEvent) => { if (e.key === 'Enter') commit(); }}
        className={cn(
          'bg-[var(--color-bg-input)] border border-[var(--color-border-input)]',
          'rounded-md text-sm text-right px-2 h-7 w-16 tabular-nums',
          'text-[var(--color-text-primary)]',
          'focus:border-[var(--color-accent)]/50 focus:ring-1 focus:ring-[var(--color-accent)]/30',
          'focus:outline-none transition-colors',
          className,
        )}
      />
      {isHovered && (
        <div className="absolute right-0 top-0 bottom-0 flex flex-col w-4">
          <button onClick={() => nudge(1)} className="flex-1 flex items-center justify-center text-[var(--color-text-muted)] hover:text-[var(--color-text-primary)] transition-colors" tabIndex={-1} aria-label="Increase">
            <ChevronUp size={10} />
          </button>
          <button onClick={() => nudge(-1)} className="flex-1 flex items-center justify-center text-[var(--color-text-muted)] hover:text-[var(--color-text-primary)] transition-colors" tabIndex={-1} aria-label="Decrease">
            <ChevronDown size={10} />
          </button>
        </div>
      )}
    </div>
  );
}
```

---

## Toast System

### Store (Zustand)

```typescript
import { create } from 'zustand';

export type ToastType = 'info' | 'success' | 'warning' | 'error';

export interface Toast {
  id: string;
  type: ToastType;
  title: string;
  message?: string;
  duration?: number;
}

interface ToastStore {
  toasts: Toast[];
  addToast: (toast: Omit<Toast, 'id'>) => void;
  dismissToast: (id: string) => void;
}

export const useToastStore = create<ToastStore>((set) => ({
  toasts: [],
  addToast: (toast) => {
    const id = crypto.randomUUID();
    set((s) => ({ toasts: [...s.toasts, { ...toast, id }] }));
    setTimeout(() => {
      set((s) => ({ toasts: s.toasts.filter((t) => t.id !== id) }));
    }, toast.duration ?? 4000);
  },
  dismissToast: (id) => set((s) => ({ toasts: s.toasts.filter((t) => t.id !== id) })),
}));

export const toast = {
  info: (title: string, message?: string) => useToastStore.getState().addToast({ type: 'info', title, message }),
  success: (title: string, message?: string) => useToastStore.getState().addToast({ type: 'success', title, message }),
  warning: (title: string, message?: string) => useToastStore.getState().addToast({ type: 'warning', title, message }),
  error: (title: string, message?: string) => useToastStore.getState().addToast({ type: 'error', title, message }),
};
```

### ToastContainer Component

```typescript
import { X, Info, CheckCircle, AlertTriangle, XCircle } from 'lucide-react';
import { useToastStore, type ToastType } from '@/stores/toastStore';

const styles: Record<ToastType, { bg: string; border: string; iconColor: string; titleColor: string }> = {
  info:    { bg: 'bg-blue-900/90',    border: 'border-blue-500/50',    iconColor: 'text-blue-400',    titleColor: 'text-blue-300' },
  success: { bg: 'bg-emerald-900/90', border: 'border-emerald-500/50', iconColor: 'text-emerald-400', titleColor: 'text-emerald-300' },
  warning: { bg: 'bg-amber-900/90',   border: 'border-amber-500/50',   iconColor: 'text-amber-400',   titleColor: 'text-amber-300' },
  error:   { bg: 'bg-red-900/90',     border: 'border-red-500/50',     iconColor: 'text-red-400',     titleColor: 'text-red-300' },
};

const icons: Record<ToastType, typeof Info> = {
  info: Info, success: CheckCircle, warning: AlertTriangle, error: XCircle,
};

export function ToastContainer() {
  const { toasts, dismissToast } = useToastStore();
  if (!toasts.length) return null;

  return (
    <div className="fixed top-4 right-4 z-[9999] flex flex-col gap-2">
      {toasts.map((t) => {
        const s = styles[t.type];
        const Icon = icons[t.type];
        return (
          <div key={t.id} role="alert" aria-live="polite"
            className={`${s.bg} ${s.border} backdrop-blur-lg border rounded-lg shadow-xl p-3 pr-10 min-w-[280px] max-w-[400px] animate-slide-in-right relative`}>
            <div className="flex items-start gap-3">
              <Icon size={18} className={s.iconColor} />
              <div className="flex-1 min-w-0">
                <div className={`text-sm font-medium ${s.titleColor}`}>{t.title}</div>
                {t.message && <div className="text-xs text-gray-300 mt-1">{t.message}</div>}
              </div>
            </div>
            <button onClick={() => dismissToast(t.id)} aria-label="Close"
              className="absolute top-2 right-2 p-1 rounded hover:bg-white/10 text-gray-400 hover:text-white transition-colors">
              <X size={14} />
            </button>
          </div>
        );
      })}
    </div>
  );
}
```

---

## Global CSS (index.css template)

```css
@import url('https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700&display=swap');

@tailwind base;
@tailwind components;
@tailwind utilities;

/* Glass utilities */
@layer utilities {
  .glass {
    @apply bg-gray-900/70 backdrop-blur-md border border-white/10 shadow-xl;
  }
  .glass-panel {
    @apply bg-gray-900/80 backdrop-blur-lg border border-white/10 shadow-2xl;
  }
  .glass-button {
    @apply bg-white/5 hover:bg-white/10 border border-white/5 hover:border-white/20 transition-all duration-300 backdrop-blur-sm;
  }
}

body {
  margin: 0;
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  background-color: var(--color-bg-primary);
  color: var(--color-text-primary);
}

/* Scrollbar */
.custom-scrollbar::-webkit-scrollbar { width: 6px; height: 6px; }
.custom-scrollbar::-webkit-scrollbar-track { background: transparent; }
.custom-scrollbar::-webkit-scrollbar-thumb { background: rgba(255,255,255,0.1); border-radius: 3px; }
.custom-scrollbar::-webkit-scrollbar-thumb:hover { background: rgba(255,255,255,0.2); }

.scrollbar-hide { -ms-overflow-style: none; scrollbar-width: none; }
.scrollbar-hide::-webkit-scrollbar { display: none; }

/* Slider - see Slider Component section for .jr-slider */

/* Animations */
@keyframes fadeIn {
  from { opacity: 0; }
  to { opacity: 1; }
}

@keyframes slideUp {
  from { transform: translateY(10px); opacity: 0; }
  to { transform: translateY(0); opacity: 1; }
}

@keyframes slide-in-right {
  from { transform: translateX(100%); opacity: 0; }
  to { transform: translateX(0); opacity: 1; }
}

@keyframes gentle-breathe {
  0%, 100% {
    filter: drop-shadow(0 0 3px var(--glow-color));
    opacity: 0.9;
  }
  50% {
    filter: drop-shadow(0 0 8px var(--glow-color)) drop-shadow(0 0 12px var(--glow-color));
    opacity: 1;
  }
}

.animate-slide-in-right { animation: slide-in-right 0.3s ease-out; }

.icon-breathe {
  --glow-color: var(--color-accent-glow);
  animation: gentle-breathe 3s ease-in-out infinite;
}

/* Reduced motion */
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
  }
}
```

---

## Tailwind Config

```javascript
/** @type {import('tailwindcss').Config} */
export default {
  content: [
    "./index.html",
    "./src/**/*.{js,ts,jsx,tsx}",
  ],
  theme: {
    extend: {
      colors: {
        'premium-dark': {
          900: '#0f172a',
          800: '#1e293b',
          700: '#334155',
          600: '#475569',
        },
        'neon-blue': {
          DEFAULT: '#3b82f6',
          glow: '#60a5fa',
        },
        'neon-purple': {
          DEFAULT: '#a855f7',
          glow: '#c084fc',
        },
      },
      animation: {
        'fade-in': 'fadeIn 0.3s ease-out',
        'slide-up': 'slideUp 0.4s ease-out',
        'pulse-slow': 'pulse 3s infinite',
        'breathe': 'gentle-breathe 3s ease-in-out infinite',
        'slide-in-right': 'slide-in-right 0.3s ease-out',
      },
      keyframes: {
        fadeIn: {
          '0%': { opacity: '0' },
          '100%': { opacity: '1' },
        },
        slideUp: {
          '0%': { transform: 'translateY(10px)', opacity: '0' },
          '100%': { transform: 'translateY(0)', opacity: '1' },
        },
        'gentle-breathe': {
          '0%, 100%': {
            filter: 'drop-shadow(0 0 3px var(--glow-color))',
            opacity: '0.9',
          },
          '50%': {
            filter: 'drop-shadow(0 0 8px var(--glow-color)) drop-shadow(0 0 12px var(--glow-color))',
            opacity: '1',
          },
        },
        'slide-in-right': {
          from: { transform: 'translateX(100%)', opacity: '0' },
          to: { transform: 'translateX(0)', opacity: '1' },
        },
      },
    },
  },
  plugins: [],
}
```

---

## useResizablePanel Hook

```typescript
import { useState, useCallback, useEffect, useRef } from 'react';

interface UseResizablePanelOptions {
  direction: 'horizontal' | 'vertical';
  initialSize: number;
  minSize: number;
  maxSize: number;
}

export function useResizablePanel({ direction, initialSize, minSize, maxSize }: UseResizablePanelOptions) {
  const [size, setSize] = useState(initialSize);
  const isDragging = useRef(false);
  const startPos = useRef(0);
  const startSize = useRef(0);

  const handleMouseDown = useCallback((e: React.MouseEvent) => {
    e.preventDefault();
    isDragging.current = true;
    startPos.current = direction === 'horizontal' ? e.clientX : e.clientY;
    startSize.current = size;
    document.body.style.cursor = direction === 'horizontal' ? 'col-resize' : 'row-resize';
    document.body.style.userSelect = 'none';
  }, [direction, size]);

  useEffect(() => {
    const onMouseMove = (e: MouseEvent) => {
      if (!isDragging.current) return;
      const delta = (direction === 'horizontal' ? e.clientX : e.clientY) - startPos.current;
      const newSize = Math.min(maxSize, Math.max(minSize, startSize.current + (direction === 'horizontal' ? -delta : -delta)));
      setSize(newSize);
    };

    const onMouseUp = () => {
      isDragging.current = false;
      document.body.style.cursor = '';
      document.body.style.userSelect = '';
    };

    document.addEventListener('mousemove', onMouseMove);
    document.addEventListener('mouseup', onMouseUp);
    return () => {
      document.removeEventListener('mousemove', onMouseMove);
      document.removeEventListener('mouseup', onMouseUp);
    };
  }, [direction, minSize, maxSize]);

  return { size, isDragging: isDragging.current, handleMouseDown };
}
```
