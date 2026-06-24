# Audit Note

This file exists so the privacy claims are checkable, not just asserted. It
records (a) the per-build compliance status and (b) an explanation of the
off-device-looking strings in the editor files, so an auditor can confirm which
ones do — and don't — cause runtime network egress.

Audited build hashes are in [`../public/SHA256SUMS.txt`](../public/SHA256SUMS.txt).
If a file's hash doesn't match, this audit doesn't apply to it.

## Summary

| Build | Privacy holds? | OCR actually works offline? | Status |
|-------|----------------|------------------------------|--------|
| `pdf-editor-lean.html` | ✅ yes | n/a (no OCR) | ✅ Fully self-contained |
| `pdf-editor-ocr-full.html` | ✅ yes (CSP blocks egress) | ✅ yes — model + worker + core all inlined | ✅ Self-contained (offline test recommended) |
| `pdf-editor-ocr-fast.html` | ✅ yes (CSP blocks egress) | ❌ no — nothing inlined | ⚠️ Not compliant |

"Privacy holds" everywhere because `connect-src blob:` is present in every
build. It blocks every network destination (any `http(s)://`/`ws://` server);
the only thing it permits is the page reading its own in-memory inlined data via
`blob:` URLs, which never touch the network. The OCR builds differ in whether OCR is
genuinely *inlined and working* versus *broken because its assets can't be
fetched*. Per invariant 7, a build whose OCR only "works" by reaching a CDN is
not allowed to be advertised as a working offline OCR build.

## `pdf-editor-lean.html` — ✅ verified self-contained

- `connect-src blob:` present as a `<meta>` tag (blocks all remote network; the
  `blob:` scheme only allows reading the app's own inlined data).
- No CDN URLs. No runtime asset fetching.
- `langPath` / `workerPath` / `traineddata` appear only as inert identifiers
  inside the bundled pdf.js; lean has no OCR code path that fetches them.
- Runs fully offline. This is the recommended default.

## `pdf-editor-ocr-full.html` — ✅ self-contained (all three OCR assets inlined)

All three OCR assets are inlined; nothing is fetched at runtime:

- **Language model** — `eng.traineddata` (~14.5 MB, gzipped) is embedded in a
  `<script id='tess-eng-data' type='text/plain'>` block, decoded at runtime,
  exposed as a blob URL, and passed via `langPath` (with `gzip:true`).
- **Worker script** — embedded in `<script id='tess-worker-src'>`, turned into a
  blob URL, and passed via `workerPath`.
- **WASM core** — tesseract.js-core 5.1.0 `simd-lstm` (the variant every modern
  Chrome/Edge/Firefox/Safari selects). The core loader (`*.wasm.js`) and its
  `.wasm` binary are both inlined inside the worker source. See the wiring below.

### How the three assets are inlined (the non-obvious part)

The stock tesseract.js worker fetches all three OCR assets at runtime: the WASM
core (`importScripts` of a jsdelivr URL, whose Emscripten loader then fetches a
sibling `.wasm`), the worker script, and the language model. Each had to be
rewired to load from in-memory bytes instead, and several browser realities made
this fiddly (blob URLs are realm-scoped to the context that created them; the OCR
work happens inside a Web Worker, not the main thread).

**WASM core.** The Emscripten loader gates all `.wasm` fetching on its
`wasmBinary` variable (`pa`). Its line `b.wasmBinary&&(pa=b.wasmBinary)` is
patched to also seed `pa` from an inlined global
`self.__VAULT_CORE_WASM__` (the `.wasm` bytes as base64). Every `.wasm`
fetch/stream path is guarded by `!pa`/`pa||…`, so once `pa` is seeded **no
network call loads the wasm**. The patched loader (core bytes embedded) is
injected at the top of the worker source so `self.TesseractCore` is already
defined; `getCore`'s `importScripts` branch is patched to early-return when
`TesseractCore` exists, so it never `importScripts` a (realm-invalid) blob.

**Worker script.** Embedded in `<script id='tess-worker-src'>`, turned into a
same-realm blob URL on the main thread, and passed as `workerPath`.

**Language model.** Embedded in `<script id='tess-eng-data'>` (~14.5 MB gzipped),
decoded to bytes at runtime. The model is **not** handed to the worker as a
`langPath` URL — a blob URL minted on the main thread is invalid inside the
worker's realm, which made the worker fall back to a (CSP-blocked, then
realm-broken) fetch. Instead, Vault intercepts the OCR worker's `postMessage`:
when tesseract sends its `loadLanguage` message with `langs:"eng"`, the wrapper
swaps in `[{code:'eng', data:<bytes>}]`. The worker then takes its
"data supplied directly" branch (`a = e.data`) and writes the bytes to its own
in-memory FS — **no fetch**. (`createWorker` flattens a `{code,data}` first-arg
back to the string `"eng"`, which is why the swap is done at the message level,
not via the public API.) With `gzip:true` the worker gunzips the bytes itself.

