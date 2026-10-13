# Quick start

This page builds decks from one web page and then from a local file.

## Prerequisites

- `vexy-dexypy` installed (see [Installation](installation.md)).
- Chromium installed with `playwright install chromium`.
- Network access for the URL example.

## Build decks from a URL

1. Run `build` with the URL:

    ```bash
    vexy-dexypy build https://www.vexy.art/lines/
    ```

    The tool fetches the page, classifies it, plans the slide breaks at the
    default 1280×720 stage, and runs every available strategy.

2. Read the summary table. Each row shows a strategy, a ✓ or ✗, the slide
   count, and the output folder or the error:

    ```text
    strategy     result  slides  output / error
    playwright   ✓           11  out/www-vexy-art-lines/playwright
    reveal       ✓            9  out/www-vexy-art-lines/reveal
    2/2 strategies succeeded.
    ```

    The slide counts depend on the live page and on which engines you have
    installed.

3. Open a strategy's preview, for example
   `out/www-vexy-art-lines/playwright/index.html`, to page through its slides.

4. Pick the best rendering of each slide from the strategy folders. Every
   folder uses the same numbering: `01-slide.pdf`, `02-slide.pdf`, and so on.

## Choose strategies and stage size

Run only two strategies at 1920×1080 and also write SVG files:

```bash
vexy-dexypy build https://blog.fontlab.com/ \
    --strategies vivliostyle,playwright --aspect 1080p --svg
```

- `--strategies` takes a comma-separated list. An unknown name stops the run
  with a usage error that lists the known strategies.
- `--aspect` takes `720p`, `1080p` or `810p`. `--size 1600x900` sets any other
  size and overrides `--aspect`.
- `--svg` needs the `vexy-pdfsvgpy` package. Without it the run logs a warning
  and keeps the PDFs.

## Build decks from a local file

A path that does not start with `http://` or `https://` is read as a local
file. The repository ships a small fixture you can use:

```bash
vexy-dexypy build tests/fixtures/generic_article/index.html --out out
```

The static reader copies the file into `out/<slug>/raw/` and downloads any
remote assets it references.

## Re-run part of the pipeline

Each stage writes JSON sidecars into `_meta/`, so you can repeat later stages
without fetching again.

- Re-render one strategy from the existing sidecars:

    ```bash
    vexy-dexypy render www-vexy-art-lines reveal
    ```

- Split an existing PDF into numbered slides with a preview:

    ```bash
    vexy-dexypy split out/www-vexy-art-lines/playwright/_combined.pdf \
        --out out/www-vexy-art-lines/playwright --svg
    ```

See [Command line](cli.md) for every command and flag.
