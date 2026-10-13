# Troubleshooting

Run the failing command again with `--verbose` first. Debug logging shows which
reader, importer and strategies were chosen and why a step fell back.

## A strategy is missing from the run

With `--strategies all`, only strategies whose `available()` check passes run.
Check the list:

```bash
vexy-dexypy build --help
```

The help ends with `Available strategies: …` and, when some are missing,
`Unavailable (install deps): …`. Install the program the strategy needs; see
[Installation](installation.md).

## `usage error: unknown strategy 'x'; known: [...]`

A name in `--strategies` is not registered. Check the spelling against the
list in the message. Names are lower case: `playwright`, `vivliostyle`,
`reveal`, `prince`, `impress`, `marp`.

## `engine not found: vivliostyle`

You asked for `vivliostyle` explicitly, but the Vivliostyle CLI is not on
`PATH`. Install it:

```bash
npm install -g @vivliostyle/cli
```

The same pattern applies to `prince` (set `VEXY_DEX_PRINCE_PATH` or put
`prince` on `PATH`) and `marp`.

## Playwright says the browser executable does not exist

Playwright is installed but its Chromium download is not:

```bash
playwright install chromium
```

With `uv run`, use `uv run playwright install chromium` so the browser matches
the environment's Playwright version.

## `live mode needs playwright; falling back to localize`

The default `live` fetch mode needs Playwright. Without it, the run downloads
the page and its assets instead. Install Playwright, or pass
`--fetch_mode localize` to make the choice explicit.

## The fetched page is almost empty

Pages rendered by client-side JavaScript have little text in their HTML. In
`localize` mode, a body with fewer than 500 characters of text makes the static
reader re-read the page in headless Chromium. If that also fails, the log shows
`dynamic escalation failed (...); using static`. Use the default `live` mode
for such pages.

## Fonts or styles are missing in `localize` mode

Asset downloads are best-effort. Each failed asset is logged as
`asset <url> -> HTTP <code>` or `failed to localize asset <url>: …` and the
run continues. The `live` mode loads assets from the original site and avoids
the problem.

## `offline tool 'single-file' not found on PATH; falling back to localize`

`--fetch_mode offline` needs the SingleFile CLI. Install it, or set
`[fetch] tool = "monolith"` in `vexy-dexypy.toml` and install `monolith`.

## Slides break in the wrong places

- Try another stage size: `--aspect 1080p` or `--size 1600x900`.
- Compare strategies. Paged engines follow `<section>` boundaries; `reveal` and
  `impress` take one page per slide of the normalized deck.
- Inspect the plan with `vexy-dexypy analyze <url>`, which prints the break
  offsets, or read `_meta/slideplan.json`.
- Try `--vision` if you run a local vision model.

## `browser preprocessor failed: …; falling back to python sectionizer`

The in-browser `vexy-dexyjs` step failed, often because Chromium is not
installed. The importer used its Python fallback instead, which can give fewer
or different slides. Fix the browser problem and run again with `--no_cache`.

## Old output appears after a change

Engine PDFs are cached in `out/<slug>/_meta/cache/` by content hash, strategy
and stage size. Pass `--no_cache` to render again, or delete the `out/<slug>/`
folder.

## `svg requested but vexy-pdfsvgpy unavailable`

`--svg` needs the `vexy-pdfsvgpy` package in the same environment. Without it,
you still get the PDFs and the preview.

## `vision model unavailable (...); using heuristic plan`

The vision server at `[vision] endpoint` (default `http://localhost:11434`) did
not answer. Start the server or drop `--vision`. When serving MiniCPM-V, turn
off its thinking template; otherwise the JSON answer cannot be parsed.

## Exit code 2

No strategy succeeded, no strategy ran, or a read error stopped the run. The
summary table and the log show the reason for each strategy.
