# Installation

`vexy-dexypy` is a Python package with a command-line tool. The core install
gives you the `playwright`, `reveal` and `impress` strategies. Other strategies
need extra programs on your `PATH`.

## Requirements

- Python 3.12 or newer.
- A Chromium build for Playwright. The default `live` fetch mode, the
  pagination probe and the browser-driven exporters all use it.

## Install the package

1. Install the package with `uv`:

    ```bash
    uv pip install vexy-dexypy
    ```

    Or run it without installing:

    ```bash
    uvx vexy-dexypy --help
    ```

2. Install the Chromium browser that Playwright drives:

    ```bash
    playwright install chromium
    ```

3. Check which strategies are available on your machine:

    ```bash
    vexy-dexypy build --help
    ```

    The help text ends with a line such as
    `Available strategies: impress, playwright, reveal.` and, if some are
    missing, `Unavailable (install deps): …`.

## Optional extras

The package defines these extras in `pyproject.toml`:

| Extra | Installs | Used for |
|---|---|---|
| `extract` | `trafilatura` | Article-body extraction in the generic importer's Python fallback |
| `pdf` | `pikepdf` | PDF post-processing |
| `vision` | `instructor` | Vision-model tooling |
| `all` | `playwright`, `trafilatura`, `pikepdf` | Everything above except `vision` |

Install an extra with:

```bash
uv pip install "vexy-dexypy[all]"
```

## Optional external programs

Each of these enables one strategy or feature. A strategy whose program is
missing is skipped when you ask for `all` strategies.

| Program | Enables | How `vexy-dexypy` finds it |
|---|---|---|
| `vivliostyle` (Vivliostyle CLI) | `vivliostyle` strategy | `vivliostyle` on `PATH` |
| `prince` (PrinceXML) | `prince` strategy | `VEXY_DEX_PRINCE_PATH` environment variable, then `prince` on `PATH` |
| `marp` (Marp CLI) or `npx` | `marp` strategy | `marp` on `PATH`; otherwise `npx -y @marp-team/marp-cli` |
| `single-file` or `monolith` | `--fetch_mode offline` | the configured tool on `PATH` |
| `vexy-pdfsvgpy` (Python) | `--svg` output | importable as `vexy_pdfsvgpy` |
| Ollama or another server with an `/api/chat` endpoint | `--vision` | `[vision] endpoint`, default `http://localhost:11434` |

To install the Vivliostyle CLI with npm:

```bash
npm install -g @vivliostyle/cli
```

## Install from source

1. Clone the repository:

    ```bash
    git clone https://github.com/vexyart/vexy-dexypy.git
    cd vexy-dexypy
    ```

2. Create the environment with all extras:

    ```bash
    uv sync --extra all
    uv run playwright install chromium
    ```

3. Run the tool from the checkout:

    ```bash
    uv run vexy-dexypy --help
    ```

The version number comes from the latest git tag (through `hatch-vcs`), so a
checkout without tags reports a development version.
