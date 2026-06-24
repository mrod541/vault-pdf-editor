# Vault — provable, browser-only PDF editing

A PDF editor that runs **entirely in your browser** and makes a
**browser-enforced** promise: your file never leaves your device. The value
isn't features — it's *provable privacy*. You verify the claim; you don't trust it.

**Live:** https://vault-pdf-editor.mozilla614.workers.dev

**New here? Read [`START_HERE.md`](START_HERE.md).**

## How the promise is kept

1. **`connect-src blob:`** in every editor file (as a `<meta>` tag) and stamped
   again as an HTTP header by the Cloudflare Worker. The browser blocks every
   network destination — no `fetch` / `XHR` / WebSocket / `sendBeacon` can reach
   any `http(s)://` or `ws://` server, so the page *cannot* upload your file. The
   one permitted scheme is `blob:`, which lets the page read its **own** in-memory
   inlined data (a `blob:` URL points at bytes already in the tab, not a network
   address). Nothing leaves the device.
2. **Everything inlined** — libraries (and, for a compliant OCR build, the OCR
   model/worker/core) are embedded in the single HTML file, so a finished build
   makes zero network requests and runs fully offline.
3. **Verifiable** — an in-app "Verify privacy" panel, an offline test, and
   published SHA-256 hashes (`public/SHA256SUMS.txt`).

## Build status (be aware)

| Build | Editing | OCR | Notes |
|-------|---------|-----|-------|
| `pdf-editor-lean.html` | ✅ | — | Fully self-contained. Recommended default. |
| `pdf-editor-ocr-full.html` | ✅ | ✅ offline | Model + worker + core all inlined. Verify with an offline OCR run. |
| `pdf-editor-ocr-fast.html` | ✅ | ❌ not yet | No OCR assets inlined. See audit. |

`ocr-full` runs OCR fully offline (model, worker, and WASM core all inlined) and
is deployed live; the quickest proof is to open it, disconnect from the internet,
and OCR a scan — verified working on the deployed site.
`ocr-fast` still inlines none of its OCR assets and so remains flagged — its OCR
fails *closed* (blocked by CSP, no leak). Details: [`docs/AUDIT_NOTE.md`](docs/AUDIT_NOTE.md).

## Layout

- `public/` — the deployed editors + `SHA256SUMS.txt` + landing `index.html`
- `src/index.js` — Cloudflare Worker; serves files, stamps security headers
- `wrangler.jsonc`, `package.json` — deploy config
- `docs/` — instructions, GitHub setup, project invariants, audit
- `build/` — (placeholder) the generator source; not yet populated
- `.github/workflows/deploy.yml` — optional auto-deploy with a hash-check gate

## Quick start

```bash
npm install
npx wrangler login
npm run dev      # preview at http://localhost:8787
npm run deploy   # ship to Cloudflare's edge
```

See [`docs/INSTRUCTIONS.md`](docs/INSTRUCTIONS.md) for the full walkthrough.

## License / contributing

Before changing anything, read [`docs/PROJECT_PROMPT.md`](docs/PROJECT_PROMPT.md)
— the privacy invariants are non-negotiable.
