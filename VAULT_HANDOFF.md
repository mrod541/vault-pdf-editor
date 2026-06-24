# Vault — Project Handoff

A snapshot of where **Vault** stands as of June 24, 2026. Drop this into your
notes (or the start of a new chat) to pick the project back up without
reconstructing context.

---

## What Vault is

A PDF editor that runs **entirely in the browser** with a **browser-enforced**
privacy guarantee: the user's file never leaves the device, and the user can
*prove* it (not just trust it). The value is provable privacy, not features.

Editing scope: additive markup (text, draw, highlight, images), **true
redaction** (removes underlying content), page reorder/rotate/delete, and OCR
(in the OCR build, fully offline). It deliberately does **not** edit existing
body text in place or move existing objects — that needs a desktop/native
engine and is out of scope for the client-side tool.

---

## Status: DONE and LIVE ✅

- **Live URL:** https://vault-pdf-editor.mozilla614.workers.dev
- **Repo:** https://github.com/mrod541/vault-pdf-editor (branch `main`)
- **Hosting:** Cloudflare Workers Static Assets (free tier). A tiny Worker
  (`src/index.js`) serves the files and stamps security headers; it never sees
  the PDF.
- Code, the deployed edge, and the docs are all in sync and accurate.

### The two builds

| File | Size | OCR | Status |
|------|------|-----|--------|
| `pdf-editor-lean.html` | ~1.9 MB | — | ✅ Fully self-contained. Recommended default. |
| `pdf-editor-ocr-full.html` | ~24.5 MB | ✅ offline | ✅ Model + worker + WASM core all inlined. OCR works offline. |

