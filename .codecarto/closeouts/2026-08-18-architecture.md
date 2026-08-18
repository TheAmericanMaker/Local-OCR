# Closeout — architecture

## Summary

- Mapped Local-OCR as four first-party modules in a strict acyclic stack:
  `main.py` -> `app.py` -> `ocr_service.py` -> `config.py`, with `config.py`
  as the stable base (it contains no `import` statement) and no cycles.
- Catalogued a GUI-only public surface (no CLI flags, no listener, no library
  API), outbound Ollama HTTP, and the `<stem>_extracted.md` output format.
- Documented the concurrency model: Tk main thread plus at most one
  short-lived daemon worker, mutual exclusion via an `OperationState` enum
  checked on the main thread, and exactly one cross-thread channel — an
  unbounded `queue.Queue` drained every 50 ms. No locks exist.
- Established porting priorities across 16 components.

## Verified by execution, not inferred

The central structural claim — that `ocr_service.py` is Tk-free so the
service layer is headlessly testable — was **tested rather than asserted**.
With no `tkinter` module present on this host, installing only `Pillow`,
`PyMuPDF`, and `ollama` was enough to run the service suite:
**72 passed, 1 skipped, 0 failures in 0.52 s**, on PyMuPDF 1.28.2 and Pillow
11.3.0 — far newer than the declared floor. The boundary genuinely holds.

Introspecting the installed client also confirmed that `ChatResponse.message`
→ `Message.content` and `ListResponse.models[].model` still exist, so the
defensive `getattr` chains in `ocr_service` are currently satisfied rather
than masking drift.

## What execution found that reading missed

The five GUI test files state they "are skipped automatically when no display
is available (CI, headless containers)", and each wraps `LocalOCRApp()` in a
`try/except` that calls `skipTest`. But `import app as app_module` sits at
**module scope**, so without `tkinter` the import fails before any skip logic
runs: `unittest discover` reports **5 errors**, not 5 skips.

A reading-only pass concluded these suites "self-skip without a display."
That is wrong in the case that matters — a headless container — and it means
the ~470 lines of UI contracts they pin have very likely never executed.
Routed as `arch-CF6`.

## Secondary outputs

All five declared outputs are accounted for, per the v0.16.0 orchestrator
duty: `public-surfaces`, `runtime-lifecycle`, and `state-and-storage` were
written; `build-and-deploy` and `config-model` were deliberately skipped
(trivial build surface; flat constant config) with the rationale recorded in
Coverage and limits as `arch-D1` and `arch-D2`.

## Routing

- `arch-CF1`, `arch-CF2`, `arch-CF6` → `defect-scan-mechanical`.
- `arch-CF3` → `protocols`; `arch-CF4` → `contracts`;
  `arch-CF5` → `defect-scan-semantic`.
- `q-shutdown-tempdir` stays open (needs a GUI session this host cannot
  provide). `q-test-invocation` is **re-triaged and narrowed**: a working
  command is now established; only the project's preferred runner is unknown.
