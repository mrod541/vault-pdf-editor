# Handoff — Text overlay feature (lean editor)

**Date:** 2026-06-25
**Scope:** `public/pdf-editor-lean.html` only. No other editor or doc touched.
**Status:** Integrated, save path tested headlessly, syntax-checked. Pending your
in-browser feel-check + deploy.

---

## What shipped

The Text tool in the lean editor was upgraded from a fixed-size, Helvetica-only
contentEditable into a direct-manipulation text box:

- **Click** a page (Text tool active) to drop a box and type.
- **Drag the box** to move it.
- **Drag the bottom-right corner** to scale the font.
- **Drag the right-side grip** to set a wrap width; text wraps onto multiple lines.
- **Font + size** are settable per box, two synced ways: a floating control that
  appears above the selected box, and a font/size pair in the toolbar (dims when
  nothing is selected).
- Fonts are the three embed-free PDF Standard-14 faces: **Helvetica, Times,
  Courier**. They embed *by reference* — zero `/FontFile` streams, so no font
  bytes are added and the file stays far under Cloudflare's per-asset cap.

The old contentEditable text path was replaced, not duplicated — there is one
text system, wired to the file's existing `S.edits[pageIndex].texts[]` store,
palette, and `1/S.scale` save math.

## Why this approach

The lean editor already had a working (but basic) text feature. Rather than bolt
on a separate module — which would have created two competing text systems — the
new interaction model was grafted onto the existing foundations. Smaller change,
one code path, reuses the proven save pipeline.

## Files changed

| File | Change |
|------|--------|
| `public/pdf-editor-lean.html` | Text engine rewrite (CSS + JS), font-aware save path, toolbar font controls |
| `public/SHA256SUMS.txt` | Lean hash regenerated (editor bytes changed) |

**New lean SHA-256:**
```
ae3fb0e465da18c24d6805a126097fbd0a1ac117d5d256294996a37992a5585c  pdf-editor-lean.html
```

## Privacy invariants — unaffected

- `connect-src blob:` CSP intact (meta tag + Worker header unchanged).
- No new network calls, no CDN, no fonts fetched at runtime — the three faces are
  pdf-lib StandardFonts, drawn from the already-inlined library.
- Verify panel, offline test, and hashes all still apply.

## Testing done

- **Save path (headless, real pdf-lib):** all three fonts resolve to
  `/Helvetica`, `/Times-Roman`, `/Courier`; multi-line + word-wrap both correct
  (a long line broke into 3 using real glyph metrics); **zero** `/FontFile`
  streams confirmed in the inflated PDF.
- **Syntax:** app script passes `node --check`.
- **Sanity:** AirLock branding intact, no `vault` strings, CSP present.

## Not yet tested (your call)

- **Drag feel in a real browser** — corner-scale sensitivity is ~0.25 pt per
  pixel dragged. If it feels twitchy or sluggish, it's a one-line tweak in
  `startTextDrag` (the `* 0.25` factor).
- Grip grab-target sizes on a real page at real zoom.

## Deploy steps (PowerShell)

```powershell
# 1. Drop the new file in
Copy-Item "$HOME\Downloads\pdf-editor-lean.html" `
  "C:\Users\Maurice\dev\airlock-clean\public\pdf-editor-lean.html" -Force

# 2. Test over http (NOT file://)
cd C:\Users\Maurice\dev\airlock-clean
npm run dev            # http://localhost:8787 — add text, size/move/wrap, Save, check the PDF

# 3. Offline proof: with dev server up, disconnect Wi-Fi and repeat

# 4. Regenerate the lean hash
cd public
$h = (Get-FileHash pdf-editor-lean.html -Algorithm SHA256).Hash.ToLower()
"$h  pdf-editor-lean.html"   # paste over the old lean line in SHA256SUMS.txt

# 5. Commit + deploy
cd ..
git add public/pdf-editor-lean.html public/SHA256SUMS.txt
git commit -m "Text overlay: per-box font/size, drag-move, corner-scale, side-wrap (lean)"
git push
npm run deploy
```

## Follow-ups / backlog

- Apply the same text engine to `pdf-editor-ocr-full.html` (not done this round —
  lean only).
- `build/` generator still unpopulated; the editors are still produced by one-off
  edits rather than a reproducible build.
