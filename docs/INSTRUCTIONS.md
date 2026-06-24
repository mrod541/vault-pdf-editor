# Vault

A PDF editor that runs **entirely in your browser**, with a **browser-enforced**
guarantee that your file never leaves your device. The value isn't features —
it's *provable privacy*.

Everything you need to use, deploy, verify, and rebuild Vault is below. Pick the
section you need.

- [Using the editor](#using-the-editor)
- [How the privacy guarantee works](#how-the-privacy-guarantee-works)
- [Build status](#build-status)
- [Deploy to Cloudflare](#deploy-to-cloudflare)
- [Rebuild from source](#rebuild-from-source)
- [Updating and re-verifying](#updating-and-re-verifying)
- [Troubleshooting](#troubleshooting)

---

## Using the editor

1. Open one of the editor files in a modern browser (Chrome, Edge, Firefox,
   Safari). You can open a local file directly, or visit your deployed site.
2. Drop a PDF onto the page, or click **Choose a PDF**.
3. Use the tools: Select, Add text, Draw, Highlight, Redact (and OCR in the OCR
   builds). Page tools (Rotate / Delete) are above each page.
4. Click **Save** to download an edited copy. The original is untouched.
5. **Discard** clears the document from memory.

> **What the editor does and doesn't do.** It adds content on top — text,
> drawings, highlights, images — and performs *true* redaction (the covered
> content is removed, not just hidden), plus page reorder/rotate/delete. It does
> **not** edit existing body text in place or move existing objects; that needs
> a desktop/server tool. This is a deliberate scope for a client-side tool.

---

## How the privacy guarantee works

The claim is **"your PDF never leaves your device,"** and it's enforced two ways:

**1. A Content-Security-Policy that blocks all network access.**
Every editor file ships this in its `<head>`, and the Cloudflare Worker also
sends it as a real HTTP header:

```
Content-Security-Policy: ... connect-src blob:; ...
```

`connect-src blob:` tells the browser to block every network destination — no
`fetch`, `XMLHttpRequest`, WebSocket, or `sendBeacon` can reach any
`http(s)://` or `ws://` server. If the page tried to upload your file, the
browser blocks the connection. The protection is the browser's, not a promise in
the code. The one permitted scheme, `blob:`, lets the page read its **own**
in-memory inlined data (the OCR model, worker, and WASM core); a `blob:` URL
points at bytes already in the tab, not a network address, so nothing leaves your
device.

**2. Everything is inlined — nothing is fetched at runtime.**
The libraries (pdf.js, pdf-lib, and for OCR builds tesseract.js + the language
model) are embedded in the single HTML file. After load, a compliant build makes
zero requests. You can run it fully offline.

**Verify it yourself (any one of these):**
- Click **Verify privacy** in the app. Watch the live request counter stay at 0,
  and use the **"try to phone home"** button to see the browser block a test
  request.
- Open DevTools (F12) -> **Network**, then edit a PDF. The list stays empty.
- **Disconnect from the internet.** A compliant build keeps working.

**The strongest guarantee — pin a verified copy.** CSP protects you during a
session; to protect yourself across time, verify a file's hash once and keep
that copy:

```powershell
# Windows
Get-FileHash public\pdf-editor-lean.html -Algorithm SHA256
```
```bash
# macOS / Linux
shasum -a 256 public/pdf-editor-lean.html
```

Compare to `public/SHA256SUMS.txt`. A match means the file is byte-for-byte the
published one. (See also [`AUDIT_NOTE.md`](AUDIT_NOTE.md) for every external
string in the files, explained — and for the current per-build audit status.)

---

## Build status

| Build | Size | OCR | Status | Use when |
|-------|------|-----|--------|----------|
| `pdf-editor-lean.html` | ~1.9 MB | — | ✅ Verified self-contained | Default. Editing, markup, redaction, page ops. |
| `pdf-editor-ocr-full.html` | ~26 MB | full | ✅ **Self-contained.** Model + worker + core all inlined; runs OCR offline. Verify with an offline run. See [`AUDIT_NOTE.md`](AUDIT_NOTE.md). | OCR on messy scans, best accuracy. |
| `pdf-editor-ocr-fast.html` | ~5.8 MB | fast | ⚠️ **OCR not working yet.** No OCR assets inlined; see [`AUDIT_NOTE.md`](AUDIT_NOTE.md). | Don't rely on its OCR until rebuilt. |

The lean build is the verified, recommended default and works fully offline.

`pdf-editor-ocr-full.html` now inlines all three OCR assets — language model,
worker, and the tesseract.js **WebAssembly core** (5.1.0 simd-lstm) — and wires
each to an in-worker blob, so OCR runs fully offline. The quickest proof is to
open it, disconnect from the internet, and OCR a scan.

`pdf-editor-ocr-fast.html` still inlines none of its OCR assets; they default to
a CDN and are blocked by the CSP, so its OCR doesn't work yet. It fails *closed*
(no leak) and stays flagged until rebuilt the same way `ocr-full` was.

Privacy holds for all three. Full per-build detail — including exactly how the
core is inlined and why the leftover CDN strings are dead branches — is in
[`AUDIT_NOTE.md`](AUDIT_NOTE.md).

---

## Deploy to Cloudflare

This uses **Cloudflare Workers Static Assets** — the current recommended way to
host sites on Cloudflare (Pages is legacy for new projects). The Worker serves
the files and stamps the security headers. Free tier covers this easily.

### Prerequisites

- A free [Cloudflare account](https://dash.cloudflare.com/sign-up).
- [Node.js](https://nodejs.org) 18+ installed (`node --version` to check).
- This repo cloned locally.

### Steps

```bash
# 1. Install dependencies (just Wrangler, Cloudflare's CLI)
npm install

# 2. Log in to Cloudflare (opens a browser to authorize)
npx wrangler login

# 3. Preview locally first — runs the Worker + assets on your machine
npm run dev
#    Open the printed http://localhost:8787 and confirm it works.

# 4. Deploy to Cloudflare's edge
npm run deploy
```

Wrangler prints your live URL, e.g.
`https://vault-pdf-editor.<your-subdomain>.workers.dev`.

### What the config does

- **`wrangler.jsonc`** points `assets.directory` at `./public` (the editor
  files) and sets `main` to the Worker at `src/index.js`.
- **`src/index.js`** serves each requested file and adds the security headers —
  including the header-level `connect-src blob:` (kept byte-identical to each
  editor's `<meta>` CSP) — to every response. Static
  assets are served from Cloudflare's edge; the Worker only runs to add headers
  and never sees your PDF (it never leaves your browser).

### Optional: a custom domain

In the Cloudflare dashboard -> your Worker -> **Settings -> Domains & Routes**,
add a custom domain (e.g. `pdf.yourdomain.com`) if the domain is on Cloudflare.

### Optional: auto-deploy on git push

The included `.github/workflows/deploy.yml` deploys on every push to `main`. It
also runs `sha256sum -c` against `public/SHA256SUMS.txt` and fails the deploy if
an editor file was changed without refreshing the hashes. To enable it, add two
GitHub repository secrets (Settings -> Secrets and variables -> Actions):
- `CLOUDFLARE_API_TOKEN` — create one at Cloudflare -> My Profile -> API Tokens
  -> *Edit Cloudflare Workers* template.
- `CLOUDFLARE_ACCOUNT_ID` — found on your Workers dashboard.

---

## Rebuild from source

The editor HTML files are **generated**, so anyone can reproduce them and
confirm the hashes. The build inputs are all local — nothing is downloaded at
build time.

```bash
# from the build source (see the /build folder)
python3 build.py all        # or: lean / ocr-fast / ocr-full
```

This reads `template.html`, `app.css`, `app.js`, and the inlined libraries in
`assets/`, and writes the self-contained files to `dist/`. Copy them into
`public/` and regenerate hashes:

```bash
cp dist/pdf-editor-*.html public/
cd public && sha256sum pdf-editor-*.html > SHA256SUMS.txt
```

Because the build is deterministic, your output hashes should match the
published ones (barring intentional edits).

> **Note:** the `build/` source (template, css, js, and inlined library assets)
> is not yet in this repo — only the generated editor files are. Adding the
> build source is required to make the OCR builds reproducible and to fix the
> flagged `ocr-fast` build by inlining its OCR assets.

---

## Updating and re-verifying

When you change the editor:

1. Rebuild (`python3 build.py all`).
2. Refresh `public/SHA256SUMS.txt`.
3. Commit and deploy (`npm run deploy`, or push if auto-deploy is on).
4. Note the new hashes anywhere you've published them, so users can re-verify.

---

## Troubleshooting

**OCR doesn't start.**
`ocr-full` should OCR offline; if it stalls, open the browser console. `ocr-fast`
is the one still-flagged build — it inlines none of its OCR assets, so the CSP
correctly blocks its CDN fetch and OCR can't load there (see
[`AUDIT_NOTE.md`](AUDIT_NOTE.md)). The lean build has no OCR machinery and always
works — use it if you don't need OCR.

**The page looks blank or a tool does nothing.**
Open DevTools -> Console for the error and check you're on a current browser.
The editors target evergreen Chrome/Edge/Firefox/Safari.

**`npm run deploy` fails with an auth error.**
Re-run `npx wrangler login`, or for CI set `CLOUDFLARE_API_TOKEN`.

**Wrangler says my version is too old.**
`npm install -D wrangler@latest`, then retry. Static-assets features need a
recent Wrangler (4.x).
