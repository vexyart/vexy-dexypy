# Command line

The `vexy-dexypy` command is built with Python Fire. It has five commands. You
can also run it as `python -m vexy_dexypy`.

```bash
vexy-dexypy build URL [flags]
vexy-dexypy read URL [flags]
vexy-dexypy analyze URL [flags]
vexy-dexypy render SLUG STRATEGY [flags]
vexy-dexypy split PDF [flags]
```

Show the help for any command with `--help`, for example
`vexy-dexypy build --help`.

!!! tip
    Fire accepts flag names with underscores (`--no_cache`) or hyphens
    (`--no-cache`). If Fire mistakes a flag value for a positional argument,
    put the flags after a `--` separator.

## `build`

Runs the whole pipeline: read, analyze, normalize, prepare, render and write,
for every selected strategy.

| Flag | Default | Meaning |
|---|---|---|
| `URL` | (required) | `http(s)://` URL or local HTML file path |
| `--out` | `out` | Output root folder |
| `--aspect` | none (`720p`) | Stage profile: `720p`, `1080p` or `810p` |
| `--size` | none | Stage size as `WxH`, for example `1600x900`; overrides `--aspect` |
| `--strategies` | `all` | Comma-separated strategy names, or `all` |
| `--svg` | `False` | Also convert each slide PDF to SVG |
| `--vision` | `False` | Refine slide breaks with a local vision model |
| `--verbose` | `False` | Log at `DEBUG` level instead of `INFO` |
| `--no_cache` | `False` | Ignore cached engine PDFs and render again |
| `--browser_engine` | none (`playwright`) | `playwright`, `playwrightauthor` or `cloakbrowser` |
| `--fetch_mode` | none (`live`) | `live`, `localize` or `offline` |

Flags left unset fall back to the configuration file, then to the built-in
default. See [Configuration](configuration.md).

### Fetch modes

`live`
:   The default. Opens the URL in the browser, captures the rendered DOM, and
    adds a `<base href>` that points at the page's own address. Fonts, CSS and
    images keep loading from the network. If Playwright is not installed, the
    run falls back to `localize`.

`localize`
:   Fetches the HTML with `httpx` and downloads every `<link>`, `<script>`,
    `<img>` and `<source>` asset into `raw/assets/`, including `url()` and
    `@import` targets inside stylesheets. If the fetched body has fewer than
    500 characters of text, the static reader treats the page as
    client-rendered and re-reads it in headless Chromium.

`offline`
:   Runs an external archiver (`single-file` by default, or `monolith`) to
    produce one self-contained HTML file. If the archiver is not on `PATH`, the
    run falls back to `localize`.

Local files always use the static reader, whatever the fetch mode.

### Browser engines

`playwright`
:   Standard Playwright Chromium. The default.

`playwrightauthor`
:   Imports `playwrightauthor` from `private/playwrightauthor/src` (relative to
    the current folder) for persistent browser sessions.

`cloakbrowser`
:   Uses the `cloakbrowser` package to launch a headless browser.

The browser engine applies to the `live` reader and to the in-browser
`vexy-dexyjs` preprocessing. The exporters always launch Playwright Chromium.

### Exit codes

| Code | Meaning |
|---|---|
| `0` | At least one strategy succeeded |
| `1` | Usage error, for example an unknown strategy or a malformed `--size` |
| `2` | No strategy succeeded, no strategy ran, or another `vexy-dexypy` error stopped the run |

## `read`

Runs stage 1 only. It fetches the page, classifies it, writes
`_meta/page.json`, and prints the path of the captured HTML.

Flags: `--out`, `--verbose`, `--browser_engine`, `--fetch_mode`.

```bash
vexy-dexypy read https://www.vexy.art/lines/
```

## `analyze`

Runs stages 1 and 2. It writes `_meta/page.json` and `_meta/slideplan.json`,
then prints the slide count and the break offsets in pixels.

Flags: `--out`, `--aspect`, `--size`, `--vision`, `--verbose`,
`--browser_engine`, `--fetch_mode`.

```bash
vexy-dexypy analyze https://www.vexy.art/lines/ --aspect 1080p
```

## `render`

Runs stages 4 to 6 for one strategy, using the sidecars of an earlier run. It
reads `_meta/page.json` and `_meta/slideplan.json`, and uses
`normalized/normalized.html` when that file exists.

| Argument or flag | Default | Meaning |
|---|---|---|
| `SLUG` | (required) | The folder name under `--out`, for example `www-vexy-art-lines` |
| `STRATEGY` | (required) | One strategy name |
| `--out` | `out` | Output root folder |
| `--svg` | `False` | Also write SVG files |
| `--verbose` | `False` | Debug logging |

```bash
vexy-dexypy render www-vexy-art-lines playwright --svg
```

## `split`

Runs stage 6 only on any PDF. It writes `01-slide.pdf`, `02-slide.pdf`, … and an
`index.html` preview into `--out`.

| Argument or flag | Default | Meaning |
|---|---|---|
| `PDF` | (required) | Multi-page PDF to split |
| `--out` | `.` | Output folder |
| `--svg` | `False` | Also write SVG files |
| `--prefix` | `slide` | File name part after the number |
| `--verbose` | `False` | Debug logging |

```bash
vexy-dexypy split deck.pdf --out slides --prefix page
```

This writes `slides/01-page.pdf`, `slides/02-page.pdf`, and so on.

## How the slug is made

The output folder name comes from the URL or path. The tool lowercases it,
removes a leading `http://` or `https://`, replaces each run of characters
other than `a`–`z` and `0`–`9` with `-`, trims leading and trailing `-`, and
keeps at most 80 characters.

| Input | Slug |
|---|---|
| `https://www.vexy.art/lines/` | `www-vexy-art-lines` |
| `https://blog.fontlab.com/` | `blog-fontlab-com` |
