# Closeout — contracts

## Summary

- Recovered 13 feature contracts across three surfaces (Desktop GUI, the
  `_extracted.md` storage format, the outbound Ollama client), each with all
  seven contract fields plus an owning module.
- Produced a **42-scenario** black-box acceptance list runnable against any
  reimplementation, each row carrying its current verification status.
- Closed `arch-CF4` by mining the ~1,540 lines of tests, which is where nearly
  all the precision came from.

## The framing that changed this run: two-tier evidence

Convention **C02** ("a test's existence is not evidence that it runs") forced
a split the previous pass had averaged away:

- **`tests/test_ocr_service.py`** (1,069 lines) — **executed, 72 passed, 1
  skipped.** Contracts it pins are `observed fact`.
- **The five GUI suites** (~470 lines) — **error at collection** without
  `tkinter` (D1.6). Every contract resting only on them is
  **`asserted-not-verified`**: not wrong, but undemonstrated.

14 of the 42 acceptance rows fall in the weaker tier, all UI-side. The GUI is
simultaneously the only interactive surface and the least verified half of the
specification — which is worth knowing before trusting it as a parity harness.

## The decisive contract, and it is verified

The OCR run is an **all-or-nothing transaction**, and both tests asserting it
are service-suite tests that **pass**: a failure on page N discards pages
1..N−1 and leaves any prior output untouched. The mechanical scan rated this
`high` (D2.1). Both readings are correct, and the distinction matters: adding
partial saves is **changing a passing, deliberate contract**, not fixing an
oversight. Routed to `porting` as `contracts-CF1`.

## Two scenarios the implementation fails

Rows 41 and 42 are new, and both came from execution rather than reading:

- **41** — quitting mid-run must not leak the render directory. Proven
  failing by the daemon-thread probe (D2.3).
- **42** — a headless test run must report pass/skip, never collection
  errors. Measured failing (D1.6).

Routed as `contracts-CF4` so they reach the spec as normative rules.

## Doc/test conflicts

Five recorded. Two are documentation gaps already routed under `mech-CF4`.
Two are tests locking in behavior the scan flagged (D1.1, D1.2) — both
GUI-only, so they specify without demonstrating. The fifth is new: **the GUI
suites' own skip docstring is false**, which is what motivated the evidence
tiering.

## Routing

- Closed `arch-CF4`.
- `contracts-CF1`, `contracts-CF2`, `contracts-CF4` → `porting`.
- `contracts-CF3` → `protocols`.
- `q-gui-tests-ever-run` **reframed** on measurement (they cannot have passed
  headless; whether they pass with a display is still unknown);
  `q-prompt-provenance` opened.
- Proposed one convention: *record verification status alongside every
  acceptance scenario*.
