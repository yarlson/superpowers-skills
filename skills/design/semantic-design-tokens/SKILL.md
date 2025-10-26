---
name: Semantic Design Tokens
description: Three-layer CSS token architecture for scalable, maintainable UI design systems
when_to_use: Creating UI with CSS. When hardcoding colors, spacing, or shadows. When mixing primitive values with semantic names. When CSS variables lack clear hierarchy. Building design systems, landing pages, or any styled interface.
version: 1.0.0
---

# Semantic Design Tokens

## Overview

**Semantic design tokens organize styles into three hierarchical layers: base primitives, semantic meanings, and component-specific values.** This creates scalable, maintainable design systems where intent is explicit and changes cascade predictably.

Using CSS variables is NOT enough. The three-layer architecture is what makes tokens semantic.

## When to Use

**Use this for ALL UI styling work:**

- Building landing pages, web apps, or any styled interface
- Creating component libraries or design systems
- Setting up project styles from scratch

**Symptoms you're violating this:**

- Hardcoded hex colors or pixel values in component styles
- CSS variables without clear base → semantic → component hierarchy
- Mixing primitives (`#6366f1`) directly with semantic names
- "Simple" or "fast" rationalizations for flat variable systems
- No spacing/radius/shadow scales
- Difficulty making consistent design changes

## The Three-Layer System

### Layer 1: Base Tokens (Primitives)

Raw values - the physical palette. Neutral, not opinionated.

```css
:root {
  /* Colors */
  --color-white: #ffffff;
  --color-black: #000000;
  --color-gray-50: #f9fafb;
  --color-gray-100: #f3f4f6;
  --color-gray-200: #e5e7eb;
  --color-gray-600: #4b5563;
  --color-gray-900: #111827;
  --color-gray-950: #030712;

  --color-primary-50: #eff6ff;
  --color-primary-500: #3b82f6;
  --color-primary-900: #1e3a8a;

  /* Spacing */
  --space-1: 0.25rem;
  --space-2: 0.5rem;
  --space-3: 0.75rem;
  --space-4: 1rem;
  --space-6: 1.5rem;
  --space-8: 2rem;
  --space-10: 2.5rem;
  --space-12: 3rem;

  /* Radius */
  --radius-xs: 0.125rem;
  --radius-sm: 0.25rem;
  --radius-md: 0.375rem;
  --radius-lg: 0.5rem;
  --radius-xl: 0.75rem;
  --radius-2xl: 1rem;

  /* Shadows */
  --shadow-xs: 0 1px 2px rgba(0, 0, 0, 0.05);
  --shadow-sm: 0 1px 3px rgba(0, 0, 0, 0.1);
  --shadow-md: 0 4px 6px rgba(0, 0, 0, 0.1);
  --shadow-lg: 0 10px 15px rgba(0, 0, 0, 0.1);
  --shadow-xl: 0 20px 25px rgba(0, 0, 0, 0.1);
}
```

### Layer 2: Semantic Tokens (Purpose-Driven)

Define the MEANING of design decisions. These reference base tokens.

```css
:root {
  --surface-bg: var(--color-gray-50);
  --surface-elevated: var(--color-white);
  --text-primary: var(--color-gray-900);
  --text-secondary: var(--color-gray-600);
  --text-tertiary: var(--color-gray-500);
  --brand-accent: var(--color-primary-500);
  --brand-accent-hover: var(--color-primary-600);
  --border-subtle: var(--color-gray-200);
  --border-default: var(--color-gray-300);

  --radius-default: var(--radius-lg);
  --shadow-default: var(--shadow-md);
  --spacing-default: var(--space-4);
}

.dark {
  --surface-bg: var(--color-gray-950);
  --surface-elevated: var(--color-gray-900);
  --text-primary: var(--color-gray-50);
  --text-secondary: var(--color-gray-400);
  --brand-accent: var(--color-primary-400);
  --border-subtle: var(--color-gray-800);
}
```

### Layer 3: Component Tokens (Contextual)

Component-specific values derived from semantics.

