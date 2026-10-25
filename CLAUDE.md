# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

"Oyoq O'qi" is a client-only PWA for analysing lower-limb alignment ("leg axis") from a CT scan (MSKT, a DICOM series uploaded as a ZIP) or from a standing full-leg X-ray image. All processing happens in the browser and no data leaves the device. The UI is in Uzbek by default, with Russian as the second language.

## Build / run / test

There is no build step, package manager, linter or test suite in the repo. The files are deployed as-is (static hosting, e.g. GitHub Pages).

- Run locally: `python3 -m http.server 8000` in the repo root, then open http://localhost:8000. Serve over HTTP, not `file://`: the service worker and the PWA manifest need it.
- The core logic is written to load in Node too: the core `<script>` block ends with `module.exports = api`. A comment says this is "for tests", but no tests are committed. To test core math, extract that block (it begins `/* Oyoq O'qi — core:`) into a `.js` file and `require` it. Pass `dicomParser` / `JSZip` on `globalThis` if the code under test needs them.

## Layout

- `index.html` (~2.4 MB) holds the whole app: CSS, markup and every app script inline. Do not `Read` it whole. Lines 411 and 415 alone are ~2 MB of base64 data. Use `grep -n` and `sed -n 'A,Bp' | cut -c1-200` to look at regions.
- Vendored, minified libraries, loaded via `<script src>` and never edited: `jszip.min.js`, `dicomParser.min.js`, `three.min.js`, `GLTFLoader.js`.
- `sw.js`: the service worker. It uses network-first for the HTML and cache-first for everything else. **Bump the `V` cache name (`oyoq-vN`) whenever any shipped file changes**, or installed PWAs keep serving stale assets. Add any new static file to the `F` list.
- `manifest.json` and the `icon-*.png` files hold the PWA metadata.

## Script blocks inside `index.html`, in load order

1. **Language bootstrap**: sets `window.OYOQ_LANG` from `localStorage['oyoq-lang']` (default `'uz'`).
2. **jpeg-lossless-decoder-js**: an inline vendored copy that decodes compressed DICOM transfer syntaxes.
3. **Embedded data**:
   - `window.OYOQ_TPL_B64` is a base64 GLB anatomical leg model, used as the 3D template through `THREE.GLTFLoader`.
   - `window.OYOQ_ROT_REFS` holds base64 JPEG reference images for the rotation tab.
   Treat both as opaque blobs.
4. **Core (`window.OyoqCore`)**: pure logic with no DOM access.
   - Reading DICOM from a ZIP (`readZip`, `buildStudy`), with HU conversion and bone threshold `BONE_HU`.
   - The DRR-style projection and building a study from a 2D image or localizer.
   - Automatic landmark detection (`autoLandmarks`).
   - Measurements (`measureLeg`) against `NORMS`: HKA, mLDFA, mMPTA, JLCA, LDTA, MAD.
   - `classify` and `risks`.
   - Rotation measurements (`rotation`, `ROT_NORMS`, Bagirov's α/β/γ convention).
   - The surface mesh for 3D, the demo-patient generator (`makeDemoSlices`) and `reportText`.
5. **App/UI**: an IIFE that consumes `OyoqCore` as `C`. All UI state lives in one object `S`.
   - Five tabs: `upload`, `points`, `result`, `3d`, `rot` (`showTab`).
   - A canvas editor for the 7 landmarks per leg (`KEYS = hip, f1, f2, t1, t2, a1, a2`), separately for leg `R` and leg `L`.
   - A Three.js 3D view (`build3D`) and the rotation tab, which persists to `localStorage['oyoq-rot']`.
   - It exposes `window.__oyoqRerender` so the language switcher can re-render.
6. **i18n**: a `RU` dictionary keyed by the exact Uzbek source string. `walk()` replaces static DOM text in place.
7. **Service worker registration.**

## Conventions

- **Coordinates:**
  - Image pixels, with x to the right and y downward.
  - x to the right is the **patient's left** side.
  - Millimetres are `px * mmpp`.
  - Math code flips y so that up is positive.
- **Sign convention:** HKA `+` is varus and `−` is valgus.
- **Bilingual text:**
  - Dynamic strings are written inline as `L('uz text', 'ru text')`.
  - Static HTML text is written in Uzbek and gets a matching entry in the `RU` map in the i18n block. The key must match the Uzbek text exactly, or it won't be translated.
  - Landmark descriptions live in `LM_.uz` / `LM_.ru`.
- Medical content (norm ranges, landmark instructions, the rotation sign rules) is domain-specific. Don't change values without a source.
