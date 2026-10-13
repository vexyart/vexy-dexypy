# Architecture

`vexy-dexypy` is a six-stage pipeline. Stages 1 to 3 run once per page. Stages
4 to 6 run once per strategy. Stages pass typed objects from
`vexy_dexypy.model` to each other and write JSON sidecars, so a later stage can
run again without the earlier ones.

This page follows one build through the stages:

```bash
vexy-dexypy build https://www.vexy.art/lines/ --strategies playwright,reveal
```

## Stage 1: read

Input: a `Source`. `Source.parse` sets `kind` to `url` for `http://` and
`https://` input, and to `file` for anything else, and computes the slug
`www-vexy-art-lines`.

The reader (`vexy_dexypy.readers.read`) chooses a reader from the fetch mode:

- Local files: `StaticReader`.
- `offline`: `OfflineReader`, which runs `single-file` or `monolith`.
- `live` (default): `LiveReader`, if Playwright is installed.
- Otherwise: the registered reader with the highest `can_read` score. The
  static reader scores `0.5`, the dynamic reader `0.3`.

Output: a `PageDoc` with the captured HTML at `out/www-vexy-art-lines/raw/index.html`,
the asset folder, a content hash, and metadata (`title`, `generator`).

The page is then classified. `classify` lowercases the HTML and counts
fingerprint strings per page type:

| Page type | Fingerprints |
|---|---|
| `webflow` | `data-wf-page`, `w-mod-js`, `data-w-id`, `webflow.js` |
| `mkdocs-material` | `mkdocs-material`, `md-content`, `data-md-component`, `md-typeset` |
| `docusaurus` | `__docusaurus`, `theme-doc-markdown` |
| `framer` | `data-framer-name`, `framer.app` |
| `bubble` | `data-bb-id`, `bubble-element` |

One hit gives confidence `0.5`; each extra hit adds `0.25`, up to `1.0`. The
highest confidence wins; no hits means `generic`. The result is written to
`_meta/page.json`.

## Stage 2: analyze

`paginate.analyze` opens the HTML in Chromium at the stage size (1280×720 by
default) and measures every `section`, `article`, `h1`, `h2`, `h3`, `img`,
`.card` and `[class*="hero"]` element. `plan_breaks` walks the visible elements
from top to bottom:

- An element taller than the stage adds breaks every stage height inside it
  (reason `overflow`).
- An element that starts more than one stage height (less 50 px) below the
  current slide start adds a break at its top (reason `overflow`).
- An `h1`, `h2`, `section` or `article` that starts more than 40% of a stage
  height below the current slide start adds a break (reason is the tag name).

Breaks are sorted and duplicates within 1 px are removed. The slide count is
the number of breaks plus one.

Without Playwright, a static fallback adds one evenly spaced advisory break per
`<section>` or `<h2>` and marks the plan `{"mode": "static-fallback"}`.

With `--vision`, a full-page screenshot is sent to the vision model, and its
`y` values are added as `vision` breaks. The merged breaks are de-duplicated
within 10% of the stage height. If the model is unreachable, the heuristic plan
is kept.

Output: a `SlidePlan`, written to `_meta/slideplan.json`.

## Stage 3: normalize

`pick_importer` chooses the importer whose name matches the page type, or the
`generic` importer. Every importer first tries the in-browser preprocessor: it
injects the bundled `vexy-dexyjs.js` into the page and calls
`window.VexyDexy.preprocess(config)` with the framework name, stage size and
breaks. The returned HTML is a sequence of `<section>` slides.

If the browser step fails, each importer has a Python fallback:

- `webflow`: keeps `<section>` elements that are not menu, nav, footer, mask,
  header or banner chrome, and tags each one light or dark from its background
  colour. With fewer than two sections, it removes chrome and distributes the
  page blocks into the planned slide count.
- `mkdocs-material`: removes the Material header, sidebars, navigation and
  footer, and splits `article.md-content__inner` at `h1` and `h2`.
- `generic`: extracts the article body with `trafilatura` if installed,
  otherwise strips `nav`, `header`, `footer`, `aside`, `script` and `style`,
  then splits at headings.
- `docusaurus`: removes page chrome and splits `.theme-doc-markdown`.
- `bubble`: changes `position: absolute` inline styles to
  `position: relative`, then hands the page to the `generic` importer.
- `framer`: hands the page to the `generic` importer.

Output: `out/www-vexy-art-lines/normalized/normalized.html`. Asset references
are rewritten from `assets/` to `../raw/assets/` so they still resolve.

## Stage 4: prepare

`preexport.prepare` creates `out/www-vexy-art-lines/playwright/` and
`out/www-vexy-art-lines/reveal/`, writes `_paged.css` and `_theme.css` into
each, and builds the engine input for the strategy's chassis (paged, reveal,
impress or marp). See [Engines](engines.md).

Output: a `RenderJob` per strategy.

## Stage 5: render

The orchestrator runs the two strategies concurrently. Each exporter writes
`_combined.pdf`, and a copy goes to `_meta/cache/`.

Local pages are never opened as `file://`. A short-lived HTTP server on
`127.0.0.1` serves the folder two levels above the HTML file, so
`../raw/assets/` links, web fonts, `fetch()` calls and ES modules behave as on
a real site. Navigation waits for the `load` event and then pauses 400 ms.

## Stage 6: write

`writers.write_deck` splits `_combined.pdf` with `pypdf` into
`01-slide.pdf`, `02-slide.pdf`, …, optionally converts them to SVG, and writes
`index.html`. The orchestrator then writes `_meta/run-summary.json`:

```json
{
  "source": "https://www.vexy.art/lines/",
  "framework": "webflow",
  "slides": 12,
  "results": [
    {"strategy": "playwright", "ok": true, "slides": 11, "error": null},
    {"strategy": "reveal", "ok": true, "slides": 9, "error": null}
  ]
}
```

The numbers in this example are illustrative.

## Error types

All errors derive from `VexyDexError` in `vexy_dexypy.errors`:

| Error | Raised when |
|---|---|
| `UsageError` | A flag or config value is invalid (exit code 1) |
| `ReadError` | Fetching, archiving or reading the source fails |
| `NormalizeError` | A normalizer fails |
| `ExportError` | An engine fails; isolated to its strategy |
| `WriteError` | Splitting or SVG conversion fails; isolated to its strategy |
| `VisionError` | The vision model is unavailable or returns unusable data |

## Source layout

```text
src/vexy_dexypy/
  cli.py            # Fire commands
  settings.py       # flags + TOML -> frozen Settings
  model.py          # Source, PageDoc, Break, SlidePlan, Strategy, RenderJob, DeckResult
  orchestrator.py   # stage sequencing and concurrency
  classify.py       # page-type fingerprints
  paginate.py       # slide-break planning
  vision.py         # optional vision refinement
  dom.py            # shared BeautifulSoup helpers
  preexport.py      # CSS and deck wrapping
  writers.py        # split, SVG, preview
  _browser.py       # loopback server, browser launch, vexy-dexyjs injection
  readers/  importers/  exporters/
  assets/           # vexy-dexyjs.js, reveal.js, impress.js
```

The full design is in the 24-chapter specification in the repository's
[`spec/`](https://github.com/vexyart/vexy-dexypy/tree/main/spec) folder.