```css
:root {
  --button-bg: var(--brand-accent);
  --button-bg-hover: var(--brand-accent-hover);
  --button-text: var(--color-white);
  --button-padding: var(--space-3) var(--space-6);
  --button-radius: var(--radius-default);
  --button-shadow: var(--shadow-sm);
  --button-shadow-hover: var(--shadow-md);

  --card-bg: var(--surface-elevated);
  --card-border: var(--border-subtle);
  --card-padding: var(--space-6);
  --card-radius: var(--radius-xl);
  --card-shadow: var(--shadow-default);

  --input-bg: var(--surface-elevated);
  --input-border: var(--border-default);
  --input-text: var(--text-primary);
  --input-placeholder: var(--text-tertiary);
}
```

## Quick Reference

| Layer         | Purpose            | Example                            | References               |
| ------------- | ------------------ | ---------------------------------- | ------------------------ |
| **Base**      | Raw primitives     | `--color-gray-500`, `--space-4`    | Nothing (literal values) |
| **Semantic**  | Design intent      | `--text-primary`, `--brand-accent` | Base tokens only         |
| **Component** | Contextual styling | `--button-bg`, `--card-shadow`     | Semantic tokens only     |

**Flow: Base → Semantic → Component**

Never skip layers. Never mix layers.

## Using Tokens in Components

```css
/* ❌ BAD: Hardcoded, no tokens */
.button {
  background: #3b82f6;
  padding: 12px 24px;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

/* ❌ BAD: Base tokens directly in components */
.button {
  background: var(--color-primary-500);
  padding: var(--space-3) var(--space-6);
  border-radius: var(--radius-lg);
}

/* ✅ GOOD: Component tokens only */
.button {
  background: var(--button-bg);
  padding: var(--button-padding);
  border-radius: var(--button-radius);
  box-shadow: var(--button-shadow);
  color: var(--button-text);
  transition: all 0.2s ease;
}

.button:hover {
  background: var(--button-bg-hover);
  box-shadow: var(--button-shadow-hover);
  transform: translateY(-1px);
}
```

## Common Mistakes

| Mistake                                   | Fix                                                   |
| ----------------------------------------- | ----------------------------------------------------- |
| CSS variables without three layers        | Refactor into base → semantic → component hierarchy   |
| Hardcoded values in component styles      | Create base tokens, then semantic, then component     |
| Mixing primitives with semantics in :root | Separate base palette from semantic meanings          |
| "This is simpler/faster" rationalization  | Structure pays off at scale - always use three layers |
| Using base tokens directly in components  | Always go through semantic → component layers         |
| One-dimensional shadow/spacing            | Create full scales (xs → xl)                          |

## Rationalization Table

| Excuse                                    | Reality                                                   |
| ----------------------------------------- | --------------------------------------------------------- |
| "CSS variables are semantic enough"       | Variables without three-layer hierarchy = unscalable mess |
| "Three layers is overkill"                | Layer separation enables consistent design changes        |
| "This is faster for small projects"       | Establishing tokens takes 5 minutes, saves hours later    |
| "I need flexibility to hardcode"          | Tokens ARE flexibility - change once, update everywhere   |
| "Design is too simple for this"           | Simplicity makes tokens easier, not unnecessary           |
| "Just using `--text-primary` is semantic" | Not if it contains `#111827` directly - needs base layer  |

## Red Flags

**STOP if you're doing any of these:**

- Hardcoding hex colors anywhere except base layer
- Hardcoding pixel values in component styles
- Justifying flat variable structure with "speed" or "simplicity"
- Skipping layer(s) because project is "too small"
- Mixing `#ffffff` with `var(--text-primary)` in same scope

**All of these mean: Restructure with proper three-layer tokens.**

## Why This Matters

**Scalability:** Add new components by referencing existing semantic tokens. No new color/spacing decisions.

**Consistency:** All blues come from one palette. All spacing from one scale. Impossible to drift.

**Theme switching:** Change semantic layer, all components update. Dark mode = redefine semantics, not every component.

**Maintenance:** Need softer shadows? Update `--shadow-md` once. Need different brand color? Update base, cascades through semantic to components.

The three-layer system makes intent explicit and changes predictable. It's the foundation of professional design systems.
