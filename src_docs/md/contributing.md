# Contributing

The design contract is the 24-chapter specification in
[`spec/`](https://github.com/vexyart/vexy-dexypy/tree/main/spec). Read
`spec/00-tldr.md` and `CLAUDE.md` first. Implement against a chapter; if a
change must deviate from it, update that chapter in the same change.

## Set up a development environment

1. Clone the repository and enter it:

    ```bash
    git clone https://github.com/vexyart/vexy-dexypy.git
    cd vexy-dexypy
    ```

2. Install the package with all extras and the dev tools:

    ```bash
    uv sync --extra all
    ```

3. Install Chromium for Playwright:

    ```bash
    uv run playwright install chromium
    ```

## Run the checks

The test suite uses the HTML fixtures in `tests/fixtures/` and mocks network
calls:

```bash
uv run --extra all pytest -v
```

`./test.sh` runs `ruff check`, the tests, and the offline example
`examples/build_local.py`.

CI (`.github/workflows/ci.yml`) runs the tests on Ubuntu, macOS and Windows
with Python 3.12 and 3.13, and a lint job with `ruff check`,
`ruff format --check` and `mypy`.

## Examples

The `examples/` folder holds runnable scripts that double as smoke tests. They
write to `examples/output/`, which git ignores.

```bash
uv run examples/build_local.py           # offline fixture, no network
uv run examples/webflow_retro_poster.py  # live Webflow page
```

`examples/expected/` holds captured manifests of real output for the live
examples.

## Build the package and the documentation

The root `build.sh` builds both:

```bash
./build.sh            # documentation and package
./build.sh docs       # documentation only, into docs/
./build.sh package    # sdist and wheel into dist/, then twine check
```

The documentation source is Markdown in `src_docs/md/`. `src_docs/build.sh`
builds it into `docs/` with ProperDocs and the MaterialX theme in strict mode,
and `src_docs/build.sh serve` starts a live preview at
`http://127.0.0.1:8000`. GitHub Pages serves `docs/` at
<https://vexy.dev/vexy-dexypy/>. The `Docs` workflow rebuilds `docs/` when
`src_docs/` changes on `main`.

## Versions and releases

The version comes from git tags through `hatch-vcs`. At build time it is written
to `src/vexy_dexypy/__version__.py`, which git ignores, and
`vexy_dexypy.__version__` reads it.

`./publish.sh` makes a release:

1. `uvx gitnextver` commits pending changes, tags the next `vX.Y.Z` and pushes.
2. The script checks that `HEAD` carries a `vX.Y.Z` tag.
3. `./build.sh package` builds and checks the distributions.
4. `uv publish` uploads them to PyPI. It needs `UV_PUBLISH_TOKEN` or another
   `uv publish` credential.

Pushing the tag also starts `.github/workflows/release.yml`, which builds the
distributions, publishes them to PyPI with trusted publishing (skipping files
that already exist), and creates a GitHub release with the matching
`CHANGELOG.md` section.

## Conventions

- Python 3.12+, `uv` for every dependency change.
- Type hints everywhere; dataclasses for the pipeline types in `model.py`.
- Typed errors from `vexy_dexypy.errors`. A failed strategy is a
  `DeckResult(ok=False)`, never an aborted run.
- Every external program call has a timeout and reports its stderr.
- Each source file carries a `this_file` comment with its path.
- Record changes in `CHANGELOG.md` and new dependencies, with their licences,
  in `DEPENDENCIES.md`.
