# Banner Blocker

Chrome extension that blocks specific annoying banners on websites.

Unlike broad-access ad blockers, this extension uses **minimal permissions** and has **zero data egress** capability. It only needs `storage` permission for toggle persistence, and content scripts are scoped to specific domains per banner.

## Supported Banners

| Banner | Description |
|--------|-------------|
| Atlassian Cloud Welcome | "Welcome to Atlassian Cloud! To report issues, raise a support ticket here." |

## Install

1. Go to `chrome://extensions`
2. Enable **Developer mode** (top right)
3. Click **Load unpacked** and select this directory
4. Navigate to any `*.atlassian.net` page

## Usage

Click the extension icon to toggle individual banners on/off. Changes apply immediately without page reload.

## Adding New Banners

Add an entry to the `BANNERS` array in `banners.js`:

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

For banners on new domains, also add the domain to the `matches` array in `manifest.json`.

## Permissions

- **Host match**: Content scripts scoped to specific domains (e.g., `*://*.atlassian.net/*`)
- **Storage**: `chrome.storage.local` (toggle persistence)
- **CSP**: `connect-src 'none'` blocks all network from extension pages
- No `host_permissions`, `activeTab`, `webRequest`, or network-capable APIs

## License

See [LICENSE](LICENSE).
