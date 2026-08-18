
---

## From: architecture phase (2026-08-18, v0.16.0)

**Boot:** appearance mode → color theme → construct `LocalOCRApp` → `mainloop()` (`main.py:9-13`). Construction sets `operation_state = IDLE`, creates the queue, builds all widgets eagerly, registers `WM_DELETE_WINDOW`, and schedules the first drain tick. **No config read, no environment probing, no network call at startup** — the app is inert until the user acts (`observed fact`).

**Steady state:** Tk event loop plus one self-rescheduling 50 ms timer (`UI_POLL_INTERVAL_MS`). `drain_ui_events` drains the queue non-blockingly until empty and dispatches each `(kind, payload)`. The reschedule sits in a `finally`, so a raising handler cannot break the pump — deliberate and commented (`app.py:257-261`). Unknown event kinds surface as `[Warn]` rather than being dropped.

**Job lifecycle:** inputs validated and snapshotted into a frozen `OCRRequest` **on the main thread** before any thread starts — this is what keeps workers from reading mutable widget state. Then a daemon thread runs three logged stages: `[1/3]` prepare (rasterize a PDF, or pass a single image through), `[2/3]` recognize page-by-page, `[3/3]` save. **Exactly one terminal event** (`ocr_success` / `ocr_error`) is enqueued by the wrapper *after* `process_ocr` returns or raises, so temp-dir cleanup always precedes it (`ocr_service.py:302-310`, `app.py:658-670`).

**Concurrency:** Tk main thread + at most one short-lived daemon worker. Mutual exclusion via the `OperationState` enum, **not** a lock — sound only because all guards run on the main thread. One cross-thread object: an **unbounded** `queue.Queue`. No `Lock`, `Event`, `Semaphore`, or `Condition` exists anywhere (`observed fact`).

**Shutdown:** idle → immediate `destroy()`; busy → yes/no confirmation. Workers are daemon threads, so an in-flight job is **abandoned rather than joined**, and `process_ocr`'s cleanup `finally` may not run at interpreter exit (`q-shutdown-tempdir`, routed as `arch-CF1`). **No cancellation mechanism exists at all** — no stop button, no cancel flag, no check inside either loop.

**Background work:** only the 50 ms drain tick and a 100 ms one-shot `after()` throttling stream flushes. No scheduler, no retry loop.

## From: contracts phase (2026-08-18, v0.16.0)

**Operation lifecycle as a user contract.** Exactly one operation runs at a time (`OperationState`); every entrypoint no-ops unless IDLE. A run proceeds: validate-and-snapshot on the main thread → disable controls, clear panels, force the Log tab → `[1/3]` prepare → `[2/3]` recognize per page → `[3/3]` save → terminal event → restore idle, switch to **Result** (success) or **Log** (failure).

**What the contract does *not* provide** (`observed fact`): no cancellation, no retry, no resume, no partial output, and no memory of the previous outcome. Failure returns straight to IDLE, which is why the app can never offer "retry."

**Shutdown is not an abort.** The confirmation prompt ("An operation is still running. Close anyway?") implies the operation is discarded cleanly. It is not: the daemon worker is neither joined nor signalled, and a probe this run proved the cleanup `finally` **does not run** at interpreter exit, so the render directory survives (D2.3). Acceptance row 41 encodes the rule the implementation currently fails.

**Ordering guarantees the lifecycle must preserve** (all **verified** by the passing service suite): `page_image` precedes the per-page send log; all render progress precedes all OCR progress; exactly one terminal event per run, enqueued after cleanup; page numbers ascend 1..N without gaps.

## From: protocols phase (2026-08-18, v0.16.0)

**`OperationState` formalized** as SM1: three states, 22 transitions, all on the Tk main thread. Three properties define it more than the transitions do (`observed fact`):

1. **No cancellation transition exists.** Once in PROCESSING_OCR the only exits are a terminal event or window destruction.
2. **No error state.** Failure returns directly to IDLE with no memory of the outcome — which is *why* the app can never offer "retry."
3. **Exactly four events can leave a busy state** (`models_loaded`, `refresh_error`, `ocr_success`, `ocr_error`). Lose one and the machine is stuck with no recovery path (D2.6).

**Event classification governing what a port may drop** (`observed fact`): observational events (`log`, `progress`, `page_image`, `stream_chunk`, `page_text`) drive UI only — the saved Markdown is built from the recognition function's *return value*, so losing all of them still yields a byte-identical file. Only the four state-bearing events need guaranteed delivery. Three synchronous barriers sit inside the worker, invisible to the queue: render-all before recognize-any, recognize-all before save, cleanup before the terminal event.

**Three subordinate machines** — progress-bar mode (SM2), result-panel streaming (SM3), review navigation (SM4) — are pinned **only by the GUI suites, which do not run here**, so their transitions are `asserted-not-verified` (conventions C02/C04). This is the weakest-evidenced part of the lifecycle description.
