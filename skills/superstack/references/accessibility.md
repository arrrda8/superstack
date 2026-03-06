# Accessibility Reference — Superstack (WCAG 2.1 AA)

Accessibility is a legal requirement in the EU since June 2025 (European Accessibility Act). Fines up to 100,000 EUR or 4% of annual revenue. It also directly impacts SEO, usability, and conversion rates. Read this during Phase 5 or whenever building UI components.

> **EAA Compliance (since June 28, 2025):** E-commerce, banking, communication services, and transport apps MUST meet WCAG 2.1 AA. A public Accessibility Statement is required. Use axe-core for automated testing and manual screen reader testing (VoiceOver on Mac, NVDA on Windows).

## Table of Contents
1. Semantic HTML
2. Keyboard Navigation
3. ARIA Attributes
4. Color & Contrast
5. Motion & Animation
6. Forms
7. Touch & Mobile
8. Verification Checklist

---

## 1. Semantic HTML

Use the right element for the job — this is the foundation of accessibility:

- `<nav>` for navigation, `<main>` for primary content, `<article>` for self-contained content
- `<button>` for actions, `<a>` for navigation — never `<div onClick>`
- `<header>`, `<footer>`, `<aside>` as landmarks
- Proper heading hierarchy: h1 → h2 → h3, no skipping levels
- Single `<h1>` per page that reflects the page content

---

## 2. Keyboard Navigation

Every interactive element must be reachable and operable with keyboard alone:

- **Tab order**: Follows visual flow. Use `tabindex="0"` only when necessary, avoid positive tabindex values.
- **Focus indicators**: Style them intentionally — don't rely on browser defaults. Use `outline` or `box-shadow` with sufficient contrast.
- **Skip-to-content link**: First focusable element on every page, visually hidden until focused.

```css
.skip-link {
  position: absolute;
  top: -100%;
  left: 0;
  z-index: 100;
  padding: 1rem;
  background: var(--primary);
  color: white;
}
.skip-link:focus {
  top: 0;
}
```

---

## 3. Focus Management

- **Modals/Dialogs**: Trap focus inside when open. Return focus to the trigger element on close.
- **Dynamic content**: When content loads or changes, manage focus to guide the user (e.g., focus the new content after a route change).
- **Dropdown menus**: Arrow keys for navigation within, Escape to close, focus returns to trigger.

---

## 4. ARIA Attributes

Use ARIA only when semantic HTML isn't sufficient:

- `aria-label` on icon-only buttons (e.g., close button, hamburger menu)
- `aria-expanded` on toggles, dropdowns, accordions
- `aria-live="polite"` for dynamic content updates (toast notifications, form errors)
- `aria-describedby` to link error messages to form fields
- `role="dialog"` with `aria-modal="true"` for modals

---

## 5. Color & Contrast

- **Text contrast**: Minimum 4.5:1 ratio (WCAG AA). Large text (18px+ bold or 24px+ regular): 3:1.
- **UI elements**: 3:1 against adjacent colors (buttons, form borders, icons).
- **Don't rely on color alone**: Use icons, patterns, or text labels alongside color to convey meaning (e.g., error states need more than just red).
- Test with a contrast checker tool during design phase.

---

## 6. Motion & Animation

Users with vestibular disorders can experience nausea or disorientation from animations:

```tsx
// Framer Motion
import { useReducedMotion } from 'framer-motion';

function AnimatedComponent() {
  const prefersReducedMotion = useReducedMotion();
  return (
    <motion.div
      animate={{ y: prefersReducedMotion ? 0 : 20, opacity: 1 }}
      transition={{ duration: prefersReducedMotion ? 0 : 0.6 }}
    />
  );
}
```

```css
/* CSS fallback */
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
  }
}
```

Also provide a motion toggle in the UI settings for users who want to disable animations explicitly.

---

## 7. Forms

- Every `<input>` has a visible `<label>` connected via `htmlFor`/`id`
- Error messages linked to fields via `aria-describedby`
- Use `<fieldset>` and `<legend>` for related groups (e.g., radio buttons)
- `autocomplete` attributes on common fields (name, email, address, credit card)
- Clear error states: red border + icon + text message (not just color)

---

## 8. Touch & Mobile

- **Touch targets**: Minimum 44x44px for all interactive elements
- **Spacing**: At least 8px between touch targets to prevent accidental taps
- **Gestures**: Any gesture-based interaction must have a button alternative

---

## 9. Verification Checklist

Run through this before delivery:

- [ ] All interactive elements keyboard-accessible with visible focus
- [ ] Skip-to-content link present and working
- [ ] Color contrast meets WCAG AA (4.5:1 / 3:1)
- [ ] All images have meaningful alt text (or `alt=""` for decorative)
- [ ] Forms fully labeled with error association
- [ ] Animations respect `prefers-reduced-motion`
- [ ] Modal focus trapping works correctly
- [ ] Screen reader announces all content logically
- [ ] Touch targets >= 44x44px on mobile
- [ ] Heading hierarchy is correct (no skipped levels)
- [ ] Page titles are descriptive and unique
