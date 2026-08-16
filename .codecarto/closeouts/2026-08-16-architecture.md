# Closeout — architecture

## Summary

- Mapped Local-OCR as four first-party modules in a strict acyclic stack:
  `main.py` -> `app.py` -> `ocr_service.py` -> `config.py`, with `config.py`
  as the stable base (it imports nothing) and no dependency cycles.
- Identified the Tk boundary as the load-bearing architectural line:
  `ocr_service.py` is deliberately Tk-free so the service layer stays
  headlessly testable, which is what makes the 1,069-line service test file
  possible. The rule is convention-only — nothing enforces it mechanically.
- Documented the concurrency model: Tk main thread plus at most one
  short-lived daemon worker, mutual exclusion via an `OperationState` enum
  checked on the main thread, and exactly one cross-thread channel (an
  unbounded `queue.Queue` drained every 50 ms by `after()`). No locks exist.
- Catalogued public surfaces: a GUI-only interaction surface (no CLI flags,
  no listener, no library API), outbound Ollama HTTP (`list` and streaming
  `chat`, one request per page), and the `<stem>_extracted.md` output format.
- Established porting priorities across 16 components, with the per-page
  streaming recognition loop, PDF rasterization, the two prompt strings, the
  event-queue contract, and the atomic save marked `core`.

## Notable structural findings

- No durable configuration of any kind: no config file, no environment
  variables, no saved preferences. Every launch resets URL, model, and DPI.
- The app performs no network or disk I/O at startup; it is inert until the
  user acts.
- Build and packaging are effectively absent — `requirements.txt` and
  `python main.py`, with no CI, no wheel, and no container. A macOS packaging
  script exists but is gitignored and therefore outside this analysis.

## Routing

- `arch-CF1`, `arch-CF2` -> `defect-scan-mechanical` (shutdown/cleanup hazard;
  unbounded queue and unevicted page bytes).
- `arch-CF3` -> `protocols` (event payload schemas, ordering, state machine).
- `arch-CF4` -> `contracts` (behavioral contracts and the test bodies as
  evidence).
- `arch-CF5` -> `defect-scan-semantic` (main-thread image decode; eager
  full-document rasterization).
- Open questions `q-shutdown-tempdir` and `q-test-invocation` remain: both
  need a runtime probe or a maintainer ruling that no phase in this pipeline
  performs.
