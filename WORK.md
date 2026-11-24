<!-- this_file: WORK.md -->

# WORK — progress log

## 2026-11-24 — release instructions and planning cleanup

- The existing `release.yml` publishes through GitHub OIDC. The contributing
  guide now describes that flow. Confirm the PyPI Trusted Publisher for
  `vexyart/vexy-dexypy`, `release.yml`, environment `pypi` before publishing.
- `TODO.md` and `PLAN.md` track the remaining checks and optional work.
  `CHANGELOG.md` records the completed work. Earlier progress notes are in
  Git at `5feafad`.
- `./build.sh docs` exited 0 with zero warnings and rebuilt the site with
  fly-january 0.2.1. `actionlint .github/workflows/release.yml`,
  `bash -n publish.sh`, and `shellcheck publish.sh` each exited 0.
- With browser startup blocked, pytest exited 1 (61 passed, one failed, five
  deselected). The known `test_bubble_relaxes_absolute_positioning` failure
  reproduced in the Python fallback. The five browser cases and browser smoke
  example still need a browser-enabled run.
