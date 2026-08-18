# Closeout — defect-scan-mechanical

## Summary

- Ran passes 1 (logic), 2 (error handling), and 6 (config/environment) over
  all 1,178 lines of first-party source. **23 findings: 0 critical, 3 high,
  8 medium, 12 low.**
- No critical findings. Nothing in the mechanical scope produces silently
  wrong OCR output or corrupts user data; the three high findings all concern
  *losing completed work*.
- Closed `arch-CF1` (D2.3), `arch-CF2` (D2.5), and `arch-CF6` (D1.6), and
  closed the open question `q-shutdown-tempdir` on new evidence.

## The dominant theme

All three high findings reduce to one design decision: the OCR pipeline is a
single all-or-nothing transaction over N independent network calls.

- **D2.1** — one failed page raises past `save_markdown_atomic`, discarding
  every page already recognized. No retry, no partial save.
- **D2.2** — a legitimately blank page is treated as fatal and takes the same
  path, so an ordinary document with an empty verso can abort.
- **D2.3** — quitting mid-job neither joins nor cancels the worker.

## Two things execution settled that reading could not

Convention **C01** ("execute the headless layer instead of inferring it")
earned its place twice in this phase:

1. **D2.3 is now `observed fact`.** A 15-line probe reproducing
   `process_ocr`'s shape — daemon thread doing `mkdtemp` then `finally:
   rmtree`, main thread returning after 0.4 s — shows the `finally` **does
   not run** and the temp directory **survives** interpreter exit on Python
   3.11. The leak is real, not merely plausible. `q-shutdown-tempdir` closed.
2. **D1.6 exists at all.** All five GUI suites promise they "are skipped
   automatically when no display is available (CI, headless containers)", but
   the guard is in `setUp` while `import app` is at module scope. Without
   `tkinter` they **error at collection**: `unittest discover` reports 5
   errors while the service suite passes 72. A guard placed after the thing
   it guards is not a guard — and those ~470 lines of UI contracts have very
   likely never executed. This is exactly what convention **C02** was
   promoted to prevent us from assuming.

## A finding tempered rather than confirmed

**D6.2** (uncapped `PyMuPDF` and `ollama`) is real but **currently
unrealized**: the suite passes on PyMuPDF 1.28.2 and Pillow 11.3.0, well
above the declared floor, and the installed client still exposes the shapes
the `getattr` chains read. That is a reason to pin deliberately and commit a
lockfile — not evidence the risk is theoretical. Reported as measured.

## What is done well

Worth recording so a port does not regress it: the `finally`-reschedule in
`drain_ui_events`, the non-fatal thumbnail failure path, the single-terminal-
event sequencing after cleanup, the frozen `OCRRequest` snapshot taken on the
main thread, and the centralized, rationale-commented constants in
`config.py` are all deliberate and correct.

## Routing

- Closed `arch-CF1`, `arch-CF2`, `arch-CF6`; closed `q-shutdown-tempdir`.
- `mech-CF1`–`mech-CF5` → `defect-scan-semantic`: TOCTOU windows, Ollama
  transport/trust, the lock-free `OperationState` invariant, README-vs-
  implementation drift, and provider-contract drift with a misleading
  diagnostic.
- Opened `q-ctk-image-clear` (needs a GUI this host cannot provide).
- Proposed one convention: *name the subsystem the error actually came from*
  (from D2.10).