(`ocr-fast` — a never-completed build that referenced its OCR assets from a CDN
instead of inlining them — was removed. It never worked offline; the CSP blocked
its fetches, so OCR simply failed closed. Removing it eliminated a false "model
inlined" claim in its own header comment. See the backlog note below.)

### Published hashes (in `public/SHA256SUMS.txt`)

```
75d4fa31...66b932  index.html
2bf818f3...d594c    pdf-editor-lean.html
3fc1aa41...26af23   pdf-editor-ocr-full.html
```
(ocr-full is exactly 24,483,201 bytes — under Cloudflare's 25 MiB asset cap.)

### Recent commits (newest first)

- `b0f5694` — docs: correct CSP to `connect-src blob:` everywhere; AUDIT_NOTE
  describes the shipped OCR inlining; add live URL; note `build.py` unpopulated.
- `e5e4150` — Worker: align header CSP with editor meta CSP (`wasm-unsafe-eval`,
  `default-src 'none'`).
- `c6e3a8d` — OCR: working offline recognition in full build + accuracy tuning;
  correct privacy wording.

---

## The privacy mechanism (so you don't re-derive it)

Two layers enforce "nothing leaves the device":

1. **CSP `connect-src blob:`** — present as a `<meta>` tag in every editor *and*
   stamped as a real HTTP header by the Worker (the two are byte-identical).
   It blocks every network destination — no `fetch`/`XHR`/`WebSocket`/`beacon`
   can reach any `http(s)://` or `ws://` server. The **only** permitted scheme
   is `blob:`, which lets the page read its *own* in-memory inlined data; a
   `blob:` URL points at bytes already in the tab, not a network address.
   - **Why `blob:` and not `'none'`:** the inlined OCR model/worker/core are
     read back via same-tab blob URLs. `'none'` blocked those self-reads and
     broke OCR. `blob:` permits self-reads while still blocking all remote
     origins, so the guarantee holds. (This was a deliberate, documented call.)
2. **Everything inlined** — libraries (and, in ocr-full, the OCR model + worker
   + WASM core) are embedded in the single HTML file. After load, zero requests.
   Runs fully offline.

Verify-privacy panel + offline test + published SHA-256 hashes = the user checks
the claim themselves. The panel's "try to phone home" button fetches
`https://example.com` on purpose to demonstrate the browser blocking it.

`script-src` includes `'wasm-unsafe-eval'` (permits WASM compilation for OCR,
**not** general `eval()`).

---

## The hard-won OCR fix (the part that took all day)

ocr-full does OCR fully offline. Getting there meant rewiring tesseract.js
(v5.0.4, core **5.1.0** — note: there is no 5.0.4 core) so all three assets load
from in-memory bytes instead of a CDN, under strict CSP, inside a Web Worker:

- **WASM core:** the Emscripten loader's wasm-fetch gate is patched to seed from
  an inlined global `self.__VAULT_CORE_WASM__`; the patched loader (core bytes
  embedded) is injected at the top of the worker source; `getCore`'s
  `importScripts` branch early-returns when `TesseractCore` already exists.
- **Worker script:** inlined, turned into a same-realm blob URL, passed as
  `workerPath`.
- **Language model:** the killer bug. A main-thread blob URL is invalid inside
  the worker's realm, so the worker fell back to a (CSP-blocked, then
  realm-broken) fetch. **Fix:** intercept the OCR worker's `postMessage` — when
  tesseract sends `loadLanguage` with `langs:"eng"`, swap in
  `[{code:'eng', data:<bytes>}]` so the worker takes its "data supplied directly"
  branch and writes bytes to its own FS — no fetch. (`createWorker` flattens a
  `{code,data}` first-arg back to `"eng"`, which is why the swap happens at the
  message level.)
- **Accuracy:** OCR renders the page to an offscreen canvas at ~300 DPI (not the
  low-res display canvas) and uses PSM 3 + `preserve_interword_spaces`. Took
  "Risk"→"Fisk" garble up to genuinely usable text.

Full detail is in `docs/AUDIT_NOTE.md`.

---

## Local dev / deploy cheatsheet

Authoritative working copy: `C:\Users\Maurice\dev\vault-clean` (clean git clone).
The other folder, `C:\Users\Maurice\dev\OnDevicePDF`, is scratch.

```powershell
cd C:\Users\Maurice\dev\vault-clean
npm install
npm run dev        # local preview at http://127.0.0.1:8787 — runs the REAL Worker + headers
npx wrangler login # already done; re-run if auth expires
npm run deploy     # ship to the edge
```

Notes:
- Wrangler is authorized to the Cloudflare account already.
- `npm run dev` is the only way to test with the **header** CSP active (the old
  `python -m http.server` sent no CSP header — meta tag only).
- After any editor change: regenerate `public/SHA256SUMS.txt`
  (`cd public && sha256sum pdf-editor-*.html index.html > SHA256SUMS.txt`) and
  commit. `.gitattributes` marks those files `-text` so bytes/hashes stay stable
  cross-platform — don't remove that.

---

## Open backlog (all optional — nothing blocks anything)

1. **`build/` pipeline not populated.** The shipped editors were produced by
   one-off patch scripts, not a reproducible `build.py`. Until that exists, the
   "rebuild from source and verify the hash" promise is aspirational (the docs
   now flag this honestly). Closing it is what makes reproducibility fully true.
2. **`ocr-fast` removed.** The deliberately-smaller "fast OCR" build was never
   finished — it referenced its OCR assets from a CDN rather than inlining them,
   so OCR failed closed under the CSP. Rather than inline assets to produce a
   build that would just duplicate `ocr-full`, it was retired. If a genuinely
   lighter OCR variant is wanted later (e.g. a smaller/faster language model
   trading accuracy for size), that's a real feature to build from scratch — not
   a matter of un-flagging the old file.
3. **Stray files:** `LICENSE` (root) and `public/.assetsignore` came from an old
   archive, not authored as part of the project — decide keep/remove.
4. **True structural editing** (drag existing boxes, edit table cells, reflow
   paragraphs) is explicitly out of scope for the browser tool — it needs a
   native engine. The separate OnDeck/Python project is the right home for that
   and for native (no-browser-limits) OCR.

---

## Non-negotiable invariants (don't let these drift)

From `docs/PROJECT_PROMPT.md` — read it before changing anything:

1. No network egress of user content, ever.
2. `connect-src blob:` stays (means: no remote network; blob: self-reads only).
   Meta tag and Worker header must stay identical. Don't add `'unsafe-eval'` or
   remote sources.
3. Everything inlined; nothing fetched at runtime. No CDN links, web fonts,
   analytics, error SDKs, or off-device `<script src>`/`fetch()`.
4. No telemetry of any kind (the Verify-privacy self-test is the only allowed
   network *attempt*, and it's meant to fail).
5. No server-side processing of user files. The Worker only serves files +
   stamps headers.
6. Verifiability is a feature (panel + offline test + hashes). Regenerate hashes
   after any editor change.
7. **Honesty over polish.** If a feature can't meet the invariants, make it
   compliant or don't ship it — never paper over a gap with a reassuring label.
   (Retiring the unfinished `ocr-fast` build instead of dressing it up was an
   application of this.)
