# 🛡️ Start Here

Welcome. This repo holds **Vault**, a PDF editor that runs entirely in your
browser and *cannot* send your file anywhere — a guarantee your browser
enforces, not one you have to take on trust.

If you read nothing else, read this page. It points you to the right next step
depending on who you are.

> **Build status at a glance.** The privacy guarantee (`connect-src blob:` —
> blocks every network destination; permits only reading the app's own inlined
> data) holds for **all** builds — nothing can leave your device. `lean` and
> `ocr-full` are complete and run fully offline (yes, OCR included in
> `ocr-full`). `ocr-fast` is **not finished** — its OCR engine still expects to
> load from the internet, which the privacy policy (correctly) blocks, so OCR
> doesn't work there yet. Details: [`docs/AUDIT_NOTE.md`](docs/AUDIT_NOTE.md).

---

## I just want to use it

Open a file from `public/` directly in your browser, or visit the deployed site
at **https://vault-pdf-editor.mozilla614.workers.dev**:

- **`pdf-editor-lean.html`** — everyday editing, smallest file, fully offline.
  **Start here.** ✅
- **`pdf-editor-ocr-full.html`** — editing plus OCR, fully inlined and working
  offline. Largest file, best OCR accuracy. ✅
- **`pdf-editor-ocr-fast.html`** — intended for quick OCR, but OCR is **not
  working yet** (none of the OCR engine is bundled). ⚠️

Drop a PDF in, edit, save. Nothing uploads. Click **"Verify privacy"** in the
app to prove it to yourself.

→ Full usage details: [`docs/INSTRUCTIONS.md`](docs/INSTRUCTIONS.md)

---

## I want to deploy my own copy

You'll put the files on Cloudflare Workers (free tier is plenty). It takes about
ten minutes and the steps are copy-paste.

→ Get it onto GitHub first: [`docs/GITHUB_SETUP.md`](docs/GITHUB_SETUP.md)
→ Deployment walkthrough: [`docs/INSTRUCTIONS.md`](docs/INSTRUCTIONS.md#deploy-to-cloudflare)

---

## I want to understand or trust the privacy claim

The whole project is built so you don't have to trust *me* — you verify.

→ How the guarantee works and how to check it:
[`docs/INSTRUCTIONS.md`](docs/INSTRUCTIONS.md#how-the-privacy-guarantee-works)
→ The audit of every external string in the files, and the per-build status:
[`docs/AUDIT_NOTE.md`](docs/AUDIT_NOTE.md)

---

## I want to rebuild or modify the editor

The HTML files are *generated* from source, so anyone can reproduce them
byte-for-byte and confirm the published hashes. (Note: the `build/` source isn't
populated in this repo yet — see INSTRUCTIONS.)

→ Build instructions: [`docs/INSTRUCTIONS.md`](docs/INSTRUCTIONS.md#rebuild-from-source)

---

## I'm an AI assistant helping with this repo

→ Read [`docs/PROJECT_PROMPT.md`](docs/PROJECT_PROMPT.md) first. It explains the
project's non-negotiable privacy invariants so you don't accidentally break them
(e.g. by adding a CDN link or an analytics snippet). In particular, invariant 7
(honesty over polish) is why the OCR builds above are labeled "not working yet"
instead of being quietly shipped.

---

## Repo map

```
vault-pdf-editor/
├── START_HERE.md              ← you are here
├── README.md                  project overview
├── wrangler.jsonc             Cloudflare Worker config
├── package.json
├── src/
│   └── index.js               Worker: serves files + stamps security headers
├── public/                    what gets deployed (the editor builds)
│   ├── index.html             landing page
│   ├── pdf-editor-*.html      the three editors (lean works; OCR builds flagged)
│   └── SHA256SUMS.txt         hashes for verification
├── docs/
│   ├── INSTRUCTIONS.md        usage + deploy + build, all in one
│   ├── GITHUB_SETUP.md        how to create and push the repo
│   ├── PROJECT_PROMPT.md      the privacy invariants (for humans and AIs)
│   └── AUDIT_NOTE.md          transparency: every external string + build status
├── build/                     (placeholder) build source — not yet populated
└── .github/workflows/
    └── deploy.yml             optional auto-deploy on push
```
