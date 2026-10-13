# Configuration

Most runs need no configuration file. Flags and defaults cover the common
cases. A TOML file lets you fix a house style for a project or for your user
account.

## Where settings come from

`vexy-dexypy` resolves settings once at startup, in this order of precedence:

1. Command-line flags.
2. `./vexy-dexypy.toml` in the current folder.
3. `~/.config/vexy-dexypy/config.toml`.
4. Built-in defaults.

The project file and the user file are merged section by section. Within a
section, a key in the project file replaces the same key in the user file.

A file that is not valid TOML stops the run with a usage error that names the
file.

## Example file

```toml
# vexy-dexypy.toml
[output]
dir = "out"
svg = false

[stage]
aspect = "720p"          # or "1080p", "810p"
# size = "1600x900"      # overrides aspect when present

[strategies]
enabled = ["playwright", "vivliostyle", "reveal"]   # or "all"

[vision]
enabled = false
model = "minicpm-v-4.6"
endpoint = "http://localhost:11434"

[engines]
prince_path = ""
node_bin = "node"

[browser]
engine = "playwright"    # or "playwrightauthor", "cloakbrowser"

[fetch]
mode = "live"            # or "localize", "offline"
tool = "single-file"     # or "monolith" (offline mode only)
```

## Keys

| Section | Key | Default | Flag that overrides it |
|---|---|---|---|
| `output` | `dir` | `"out"` | `--out` |
| `output` | `svg` | `false` | `--svg` |
| `stage` | `aspect` | none | `--aspect` |
| `stage` | `size` | none | `--size` |
| `strategies` | `enabled` | `"all"` | `--strategies` |
| `vision` | `enabled` | `false` | `--vision` |
| `vision` | `model` | `"minicpm-v-4.6"` | — |
| `vision` | `endpoint` | `"http://localhost:11434"` | — |
| `engines` | `prince_path` | `""` | — |
| `engines` | `node_bin` | `"node"` | — |
| `browser` | `engine` | `"playwright"` | `--browser_engine` |
| `fetch` | `mode` | `"live"` | `--fetch_mode` |
| `fetch` | `tool` | `"single-file"` | — |

`strategies.enabled` accepts a TOML list or a comma-separated string.

!!! note
    The Prince exporter currently locates the binary through the
    `VEXY_DEX_PRINCE_PATH` environment variable or `prince` on `PATH`. The
    `engines.prince_path` key is read into the settings but the exporter does
    not use it yet.

## Stage size

The stage is the slide rectangle in CSS pixels. The same width and height are
used for the browser viewport during analysis, for the `@page` size in the
paged-media CSS, and for the PDF page size in every exporter.

| Profile | Width × height |
|---|---|
| `720p` (default) | 1280 × 720 |
| `1080p` | 1920 × 1080 |
| `810p` | 1440 × 810 |

All three profiles are 16:9. For another shape, give a size such as
`--size 1024x768`. A size wins over an aspect profile. The size string accepts
`x` or `×` between the numbers, and both numbers must be positive integers.

Errors you can see here:

- `unknown --aspect '4:3'; choose from ['1080p', '720p', '810p'] or pass --size WxH`
- `bad --size '1280'; expected WxH like 1280x720 or an --aspect from [...]`
- `--size must be positive, got '0x720'`

## Strategy selection

- `all` runs every exporter registered as a plugin whose `available()` check
  passes. The classifier's recommended order comes first; other strategies
  follow.
- An explicit list runs exactly those strategies. A name that is not
  registered stops the run with a usage error. A registered strategy whose
  program is missing still runs and fails in the summary table with the
  engine's error message.

The recommended order per page type:

| Page type | Recommended first |
|---|---|
| `webflow` | `playwright`, `reveal` |
| `mkdocs-material` | `vivliostyle`, `playwright` |
| `docusaurus` | `vivliostyle`, `playwright` |
| `framer` | `playwright`, `reveal` |
| `bubble` | `playwright`, `vivliostyle` |
| `generic` | `vivliostyle`, `playwright` |

## Caching

After an engine writes its combined PDF, the orchestrator stores a copy in
`out/<slug>/_meta/cache/`. The cache key joins the page's content hash, the
strategy name and the stage size. On the next run with the same key, the cached
PDF is reused and the engine is not called. Pass `--no_cache` to render again.

Vision results are cached separately in `out/<slug>/_meta/vision/`, keyed by a
hash of the page screenshot and the model name.
