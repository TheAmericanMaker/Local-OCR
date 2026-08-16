# Closeout — contracts

## Summary

- Recovered 13 feature contracts across three surfaces (Desktop GUI, the
  `_extracted.md` storage format, and the outbound Ollama client), each with
  trigger, defaults, observable output, side effects, persisted state, error
  behavior, recovery behavior, and owning module.
- Produced a 40-scenario black-box acceptance list runnable against any
  reimplementation without reading the source.
- Closed `arch-CF4` by mining the ~1,540 lines of tests as executable
  contracts, which is where nearly all the precision came from.

## The decisive finding

The OCR run is an **all-or-nothing transaction, and that is a tested
contract**. `test_late_page_failure_leaves_no_new_output` and
`test_late_page_failure_preserves_existing_output` assert that a failure on
page N discards pages 1..N-1 and leaves any prior output file untouched.

The mechanical scan rated this behavior `high` (D2.1). Both readings are
correct, and the distinction matters for the port: adding partial saves is
**changing a deliberate, tested contract**, not fixing an oversight. Routed
to `porting` as `contracts-CF1` so the Defect Synthesis section decides it
explicitly rather than by default.

## Tests as the specification

The README describes the happy path; the tests pin the edges. Exact strings
and guarantees recovered only from tests include:

- URL normalization: all trailing slashes stripped, reverse-proxy path
  prefixes preserved, `/api` never appended.
- Model listing: deduplicated, whitespace-stripped, empty/None discarded,
  sorted case-insensitively; an empty list is not an error.
- Output format: UTF-8, `\r\n` and bare `\r` both normalized to `\n`, pages
  joined with exactly `"\n\n"`, each page stripped, no header or page markers.
- Ordering: `page_image` precedes the send log; all render events precede all
  OCR events; exactly one terminal event, enqueued after cleanup.
- Cleanup: the temp directory is removed on success and on every failure
  path, asserted in six separate tests.
- PDF rejection: password-protected and zero-page documents fail *before*
  any page is rendered.

## Doc/test conflicts

Four recorded. Two are documentation gaps already routed to the semantic
phase under `mech-CF4` (the overstated atomicity claim; "returned no text"
attributed solely to a non-vision model). Two are tests that lock in behavior
the mechanical scan flagged — the progress formula (D1.1) and the
"unreachable" fallback branch (D1.2) — which corrects the framing of those
findings from bug to deliberate choice.

## Routing

- Closed `arch-CF4`.
- `contracts-CF1`, `contracts-CF2` → `porting` (tested-contract vs. defect
  tradeoffs).
- `contracts-CF3` → `protocols` (formal event catalog and state machine).
- Opened `q-gui-tests-ever-run` (the GUI suites self-skip without a display
  and there is no CI, so it is unknown whether they have ever passed) and
  `q-prompt-provenance`.
