# Closeout — defect-scan-mechanical

## Summary

- Ran passes 1 (logic), 2 (error handling), and 6 (config/environment) over
  all 1,178 lines of first-party source. 21 findings: 0 critical, 3 high,
  7 medium, 11 low.
- No critical findings. Nothing in the mechanical scope produces silently
  wrong OCR output or corrupts user data in normal operation; the three high
  findings all concern *losing completed work*.

## The dominant theme

All three high findings reduce to one design decision: the OCR pipeline is a
single all-or-nothing transaction over N independent network calls.

- **D2.1** — one failed page raises straight past `save_markdown_atomic`,
  discarding every page already recognized. No retry, no partial save.
- **D2.2** — a legitimately blank page is treated as a fatal error and takes
  the same path, so ordinary documents with an empty verso can abort.
- **D2.3** (closes `arch-CF1`) — quitting mid-job neither joins nor cancels
  the daemon worker; the cleanup `finally` is not reached at interpreter
  exit, leaking a directory of rendered pages. No cancellation exists at all.

A port should model the job as N independently retryable units with partial
results preserved, not as one transaction.

## Other notable findings

- **D6.3** — every page is rasterized before any recognition starts, with no
  free-space check; gigabytes of temp on a large PDF at 300 DPI. This is also
  what makes D2.3's leak expensive.
- **D2.5** (closes `arch-CF2`) — the event queue has no `maxsize` and raw
  page PNG bytes in `review_pages` are never evicted; both grow with document
  length. The 5-entry LRU bounds only the *decoded* images.
- **D2.6** — `handle_event` is not individually guarded, so a raising handler
  drops the events already dequeued; losing a terminal event strands the UI
  in its busy state permanently.
- **D6.1** — no configuration surface of any kind, and no persistence: URL,
  model, and DPI are retyped on every launch.
- **D6.2** — `PyMuPDF` and `ollama` are uncapped while `customtkinter` and
  `Pillow` are capped, and those two are exactly the APIs the code already
  guards with defensive `getattr` chains.

## What is done well

Worth recording so the port does not regress it: the `finally`-reschedule in
`drain_ui_events`, the non-fatal thumbnail failure path, the single-terminal-
event sequencing after cleanup, the frozen `OCRRequest` snapshot taken on the
main thread, and the centralized, rationale-commented constants in
`config.py` are all deliberate and correct.

## Routing

- Closed `arch-CF1` (D2.3) and `arch-CF2` (D2.5).
- `mech-CF1`–`mech-CF4` → `defect-scan-semantic`: TOCTOU windows, Ollama
  transport/trust, the lock-free `OperationState` invariant, and README-vs-
  implementation drift.
- `q-ctk-image-clear` opened: customtkinter's `configure(image=None)`
  clearing behavior needs a runtime check.
