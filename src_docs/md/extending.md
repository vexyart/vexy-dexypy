# Extending

Readers, importers and exporters are plugins. `vexy-dexypy` finds them through
Python entry points, so another package can add one without changing
`vexy-dexypy` itself.

## Entry-point groups

| Group | Plugin returns | Selected by |
|---|---|---|
| `vexy_dexypy.readers` | a reader | highest `can_read(source)` score (in `localize` mode) |
| `vexy_dexypy.importers` | an importer | `name` equal to the classified page type, else `generic` |
| `vexy_dexypy.exporters` | an exporter | `name` listed in `--strategies`, or `all` with `available()` true |

Each entry point names a class. `vexy-dexypy` calls it with no arguments to
create the plugin. A plugin that fails to load is skipped and logged at
`DEBUG` level.

The built-in plugins are registered in `vexy-dexypy`'s own `pyproject.toml`:

```toml
[project.entry-points."vexy_dexypy.exporters"]
playwright = "vexy_dexypy.exporters.playwright:PlaywrightExporter"
vivliostyle = "vexy_dexypy.exporters.vivliostyle:VivliostyleExporter"
reveal = "vexy_dexypy.exporters.reveal:RevealExporter"
```

## Exporter contract

An exporter has three attributes and two methods:

```python
from pathlib import Path

from vexy_dexypy.model import RenderJob


class MyExporter:
    name = "mine"                  # strategy name and output folder name
    needs_js = False               # True: counts toward the 3-browser limit
    supports_paged_media = True

    def available(self) -> bool:
        """Return True when the engine's dependencies are present."""
        return True

    def export(self, job: RenderJob, out: Path) -> Path:
        """Render job.html_path to a multi-page PDF at `out` and return `out`."""
        ...
        return out
```

The `RenderJob` gives you:

- `job.html_path`: the prepared input file.
- `job.plan.stage_w` and `job.plan.stage_h`: the page size in CSS pixels.
- `job.extra_css`: a list holding the path of `_paged.css`.
- `job.page` and `job.strategy`: the source page and the strategy record.

Raise `vexy_dexypy.errors.ExportError` with a clear message on failure. For
external programs, `vexy_dexypy.exporters._subprocess.run_engine(cmd, timeout=…)`
runs the command with a timeout and turns missing programs, timeouts and
non-zero exits into `ExportError`.

The orchestrator maps strategy names to a chassis (the input format). Names it
does not know get the `reveal` chassis, so a new exporter receives a reveal.js
deck as `_input.html`.

Register the class in your package:

```toml
[project.entry-points."vexy_dexypy.exporters"]
mine = "my_package.exporter:MyExporter"
```

## Importer contract

```python
from vexy_dexypy.model import PageDoc, SlidePlan
from vexy_dexypy.settings import Settings


class MyImporter:
    name = "my-framework"

    def detect(self, page: PageDoc) -> float:
        """Confidence 0..1 that this importer fits the page."""
        return 0.0

    def transform(
        self, page: PageDoc, plan: SlidePlan, settings: Settings | None = None
    ) -> PageDoc:
        """Return a PageDoc whose html_path points at slide-shaped HTML."""
        ...
```

`vexy_dexypy.importers._common.write_normalized(page, html)` writes the HTML to
`out/<slug>/normalized/normalized.html`, fixes asset paths, and returns the
updated `PageDoc`. The helpers in `vexy_dexypy.dom` (`parse`, `drop_chrome`,
`sectionize`, `head_styles`, `wrap_neutral`) cover common DOM work.

An importer is chosen by name, and names come from the classifier. A new page
type therefore also needs a fingerprint rule in `vexy_dexypy.classify`, which
is a change to `vexy-dexypy` itself.

## Reader contract

```python
from vexy_dexypy.model import PageDoc, Source
from vexy_dexypy.settings import Settings


class MyReader:
    name = "mine"

    def can_read(self, source: Source) -> float:
        """Confidence 0..1. The built-in static reader returns 0.5."""
        return 0.0

    def read(self, source: Source, settings: Settings) -> PageDoc:
        """Write the page to settings.out_dir / source.slug / "raw" / "index.html"."""
        ...
```

Readers are ranked only in `localize` mode. The `live` and `offline` modes
call their own readers directly.

## Check that a plugin is found

After installing your package in the same environment, run:

```bash
vexy-dexypy build --help
```

A new exporter appears in the `Available strategies:` line when its
`available()` returns `True`, or in the `Unavailable (install deps):` line
otherwise.
