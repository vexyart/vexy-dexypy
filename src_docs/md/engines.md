# Engines

A strategy is a named route from the normalized page to a combined PDF. The
strategy name is also the name of its output folder. `vexy-dexypy` registers
six strategies.

| Strategy | Engine | Wrapping (chassis) | Available when |
|---|---|---|---|
| `playwright` | Chromium `page.pdf` | paged | `playwright` can be imported |
| `vivliostyle` | Vivliostyle CLI | paged | `vivliostyle` is on `PATH` |
| `prince` | PrinceXML | paged | `VEXY_DEX_PRINCE_PATH` is set or `prince` is on `PATH` |
| `reveal` | reveal.js in Chromium | reveal | `playwright` and `pypdf` can be imported |
| `impress` | impress.js in Chromium | impress | `playwright` and `pypdf` can be imported |
| `marp` | Marp CLI | marp (Markdown) | `marp` or `npx` is on `PATH` |

## How a strategy runs

For each strategy the orchestrator:

1. Writes the strategy folder `out/<slug>/<strategy>/`, with `_paged.css` (the
   page size rules) and `_theme.css` (the cosmetic layer).
2. Builds the engine input from the normalized HTML: `_input.html`, or
   `_input.md` for Marp.
3. Calls the engine, which writes `_combined.pdf`, or copies the cached PDF.
4. Splits `_combined.pdf` into `01-slide.pdf`, `02-slide.pdf`, …, converts them
   to SVG if requested, and writes `index.html`.

Strategies run concurrently in worker threads. Strategies that drive a browser
share a limit of three at a time. An exception in one strategy becomes a failed
row in the summary; the other strategies continue.

## Paged strategies

The `playwright`, `vivliostyle` and `prince` strategies receive one HTML file
with this CSS added before `</head>` (shown for the 1280×720 stage):

```css
@page { size: 1280px 720px; margin: 0; }
html, body { margin: 0; padding: 0; }
* { -webkit-print-color-adjust: exact; print-color-adjust: exact; }
section, .slide {
  break-after: page;
  page-break-after: always;
  width: 1280px; height: 720px;
  overflow: hidden; box-sizing: border-box;
}
section h2 { break-before: page; page-break-before: always; }
```

Each `<section>` becomes at least one page. Content that does not fit is
clipped by `overflow: hidden`.

The cosmetic layer depends on the page type. Webflow and Framer pages keep
their own CSS and only hide the "Made in Webflow" badge. All other pages get a
small neutral theme: 4% padding, a system sans-serif font, and light or dark
slide backgrounds.

### `playwright`

Opens the input in Chromium at the stage size, switches to `screen` media, and
prints with `page.pdf` at the exact stage size, with backgrounds and zero
margins. The result is close to a vector screenshot of each slide. This is
usually the best choice for designed, script-heavy pages.

### `vivliostyle`

Runs:

```bash
vivliostyle build _input.html -s 1280px,720px -o _combined.pdf
```

with a 180-second timeout. Vivliostyle has strong CSS paged-media support and
suits long-form and documentation pages. It is AGPL-licensed;
`vexy-dexypy` only calls it as a separate program.

### `prince`

Runs the Prince binary with `--page-size=1280px 720px` and a 180-second
timeout. Prince is proprietary and opt-in: it is used only if you set
`VEXY_DEX_PRINCE_PATH` or put `prince` on `PATH`.

## Slide-deck strategies

### `reveal`

Wraps the sections in a reveal.js deck that uses the reveal.js copy shipped
inside the package, so no network is needed. Chromium opens the deck, waits
until `Reveal.getTotalSlides()` reports at least one slide, then prints one PDF
page per slide while stepping with `Reveal.next()`. `pypdf` merges the pages
into `_combined.pdf`. This replaces DeckTape and needs no Node toolchain.

### `impress`

Works the same way with the bundled impress.js: it waits for
`#impress .step` elements, prints one page per step, and moves with
`impress().next()`.

### `marp`

Translates the normalized HTML into Marp Markdown (`_input.md`) and runs:

```bash
marp --pdf _input.md -o _combined.pdf --allow-local-files --no-stdin
```

If `marp` is not installed but `npx` is, it runs
`npx -y @marp-team/marp-cli` with the same arguments.

## Engine errors

Program-based engines (`vivliostyle`, `prince`, `marp`) report one of these
messages in the summary table:

- `engine not found: <program> (...)`: the program is not on `PATH`.
- `<program> timed out after <N>s`
- `<program> failed: <last 500 characters of stderr>`

Browser-based engines report `playwright export failed: …`,
`reveal export failed: …` or `impress export failed: …` with the underlying
Playwright error.

## SVG output

With `--svg`, each single-page PDF is converted with
`vexy_pdfsvgpy.pdf_to_svg`. The preview `index.html` then shows the SVG files
instead of embedding the PDFs. If `vexy-pdfsvgpy` is not installed, or a
conversion fails, the run logs a warning and keeps the PDFs.
