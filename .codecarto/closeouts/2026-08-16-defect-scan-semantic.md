# Closeout — defect-scan-semantic

## Summary

- Ran passes 3 (concurrency and resources), 4 (security and trust), and 5
  (API contract violations) with the contracts and protocols outputs in hand.
  18 findings: 0 critical, 2 high, 7 medium, 9 low.
- Full audit total across both scans: **39 findings — 0 critical, 5 high,
  14 medium, 20 low.**

## The two high findings

- **S5.1** — a rename of `message.content` in a future `ollama` release
  makes every delta read as empty, every page fail, and the error report
  "Ollama returned no text (model X)" — which the README maps to "no vision
  support." The user is told to change models; no model will help. Reachable
  today because `ollama>=0.4.0` is uncapped with no lockfile.
- **S3.1** — the 120 s timeout guards gaps *between* chunks, so a slow but
  live stream never trips it. There is no overall job timeout and SM1 has no
  cancellation transition, so the app can enter a state it cannot leave. The
  only escape is quitting, which leaks the temp directory (D2.3) and may
  still write the output file (S3.3).

## The refutation

`mech-CF3` questioned the lock-free `OperationState` mutual exclusion and
whether `self.closing` races. Enumerating every access site and its thread
shows **both are main-thread-only** — workers touch nothing but
`event_queue.put`. The design is correct as written and a lock would add
nothing.

The invariant worth carrying: *all mutable UI state is confined to a single
thread; the queue is the only shared object.* A port must re-establish that
confinement explicitly rather than read the absence of locks as the absence
of a concurrency contract.

## Structural themes

- **Absent bounds.** No job timeout, no output size cap, no input size cap,
  no queue bound, no per-drain budget, no temp-space check. The system is
  correct on the happy path and unbounded in every other direction.
- **Errors that misdirect.** S5.1, S5.2, and S5.3 are one weakness in three
  places: defensive reads and generous prose combine so the app asserts
  causes it has not established.

## Security posture

The real attack surface is **untrusted binary parsing** (`pymupdf.open`,
`Image.open`) with no size limit and no upper version bound — not the
network. Verified clean: no secrets anywhere, and no command-injection path
(every OS call is an argv vector, no `shell=True`).

## Routing

All six routed items resolved, none re-routed: `arch-CF5` (S3.2, S3.5),
`mech-CF1` (split into S4.2 medium / S4.5 low), `mech-CF2` (S4.3),
`mech-CF3` (**refuted**), `mech-CF4` (S5.2, S5.3), `protocols-CF2`
(**escalated to high** as S5.1). No new carry-forwards and no new open
questions — the porting phase already carries the tradeoffs that matter
(`contracts-CF1`, `contracts-CF2`, `protocols-CF1`).
