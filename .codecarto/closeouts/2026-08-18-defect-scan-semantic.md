# Closeout — defect-scan-semantic

## Summary

- Ran passes 3 (concurrency and resources), 4 (security and trust), and 5
  (API contract violations) with contracts and protocols in hand. **18
  findings: 0 critical, 1 high, 8 medium, 9 low.**
- Full audit across both scans: **41 findings — 0 critical, 4 high, 16
  medium, 21 low.**
- All seven routed items resolved; none re-routed.

## The contradiction sweep changed a finding

The v0.16.0 loop asks each phase to compare measured facts against earlier
summarized claims. That duty paid off here in an uncomfortable direction.

The previous pass rated provider-schema drift **high**, partly because it was
"reachable today through ordinary dependency resolution." The protocols phase
then *measured* the installed client and found `ChatResponse.message.content`
and `ListResponse.models[].model` still present. The premise was wrong: the
newest resolvable client satisfies the chains, so the trigger is a **future**
rename.

S5.1 is therefore **medium**, not high. Impact remains total and the
diagnostic remains actively misleading — the README maps that exact error
string to "no vision support" — which keeps it above `low`. But a headline
severity that rests on a contradicted premise is worth less than an accurate
one, and the mild form of the same weakness (H14/D2.10, a local missing-file
error reported as an Ollama failure) **is** already live.

## The refutation

`mech-CF3` questioned the lock-free `OperationState` exclusion and whether
`self.closing` races. Enumerating every access site and its thread shows
**both are main-thread-only** — workers touch nothing but `event_queue.put`.
The design is correct as written; a lock would add nothing.

The invariant worth carrying: *all mutable UI state is confined to a single
thread; the queue is the only shared object.* A port must re-establish that
confinement explicitly rather than read the absence of locks as the absence
of a concurrency contract.

## Two findings sharpened by upstream work

Because the protocols phase resolved the wire encoding, two security findings
got more precise rather than merely more confident:

- **S4.3** — what crosses the wire is the **complete pixel content of every
  page**, base64'd, over unauthenticated plaintext HTTP. Previously it was
  unclear whether bytes or a path were sent.
- **S4.5** — the TOCTOU window extends to **base64 encode time, per page**,
  because the client re-reads the file at request time. For a PDF that spans
  the whole recognition loop, not just the initial open.

## Structural themes

- **Absent bounds.** No job timeout, no output cap, no input cap, no queue
  bound, no per-drain budget, no free-space check.
- **Errors that misdirect.** S5.1, S5.2, S5.3, and D2.10 are one weakness in
  four places — exactly what convention C03 was promoted to prevent.

## Security posture

The real attack surface is **untrusted binary parsing** (`pymupdf.open`,
`Image.open`) with no size limit and PyMuPDF uncapped — not the network.
Verified clean: no secrets anywhere, and no command-injection path, with all
six OS-integration branches covered by passing tests.

## Routing

Closed all seven: `arch-CF5` (S3.2, S3.5), `mech-CF1` (split S4.2/S4.5),
`mech-CF2` (S4.3, sharpened), `mech-CF3` (**refuted**), `mech-CF4` (S5.2 —
now four causes, not three — and S5.3), `mech-CF5` and `protocols-CF2` (S5.1,
**downgraded**). No new carry-forwards and no new open questions: the porting
phase already carries the tradeoffs that matter (`contracts-CF1`,
`contracts-CF2`, `contracts-CF4`, `protocols-CF1`).
