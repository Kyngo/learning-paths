---
title: "Accessibility in Depth"
weight: 11
---

# Accessibility in Depth

Web accessibility means building interfaces that work for everyone — including people who use screen readers, keyboard-only navigation, magnification, voice control, or switch devices. This section covers ARIA, testing tools, and the patterns that make complex UI components accessible.

---

## WCAG 2.2 Levels

| Level | Standard | Requirement |
|-------|---------|-------------|
| **A** | Minimum | Basic accessibility (alt text, keyboard navigation, no seizure triggers) |
| **AA** | Industry standard | Colour contrast ≥ 4.5:1, focus visible, consistent navigation, error identification |
| **AAA** | Enhanced | Contrast ≥ 7:1, sign language, extended audio descriptions (rarely required) |

Most legal requirements and organisational standards target **Level AA**.

---

## ARIA (Accessible Rich Internet Applications)

ARIA attributes add semantic information to elements that HTML alone cannot express.

### The First Rule of ARIA

> No ARIA is better than bad ARIA.

Use semantic HTML first. ARIA is a last resort for custom components that have no native HTML equivalent.

```html
<!-- GOOD — no ARIA needed -->
<button>Submit</button>
<nav>...</nav>
<input type="checkbox">

<!-- BAD — ARIA reimplements what HTML already provides -->
<div role="button" tabindex="0" aria-pressed="false">Submit</div>
```

### Key ARIA Attributes

| Attribute | Purpose | Example |
|-----------|---------|---------|
| `role` | Defines what the element IS | `role="dialog"`, `role="tablist"` |
| `aria-label` | Accessible name (when no visible text) | `<button aria-label="Close">×</button>` |
| `aria-labelledby` | References visible text as label | `aria-labelledby="section-heading"` |
| `aria-describedby` | References additional description | `aria-describedby="password-hint"` |
| `aria-hidden` | Hides from accessibility tree | `<div aria-hidden="true">decorative icon</div>` |
| `aria-live` | Announces dynamic content changes | `aria-live="polite"` for status updates |
| `aria-expanded` | Collapsible state | `aria-expanded="false"` on dropdown trigger |
| `aria-selected` | Selection state | `aria-selected="true"` on active tab |
| `aria-invalid` | Validation state | `aria-invalid="true"` on errored input |
| `aria-required` | Required field | `aria-required="true"` |

### Live Regions

Announce dynamic content to screen readers without focus change:

```html
<!-- Polite: announces after current speech finishes -->
<div aria-live="polite" aria-atomic="true">
    3 items in cart
</div>

<!-- Assertive: interrupts current speech (use sparingly) -->
<div role="alert">
    Payment failed. Please try again.
</div>

<!-- Status: polite equivalent for status messages -->
<div role="status">
    Search results: 42 items found
</div>
```

---

## Keyboard Navigation Patterns

### Focus Management

| Key | Expected Behaviour |
|-----|--------------------|
| Tab | Move to next focusable element |
| Shift+Tab | Move to previous focusable element |
| Enter/Space | Activate focused element |
| Escape | Close modal/dropdown, cancel |
| Arrow keys | Navigate within composite widgets (tabs, menus, grids) |

### Focus Trapping (Modals)

When a modal is open, Tab should cycle through elements inside it — not escape to the page behind:

```javascript
function trapFocus(modal) {
    const focusable = modal.querySelectorAll(
        'a[href], button:not([disabled]), input, textarea, select, [tabindex]:not([tabindex="-1"])'
    );
    const first = focusable[0];
    const last = focusable[focusable.length - 1];

    modal.addEventListener('keydown', (e) => {
        if (e.key !== 'Tab') return;
        if (e.shiftKey && document.activeElement === first) {
            e.preventDefault();
            last.focus();
        } else if (!e.shiftKey && document.activeElement === last) {
            e.preventDefault();
            first.focus();
        }
    });

    first.focus();
}
```

### Skip Links

Allow keyboard users to skip repetitive navigation:

```html
<body>
    <a href="#main-content" class="skip-link">Skip to main content</a>
    <nav><!-- long navigation --></nav>
    <main id="main-content"><!-- page content --></main>
</body>

<style>
.skip-link {
    position: absolute;
    top: -40px;
    left: 0;
    z-index: 100;
}
.skip-link:focus {
    top: 0;  /* visible when focused */
}
</style>
```

---

## Common ARIA Patterns

### Tabs

```html
<div role="tablist" aria-label="Settings">
    <button role="tab" id="tab-1" aria-selected="true" aria-controls="panel-1">General</button>
    <button role="tab" id="tab-2" aria-selected="false" aria-controls="panel-2" tabindex="-1">Security</button>
</div>
<div role="tabpanel" id="panel-1" aria-labelledby="tab-1">General content</div>
<div role="tabpanel" id="panel-2" aria-labelledby="tab-2" hidden>Security content</div>
```

Arrow keys navigate between tabs. Tab key moves focus into the panel content.

### Disclosure (Accordion)

```html
<button aria-expanded="false" aria-controls="details-1">
    Show details
</button>
<div id="details-1" hidden>
    Expanded content here.
</div>
```

### Combobox (Autocomplete)

```html
<label for="city">City</label>
<input id="city" role="combobox" aria-expanded="false"
       aria-autocomplete="list" aria-controls="city-listbox">
<ul id="city-listbox" role="listbox" hidden>
    <li role="option" id="opt-1">Barcelona</li>
    <li role="option" id="opt-2">Berlin</li>
</ul>
```

---

## Testing Accessibility

### Automated Tools

| Tool | Type | What It Catches |
|------|------|----------------|
| axe-core | Library / browser extension | ~50% of WCAG violations |
| Lighthouse | Chrome DevTools | Accessibility score + specific issues |
| pa11y | CLI / CI | Automated WCAG checking |
| eslint-plugin-jsx-a11y | ESLint plugin | React accessibility issues at build time |
| @axe-core/playwright | Playwright integration | Automated accessibility in E2E tests |

### Playwright + axe Example

```javascript
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

test('home page passes accessibility checks', async ({ page }) => {
    await page.goto('/');
    const results = await new AxeBuilder({ page }).analyze();
    expect(results.violations).toEqual([]);
});

test('form page passes WCAG AA', async ({ page }) => {
    await page.goto('/contact');
    const results = await new AxeBuilder({ page })
        .withTags(['wcag2a', 'wcag2aa'])
        .analyze();
    expect(results.violations).toEqual([]);
});
```

### Manual Testing Checklist

- [ ] Navigate the entire page using only the keyboard (Tab, Enter, Escape, arrows)
- [ ] Test with a screen reader (VoiceOver on macOS, NVDA on Windows)
- [ ] Verify all interactive elements have visible focus indicators
- [ ] Check colour contrast with browser DevTools or WebAIM contrast checker
- [ ] Zoom to 200% — is all content still usable?
- [ ] Test with browser extensions disabled (some users can't run JS)
- [ ] Verify form errors are announced and associated with inputs

---

## Key Takeaways

- Use semantic HTML first. ARIA is a repair tool for custom components, not a first choice.
- Live regions (`aria-live`, `role="alert"`) announce dynamic content changes to screen readers.
- Keyboard navigation must work for every interactive element. Modals must trap focus.
- Automated tools catch ~50% of issues. Manual testing (keyboard + screen reader) catches the rest.
- Target WCAG 2.2 Level AA. Test with axe-core in CI and VoiceOver/NVDA manually.
- Accessibility is not a feature — it is a quality attribute. Build it in from the start, not as a retrofit.
