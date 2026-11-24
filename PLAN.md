<!-- this_file: PLAN.md -->

# PLAN — remaining work

## Offline capture and the companion bundle

- **SingleFile and monolith.** The Python reader has tests for command
  construction and fallback. Run each installed archiver against a served
  fixture with CSS and an image. Stop the fixture server and verify that the
  archived page still has both assets.
- **JS-side critical CSS.** For the optional browser-side path in
  [issue 103](issues/103.md), check that critical styles and rewritten asset
  references still render after disconnecting from the source.
- **Published `vexy-dexyjs` pin.** Compare the pinned published bundle with the
  vendored `build:py` output. Verify the same transform and offline startup
  before switching the Python consumer.

## Browser and exporter checks

- **Warm browser/context reuse.** Compare launch counts and slide output on
  the same fixture before and after sharing the browser between pagination
  and Playwright export. Check cleanup after a failed export.
- **Chassis divergence.** `test_chassis_divergence_guarantee` verifies four
  prepared inputs. Export the same fixture through at least two chassis and
  inspect the decks for preserved content and distinct layouts.

## Optional capture paths from issue 104

- **DOM→SVG.** Export a styled slide as a standalone SVG, render its PDF from
  the same SVG, and compare geometry and background transparency.
- **macOS native PDF.** Compare `WKWebView.createPDF` output with the existing
  screen capture on a fixture with backgrounds and viewport-sized content.
- **Single-page capture.** Check that a tall fixture fits one page at its
  measured scroll width and height without clipping.
- **Transparent PDF backgrounds.** Check whether `qpdf` or `mutool` can remove
  a known opaque background while retaining text and foreground graphics.
