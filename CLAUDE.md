# Banner Blocker Extension — Development Guide

## Project Overview

Chrome Manifest V3 extension that blocks specific annoying banners on websites. Minimal permissions (storage only), zero data egress (CSP blocks all network from extension pages). Content scripts scoped to specific domains per banner.

## Architecture

Two-layer approach to hide banners:

1. **`blocker.css`** (injected at `document_start`) — Static CSS that hides banners before the page renders. Primary mechanism. Prevents visual flash.
2. **`content.js`** (runs at `document_idle`) — JavaScript for toggle state, MutationObserver for SPA navigation, and storage persistence.

The CSS layer does the heavy lifting. The JS layer exists for user preferences and dynamic behavior.

## File Map

| File | Purpose |
|------|---------|
| `blocker.css` | Static CSS rules injected before page render. Add new banner selectors here first. |
| `banners.js` | Banner registry (`BANNERS` array) + shared utilities (`hideBanner`, `showBanner`, `matchesBanner`). |
| `content.js` | Content script: loads storage, processes banners, observes DOM mutations via MutationObserver. |
| `popup.js` | Popup UI: renders toggle switches per banner, persists state to `chrome.storage.local`. |
| `popup.html` / `popup.css` | Popup markup and styles. Minimal — header, toggle list. |
| `manifest.json` | Extension manifest. Domain matches, content script injection order, CSP. |
| `Makefile` | `make test`, `make coverage`, `make check`, `make clean`, `make help` |
| `vitest.config.js` | Test config: jsdom environment, setup file. |
| `tests/setup.js` | Loads `banners.js` globals into test environment via function evaluation. |
| `tests/chrome-mock.js` | Chrome API stubs: `storage.local`, `storage.onChanged`, `runtime.lastError`. |

## Code Conventions

- **Plain JavaScript** — No transpilation, no modules, no TypeScript. Files are browser globals loaded via manifest script order.
- **`var` declarations** — Intentional choice for browser content script compatibility. `const` used only in `banners.js` for frozen registry objects.
- **IIFE wrapping** — `content.js` and `popup.js` wrap in `(function() { "use strict"; ... })()` to avoid global pollution.
- **`[Banner Blocker]` log tag** — All `console.log`/`console.warn` calls use this prefix for filtering.
- **`Object.freeze`** — `BANNERS` and `BANNER_IDS` are frozen to prevent accidental mutation.

## Adding a New Banner

Four steps:

### 1. Find the right selectors (see "Finding the Right Element" below)

### 2. Add CSS rules to `blocker.css`

Target both the banner element AND its layout slot container.

### 3. Add a registry entry to `banners.js`

```js
{
  id: "unique-id",
  name: "Display Name",
  description: "What the banner says",
  defaultEnabled: true,
  selector: '[data-testid="some-selector"]',
  textMatch: "Text to match"
}
```

### 4. Add the domain to `manifest.json` if new

Add to both `content_scripts` entries (CSS at `document_start` and JS at `document_idle`).

## Finding the Right Element to Block

This is the hard part. Banners on modern SPAs are wrapped in multiple layers of layout containers, and hiding the visible element often leaves blank space.

### What to ask the user for

1. **The URL** where the banner appears
2. **The banner's DOM** — Right-click the banner > Inspect > copy outerHTML of the banner AND 2-3 parent levels up
3. **The parent layout slot** — The container that reserves vertical space

### What to look for in the DOM

Work outward from the banner text:

1. **The banner element** — Usually has `data-testid`, `data-ssr-placeholder`, `data-vc`, or `role="banner"`. Gets `display: none`.
2. **The layout slot** — Parent that reserves space. Look for `data-layout-slot="true"`, `data-testid="page-layout.banner"`, or `data-vc="banner-container-component"`. Needs `display: none` + height clamping.
3. **Inline `<style>` tags** — Frameworks inject `<style>` elements setting CSS custom properties (e.g., `--bannerHeight: 64px`). Often siblings of the banner inside the layout slot. `display: none` on the parent does NOT stop these from executing.

### The three layers of hiding

Most banners need all three:

```css
/* Layer 1: Hide the banner element */
[data-ssr-placeholder="announcement-banner"] { display: none !important; }

/* Layer 2: Collapse the layout slot */
[data-testid="page-layout.banner"] {
  display: none !important;
  height: 0 !important;
  min-height: 0 !important;
  max-height: 0 !important;
  overflow: hidden !important;
  padding: 0 !important;
  margin: 0 !important;
}

/* Layer 3: Override CSS variables that push content down */
:root { --bannerHeight: 0px !important; }
```

### Why `display: none` alone fails

- `display: none` hides rendering but `<style>` children still execute
- CSS custom properties from those `<style>` tags still apply to `:root`
- Other elements use `var(--bannerHeight)` in `margin-top` or `calc()`
- The layout slot may have fixed height from a class

### Diagnosing blank space

If the banner disappears but space remains:

1. Check if the layout slot parent has its own height/min-height
2. Look for `<style>` tags inside the hidden container that set CSS variables
3. Search the page for uses of those variables
4. Check if other elements use `calc(var(--bannerHeight))` for positioning

### Same banner, different products

Atlassian uses different DOM structures across products:

| Product | Banner selector | Layout slot selector |
|---------|----------------|---------------------|
| Jira | `[data-ssr-placeholder="announcement-banner"]` | `[data-testid="page-layout.banner"]` |
| Confluence | `[data-vc="admin-announcement-banner"]` | `[data-vc="banner-container-component"]` |

Always check the actual DOM on each product.

## SPA Navigation

Atlassian products are SPAs. The content script runs once on initial page load:

- `blocker.css` rules persist across SPA navigations (CSS stays in the document)
- `MutationObserver` in `content.js` detects DOM changes and re-runs `processBanners`
- The CSS handles immediate visual hiding; JS handles attribute-based toggle state

## Testing

```
make test      # Run vitest
make coverage  # Run with coverage (requires @vitest/coverage-v8)
make check     # test + coverage
make clean     # Remove node_modules/ and coverage/
```

- Tests use Vitest + jsdom
- Source files loaded via function evaluation in `tests/setup.js` (not ES module imports) because they are plain browser scripts
- Chrome APIs stubbed via `tests/chrome-mock.js`
- MutationObserver tests require `vi.useFakeTimers()` + `vi.advanceTimersByTimeAsync()` for debounce handling

## Common Pitfalls

- **Don't rely on JS-only hiding** — There will always be a flash between render and script execution. Use `blocker.css` for visual hide.
- **Don't try `:root:has()` for CSS variable overrides** — Some frameworks re-inject `<style>` tags that win specificity. Direct variable overrides on `:root` with `!important` are more reliable.
- **Don't remove DOM elements** — Removing elements breaks React virtual DOM reconciliation and causes re-injection loops. Use `display: none`.
- **Don't forget height clamping** — `display: none` on a flex/grid child may not collapse parent reserved space. Add `height: 0 !important` as backup.
- **Check for `<style>` siblings** — Most common cause of persistent blank space.
- **Test on both Jira and Confluence** — Same banner, different DOM structures.

## CI

GitHub Actions workflows in `.github/workflows/`:
- `test.yml` — Runs `make test` on Node 20 and 22
- `coverage.yml` — Runs `make coverage` with 80% threshold, comments on PRs if below

## Dependencies

- `vitest` — Test runner
- `jsdom` — Browser environment for tests
- `@vitest/coverage-v8` — Coverage provider

No runtime dependencies. No build step.
