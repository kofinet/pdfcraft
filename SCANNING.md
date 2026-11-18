# PdfCraft with scanning (fork)

This fork of [storytold/pdfcraft](https://github.com/storytold/pdfcraft) adds **scanning straight into PdfCraft** on
Windows (WIA). It is built on current `main` and brings in the work of
[storytold/pdfcraft#1724](https://github.com/storytold/pdfcraft/pull/1724) by @b1tninja, which upstream has not
merged yet. Branch: `scan-wia`.

## What is added

- **From scanner…** in the app (Create a PDF, Organize pages, Scan & OCR). You pick the scanner, bed or feeder,
  resolution, colour and page size, and see a preview where you can choose a region. Then **Create PDF** makes a new
  document, or **Insert scanned pages** adds the pages to the open one. It also handles manual duplex
  (scan reverse sides).
- **`pdfcraft-cli scan`**:
  - `devices` / `check` list what Windows offers.
  - `scan --out DIR --yes` scans as a resumable job with a journal: `scan jobs`, `scan job DIR`,
    `scan --continue DIR`.
- **Automation tools** `scan_devices`, `scan_acquire` and `scan_jobs` (CLI `run` and MCP).
- **Create PDF sizes each page from the image's own resolution** (BMP, PNG, JPEG, TIFF). A 300 dpi scan therefore
  becomes an A4 page, not a 35-inch one, and a black-and-white BMP stays 1-bit.

## Fixes on top of #1724

Found while testing on a Brother DCP-J105 (flatbed only) over the network. They are offered to the pull request as
[b1tninja/pdfcraft#2](https://github.com/b1tninja/pdfcraft/pull/2), on its own branch:

1. **Flatbed-only scanners have no feeder.**
   - Before, the scanner's only transfer item was lent to the feeder. The Feeder button was enabled, and
     `--source feeder` scanned the glass and stopped with "the feeder is empty".
   - Now the feeder is absent when the scanner's capabilities say so: the button is disabled, and the CLI says
     "this scanner has no feeder; scan from the glass".
2. **`--page-size a4` gives A4's width.** Some drivers accept the page size but keep their whole-bed extent: 8.50 in
   instead of 8.27 in. The page size now narrows the scan to its width. It never widens it.
   - A WSD port is left alone. The inbox WSD driver already returns A4's width from the whole extent, and it narrows a
     written 8.27 in to 8.11 in.
3. **A page clipped by the driver is reported.** The inbox Windows WSD driver returns 290.7 mm of an A4 page's
   297 mm, whatever is asked; Windows' own WIA automation gets the same. This is not fixable in PdfCraft. The
   backend now logs and traces it (`page_length_short`). Prefer the maker's driver where Windows lists both.
4. **The dialog uses the window's height**, so a whole page's preview shows without scrolling (it was capped at
   520 px).

## Building

Rust 1.95 or later, on Windows with the MSVC C++ build tools:

```
cargo build --release -p pdfcraft -p pdfcraft-cli
```

The scanning crates (`craft-scan`, `craft-job`, `craft-devices`) and the `craft-ui` scanning component come from the
companion fork [kofinet/craft-libs](https://github.com/kofinet/craft-libs), branch `pdfcraft-scan`.
That branch holds storytold/craft-libs#144 and #112 applied to the craft-libs revision PdfCraft `main` uses.

## Status

- Scanning is **Windows only (WIA)**. Other platforms build and run, and say scanning is not available.
- **Tested by hand on one scanner:** Brother DCP-J105 over the network, from the glass, through both the Brother
  driver and WSD. On this branch:
  - The Feeder button is disabled, and `--source feeder` is refused before anything moves.
  - A4 at 300 dpi is 2480 px wide through both drivers. It is 3507 rows long through the Brother driver. Through WSD it
    is 3433 rows, and the trace reports `page_length_short`, 250 mils.
  - The whole preview fits in a 1009 px window.
- **Automated checks run locally:**
  - The tests of the touched crates pass: pdfcraft-wia, -create, -engine, -platform, -automation, -cli and -ui-egui.
  - `cargo clippy -D warnings` and `cargo fmt --check` pass.
  - The new tests fail without their fixes.
  - pdfcraft-ui-egui's `it` suite passes with `--test-threads=1`. Run in parallel on this machine, it crashes the process
    (access violation) in a different test each time.
  - One test needs the German catalog (craft-i18n), which this build does not have.
- **Not tested on hardware here:** feeders, duplex and jams.
- The WIA backend (`crates/scan-wia`) is the only crate with `unsafe` code (COM). See `docs/unsafe-audit.md`.

The licence is the same as PdfCraft's: MIT OR Apache-2.0.