The jsdelivr default strings still appear in the bundled library code
(`tesseract.js@v…`, `tesseract.js-core@v…`, `@tesseract.js-data/…`) but are
**dead branches** — the worker, core, and model are each supplied from in-memory
bytes before any fetch fires. And `connect-src blob:` blocks every remote
destination regardless. The inlining makes OCR *work* offline; the CSP makes
leakage *impossible*.

> CSP note: the OCR build's `script-src` includes `'wasm-unsafe-eval'` (permits
> WebAssembly compilation, **not** general `eval()`), and `connect-src` is
> `blob:` (so the worker/model/core blobs can be read in-tab). Neither weakens
> the no-egress guarantee: no remote origin is ever permitted.

### Verify it (the real proof)

Open `pdf-editor-ocr-full.html`, **disconnect from the internet**, load a scanned
PDF, and run OCR. If it produces text offline, the inlining is correct. (Code-
level review confirms no reachable network path for the core; the offline run is
the user-facing proof the project is built around.)

The core files were obtained at build time via `npm pack tesseract.js-core@5.1.0`
(authoritative npm artifact; note tesseract.js v5 pairs with core **5.1.0**, not
5.0.4 — there is no 5.0.4 core). Build-time downloads are fine; runtime ones are
the thing the project forbids.

## `pdf-editor-ocr-fast.html` — ⚠️ not compliant (nothing inlined)

The OCR engine uses tesseract.js's stock CDN paths for **all three** assets — it
attempts to fetch from the public internet at first use:

- worker — `cdn.jsdelivr.net/npm/tesseract.js@v.../dist/worker.min.js`
- WASM core — `cdn.jsdelivr.net/npm/tesseract.js-core@v...`
- language model — `cdn.jsdelivr.net/npm/@tesseract.js-data/.../*.traineddata`

There is no inlined base64 blob for the core or model and no `corePath`/`langPath`/
`workerPath` override to local sources. Breaks invariant 3 (and invariant 7 if
advertised as working). `connect-src blob:` blocks the remote fetches, so no data leaks, but OCR does
not function. Same remediation as `ocr-full`, plus inlining the model and worker
that `ocr-full` already inlines.

## External-looking strings, explained

The editor files contain URLs that look like network endpoints. Categorized:

- **XML namespace / schema URIs** — `http://www.w3.org/...`, `http://ns.adobe.com/...`,
  `http://www.xfa.org/...`. XML namespace identifiers used by PDF/XMP parsing in
  pdf.js. String labels, never fetched.
- **License header URL** — `http://www.apache.org/licenses/LICENSE-2.0` in bundled
  license comments. Inert text.
- **Source-repo reference** — `https://github.com/Hopding/pdf-lib`. Inert text.
- **Self-test URLs** — `https://example.com` / `.../collect?leak=1` power the
  Verify-privacy panel's deliberate "try to phone home" button, which exists to
  demonstrate the browser blocking the request. This is the one allowed network
  *attempt* (invariant 4), meant to fail.
- **Template-literal fragments** — `http://${...}` inside library code; string
  builders, not live endpoints.
- **jsdelivr CDN URLs** — these are NOT inert. They are the OCR-asset fetches
  described above: fully active in `ocr-fast`, and the WASM-core fetch in
  `ocr-full`. They are the reason those builds are flagged.

## How to reproduce this audit

```bash
# Off-device-looking URLs in a build:
grep -oiE "https?://[^\"'\`) ]+" public/pdf-editor-lean.html | sort -u

# CSP meta tag present? (should print: connect-src blob:)
grep -o "connect-src blob:" public/pdf-editor-lean.html

# Runtime CDN asset paths? (lean prints nothing)
grep -oiE "cdn\.[a-z]+|corePath|langPath|workerPath" public/pdf-editor-ocr-full.html

# Are the OCR assets inlined? (ocr-full: model + worker + core all yes)
#   tess-eng-data = model, tess-worker-src = worker, __VAULT_CORE_WASM__ = core
grep -oiE "tess-eng-data|tess-worker-src|__VAULT_CORE_WASM__" public/pdf-editor-ocr-full.html

# Verify published hashes:
cd public && sha256sum -c SHA256SUMS.txt
```
