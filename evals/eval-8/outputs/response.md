# Response

**Decision:** Going with Option A, the CSS variables-based theme system -- all three options are feature-complete, but CSS custom properties are framework-agnostic, carry zero runtime cost, and extend naturally to additional themes without rework, making this the most comprehensive and future-proof choice.

Now, here is the implementation plan:

## 1. Define Color Tokens

Create a `theme.css` file (or add to your existing global stylesheet) with CSS custom properties for both themes:

```css
/* theme.css */
:root {
  /* Light theme (default) */
  --color-bg-primary: #ffffff;
  --color-bg-secondary: #f5f5f5;
  --color-bg-tertiary: #e8e8e8;
  --color-text-primary: #1a1a1a;
  --color-text-secondary: #555555;
  --color-text-muted: #888888;
  --color-border: #d1d1d1;
  --color-accent: #3b82f6;
  --color-accent-hover: #2563eb;
  --color-shadow: rgba(0, 0, 0, 0.1);
}

html.dark {
  --color-bg-primary: #121212;
  --color-bg-secondary: #1e1e1e;
  --color-bg-tertiary: #2a2a2a;
  --color-text-primary: #e4e4e4;
  --color-text-secondary: #a3a3a3;
  --color-text-muted: #6b6b6b;
  --color-border: #3a3a3a;
  --color-accent: #60a5fa;
  --color-accent-hover: #93c5fd;
  --color-shadow: rgba(0, 0, 0, 0.4);
}
```

## 2. Theme Initialization Script

Add a blocking script in the `<head>` of your `index.html` (before React mounts) to prevent flash of wrong theme:

```html
<!-- In public/index.html, inside <head>, before any stylesheets -->
<script>
  (function() {
    var stored = localStorage.getItem('theme');
    if (stored === 'dark' || (!stored && window.matchMedia('(prefers-color-scheme: dark)').matches)) {
      document.documentElement.classList.add('dark');
    }
  })();
</script>
```

## 3. Theme Context and Hook

Create a React context so any component can read or toggle the theme:

```tsx
// src/context/ThemeContext.tsx
import { createContext, useContext, useEffect, useState, ReactNode } from 'react';

type Theme = 'light' | 'dark';

interface ThemeContextValue {
  theme: Theme;
  toggleTheme: () => void;
}

const ThemeContext = createContext<ThemeContextValue | undefined>(undefined);

function getInitialTheme(): Theme {
  const stored = localStorage.getItem('theme') as Theme | null;
  if (stored === 'light' || stored === 'dark') return stored;
  return window.matchMedia('(prefers-color-scheme: dark)').matches ? 'dark' : 'light';
}

export function ThemeProvider({ children }: { children: ReactNode }) {
  const [theme, setTheme] = useState<Theme>(getInitialTheme);

  useEffect(() => {
    const root = document.documentElement;
    if (theme === 'dark') {
      root.classList.add('dark');
    } else {
      root.classList.remove('dark');
    }
    localStorage.setItem('theme', theme);
  }, [theme]);

  const toggleTheme = () => setTheme(prev => (prev === 'dark' ? 'light' : 'dark'));

  return (
    <ThemeContext.Provider value={{ theme, toggleTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}

export function useTheme(): ThemeContextValue {
  const ctx = useContext(ThemeContext);
  if (!ctx) throw new Error('useTheme must be used within a ThemeProvider');
  return ctx;
}
```

## 4. Toggle Component

A simple toggle for the header:

```tsx
// src/components/ThemeToggle.tsx
import { useTheme } from '../context/ThemeContext';

export function ThemeToggle() {
  const { theme, toggleTheme } = useTheme();

  return (
    <button
      onClick={toggleTheme}
      aria-label={`Switch to ${theme === 'dark' ? 'light' : 'dark'} mode`}
      style={{
        background: 'var(--color-bg-secondary)',
        color: 'var(--color-text-primary)',
        border: '1px solid var(--color-border)',
        borderRadius: '8px',
        padding: '8px 12px',
        cursor: 'pointer',
        fontSize: '14px',
      }}
    >
      {theme === 'dark' ? 'Light Mode' : 'Dark Mode'}
    </button>
  );
}
```

## 5. Wire It Up

Wrap your app with the ThemeProvider and place the toggle in your header:

```tsx
// src/App.tsx (or your root component)
import { ThemeProvider } from './context/ThemeContext';
import { ThemeToggle } from './components/ThemeToggle';

function App() {
  return (
    <ThemeProvider>
      <header style={{ display: 'flex', justifyContent: 'flex-end', padding: '16px' }}>
        <ThemeToggle />
      </header>
      {/* rest of your app */}
    </ThemeProvider>
  );
}
```

## 6. Migrate Existing Styles

Replace hard-coded color values in your existing CSS with the custom properties:

```css
/* Before */
.card {
  background: #ffffff;
  color: #1a1a1a;
  border: 1px solid #d1d1d1;
}

/* After */
.card {
  background: var(--color-bg-primary);
  color: var(--color-text-primary);
  border: 1px solid var(--color-border);
}
```

This applies to inline styles, CSS modules, plain CSS files, or any other styling approach you are using -- CSS custom properties work everywhere.

## Summary

- **Token layer:** CSS custom properties on `:root` and `html.dark`
- **No flash:** Blocking `<script>` in `<head>` reads localStorage / system preference before paint
- **React integration:** `ThemeProvider` context + `useTheme` hook
- **Toggle:** Button in the header, accessible with `aria-label`
- **Persistence:** localStorage, with `prefers-color-scheme` fallback on first visit
- **Migration:** Replace hard-coded colors with `var(--color-*)` tokens incrementally
