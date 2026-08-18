---
selection: language-agnostic
selection_basis: >-
  The Strategic Alignment Hook was not answered interactively. The originating
  request was to analyze Local-OCR; no target stack, project identity, module
  names, or scope cuts were committed. Per GUIDE.md the spec therefore defaults
  to language-agnostic and uses templates/reimplementation-spec.md. The choices
  the hook would have elicited are recorded as Known Unknowns (`q-target-stack`,
  `q-scope-tier-commitment`) rather than guessed.
build_order: kernel-first with fake-driven acceptance tests
platform_assumption: >-
  None committed. The MVP relies on only two platform primitives, both named in
  §Protocols and Persisted State: same-filesystem atomic rename, and fsync.
framework: CodeCartographer v0.16.0
date: 2026-08-18
---

# Reimplementation Spec — Local-OCR

Primary input: `findings/porting/reverse-engineering-bundle.md`.
**Closes `porting-CF1`, `porting-CF2`, `porting-CF3`.**

### Orchestrator duties discharged

**Open-question re-triage** (5 inherited). All five re-tested and **confirmed**; each needs a resource this environment lacks (a `tkinter` build, a display, a GPU-backed model, or a maintainer). None is closable by reading, and convention C01 reaches none of them. Two new Known Unknowns are added for the strategic choices the hook would have elicited.

**Contradiction sweep.** No contradictions. The bundle's severities already reflect the semantic phase's evidence-driven downgrade of provider drift.

**Targeted deep reads** (per the skill, recorded rather than assumed). Three, each triggered by a gap the bundle explicitly named: (1) `behavioral-contracts.md` §Black-Box Acceptance List, to fold the 42 scenarios and their verification-status column into §Acceptance Scenarios — the bundle omitted them by design (`porting-CF3`); (2) `protocols-and-state.md` §B1 payload schemas and SM1, to pin the event contract concretely — same trigger; (3) `mechanical-defects.md` D1.6 and `semantic-defects.md` §Verified Safe, for the original rationale behind two dispositions that become MUST rules here. No other upstream report was reloaded.

## System Summary

Build a **single-user desktop application that converts one document — a PDF or a single image — into one Markdown file**, by sending each page as an image to a vision-language model on a user-named Ollama server and concatenating the results. Nothing leaves the machine except traffic to that URL. The app never installs, pulls, or starts a model.

The original is 1,178 lines across four modules, with a disciplined separation: a conversion engine holding all rules and no UI imports, and a presentation shell owning every widget and all mutable state, connected by a single queue. **Preserve that separation** — it is what makes the engine testable without a display, and the reimplementation should enforce it mechanically rather than by convention.

The original has **no critical defects** and never silently produces wrong output. Its defining weakness is that **the run is one all-or-nothing transaction over N independent network calls**: one failed page, one blank page, or one mid-run quit discards every page already recognized. The second weakness is that **nothing is bounded** — no job timeout, output cap, input cap, queue bound, per-drain budget, or free-space check. This spec's central purpose is to build the same product without either property.

One caveat inherited from the audit and carried into §Acceptance Scenarios: the original's specification is **two-tier**. Its 1,069-line service suite passes; its ~470 lines of GUI tests error at collection and have very likely never run. Fourteen of the 42 inherited scenarios are therefore *asserted, not verified*, and two describe behavior the original **fails**.

## Conceptual Module Model

Concept names. Do not mirror the source's file layout.

### M1 — Recognition Orchestrator *(kernel)*

| Field | Value |
|---|---|
| **Responsibility** | Drive a document to a completed extraction: obtain pages, recognize each, assemble the result, and decide what happens when a page fails. |
| **Public inputs** | A `ConversionRequest` — immutable snapshot of `{input path, output path, server URL, model tag, DPI}` — plus a cancellation token. |
| **Public outputs** | A `ConversionOutcome`: `{status: completed \| partial \| failed, page results (ordered), artifact path, failure detail?}`. |
| **Owned state** | Per-page results and their statuses; retry counters; the current page index. Nothing UI-related. |
| **Invariants** | Pages are recognized in document order and results returned in that order. **The assembled artifact is derived from returned page results, never from the event stream.** A page's outcome is one of `recognized`, `empty`, `failed-after-retries`. |
| **Collaborators** | M2 (page source), M3 (recognizer), M4 (artifact writer), M6 (event sink), M7 (clock/cancellation). |

### M2 — Page Source *(port, with adapters)*

| Field | Value |
|---|---|
| **Responsibility** | Yield the document's pages as images, one at a time, in document order. |
| **Public inputs** | Input path, DPI (PDF only). |
| **Public outputs** | An **iterator** of `{page number, total, image bytes-or-handle}`; a total page count known before the first page. |
| **Owned state** | Any transient render storage, and its lifetime. |
| **Invariants** | **MUST be lazy** — at most a small constant number of rendered pages exist at once. Reject encrypted and zero-page documents **before** rendering any page. A released page's storage MUST be reclaimed before the next is produced. |
| **Collaborators** | M1. Adapters: PDF rasterizer; pass-through image reader. |

### M3 — Recognizer *(port, with adapter)*

| Field | Value |
|---|---|
| **Responsibility** | Turn one page image into text via one independent model request, streaming deltas as they arrive. |
| **Public inputs** | Page image, model tag, server URL, the two prompt strings, a per-page deadline, a cancellation token. |
| **Public outputs** | A stream of text deltas, then a final assembled page text; or a typed error. |
| **Owned state** | Client/connection lifetime — **MUST be explicitly closed**. |
| **Invariants** | One request per page; **no cross-page context**. Errors MUST be typed by cause: `encode-failure` (local), `transport-failure`, `protocol-mismatch` (response shape unrecognized), `empty-result`. **A local encode failure MUST NOT be reported as a remote failure.** |
| **Collaborators** | M1, M6. Adapter: Ollama HTTP client. |

### M4 — Artifact Writer *(port, with adapter)*

| Field | Value |
|---|---|
| **Responsibility** | Publish the extraction durably and atomically, and never damage an existing artifact. |
| **Public inputs** | Destination path, assembled content, a `canonical \| partial` designation. |
| **Public outputs** | The written path, or a typed error. |
| **Owned state** | Staging-file lifetime. |
| **Invariants** | Stage in the destination's own directory → flush → **fsync** → atomic same-filesystem rename. On any failure the pre-existing artifact is byte-unchanged and no staging file remains. Content is UTF-8, LF-only, pages joined by one blank line, each page trimmed. |
| **Collaborators** | M1. |

### M5 — Operation State Machine *(kernel)*

| Field | Value |
|---|---|
| **Responsibility** | Enforce that at most one operation runs, expose its phase, and remember the last outcome. |
| **Public inputs** | Start/cancel requests; state-bearing events from M6. |
| **Public outputs** | Current state and last outcome, for the shell to render. |
| **Owned state** | `IDLE(outcome?) \| LISTING_MODELS \| CONVERTING \| CANCELLING`. |
| **Invariants** | All transitions occur on one designated thread or task context. **Every busy state has a path to IDLE that does not require a network response** — cancellation. **IDLE carries the last outcome**, so retry can be offered. |
| **Collaborators** | M6, M8. |

### M6 — Event Sink *(port, with adapters)*

| Field | Value |
|---|---|
| **Responsibility** | Carry progress and results from the worker context to any observer. |
| **Public inputs** | Typed events (§Protocols). |
| **Public outputs** | Delivery to zero or more observers. |
| **Owned state** | A **bounded** channel. |
| **Invariants** | **Bounded — a saturated observer MUST apply backpressure to the producer.** Observational events MAY be dropped or coalesced; **state-bearing events MUST NOT be.** A raising observer MUST NOT prevent delivery of subsequent events. |
| **Collaborators** | All. |

### M7 — Clock and Cancellation *(port)*

| Field | Value |
|---|---|
| **Responsibility** | Supply deadlines and a cooperative cancellation signal. |
| **Public inputs** | Deadline configuration. |
| **Public outputs** | Deadline expiry; a cancellation token observable at page and chunk granularity. |
| **Invariants** | Two independent limits: an **inter-chunk idle timeout** and a **per-page wall-clock ceiling**. Injectable, so tests need no real waiting. |
| **Collaborators** | M1, M3. |

### M8 — Presentation Shell *(delivery surface)*

| Field | Value |
|---|---|
| **Responsibility** | Collect inputs, render progress and results, and offer post-run actions. |
| **Public inputs** | User interaction; events from M6; state from M5. |
| **Public outputs** | A `ConversionRequest`; cancel requests. |
| **Owned state** | All widget and view state, including the live text buffer and review model. |
| **Invariants** | **MUST NOT be imported by any kernel or port module** — enforced mechanically. Control enablement derives from **one** state→controls mapping. All view state confined to one thread/context. |
| **Collaborators** | M5, M6. |

### M9 — Settings Registry *(core semantics)*

| Field | Value |
|---|---|
| **Responsibility** | Supply defaults and tunables, including the two prompt strings, and persist the user's last-used choices. |
| **Public inputs** | A user config file; built-in defaults. |
| **Public outputs** | Resolved settings; a persist operation. |
| **Invariants** | Depends on nothing. **The prompt pair is versioned configuration, not a literal.** Every numeric setting is range-checked at load; an absent value is distinguishable from an explicit zero. |
| **Collaborators** | All. |

## Layer Split

| Module | Layer | Notes |
|---|---|---|
| M1 Recognition Orchestrator | **core semantics** | The behavior that must survive unchanged: page ordering, per-page outcomes, assembly. |
| M5 Operation State Machine | **core semantics** | Exclusion, cancellation, last-outcome memory. |
| M9 Settings Registry | **core semantics** | Prompts and defaults are behavior, not configuration trivia. |
| M2 Page Source | **adapter** (behind a port) | PDF rasterizer and image pass-through. |
| M3 Recognizer | **adapter** (behind a port) | Ollama HTTP client; base64 encoding lives here. |
| M4 Artifact Writer | **adapter** (behind a port) | Filesystem; atomic-rename semantics. |
| M6 Event Sink | **adapter** (behind a port) | Bounded channel; UI observer. |
| M7 Clock and Cancellation | **adapter** (behind a port) | Injectable for deterministic tests. |
| M8 Presentation Shell | **delivery surface** | The only surface today. A CLI becomes trivial once M1–M7 exist. |

**The load-bearing rule:** M1, M5, and M9 MUST be buildable and testable with no adapter from M2, M3, M4, or M8 present. If they are not, the port boundary is in the wrong place.

## Required Behaviors

Normative rules. Each **MUST** traces to a defect disposition or a verified contract; the acceptance scenario that proves it is in parentheses. This section, with §Acceptance Scenarios, closes `porting-CF1` and `porting-CF2`.

### R1 — Partial success is a first-class outcome *(D2.1, D2.2, Decision 1)*
1. **R1.1** Each page MUST be independently retryable: on a retryable failure, retry with bounded attempts and backoff before recording `failed-after-retries`. *(A1)*
2. **R1.2** A page whose recognized text is empty MUST be recorded as `empty` and contribute an empty section — **it MUST NOT fail the document.** *(A2)*
3. **R1.3** When at least one page is recognized and at least one has `failed-after-retries`, the run MUST complete with status `partial` and MUST persist the recognized pages, with each failed page explicitly marked in the artifact. *(A1)*
4. **R1.4** **PRESERVED FROM THE ORIGINAL, as a hard rule:** the canonical artifact MUST never be left corrupt, partial, or truncated, and MUST never be destroyed except by a successful complete write. A `partial` run MUST write to a **distinct, clearly-named** artifact and leave any existing canonical artifact byte-unchanged. *(A3, A4)*
5. **R1.5** A `failed` run (no page recognized) MUST write nothing. *(A4)*

> **Contract change, stated explicitly.** R1.1–R1.3 deliberately break the original's tested all-or-nothing behavior. `test_late_page_failure_preserves_existing_output` remains satisfied by R1.4 and needs no change. `test_late_page_failure_leaves_no_new_output` MUST be reworded to assert *no canonical artifact was created*. These are the only two test changes Decision 1 requires.

### R2 — Everything is bounded *(S3.1, D2.5, S3.2, S4.4, D6.3, S3.5)*
1. **R2.1** Two independent limits MUST apply per page: an inter-chunk idle timeout **and** a wall-clock ceiling. Exceeding either yields a typed timeout for that page, handled under R1. *(A5)*
2. **R2.2** Recognized output MUST be capped per page and per document; exceeding either fails that page under R1 with a distinct message. *(A6)*
3. **R2.3** Input MUST be bounded before parsing: a maximum page count and a maximum byte size, both rejected with a clear message. *(A7)*
4. **R2.4** The event channel MUST be bounded, and a saturated observer MUST slow the producer. *(A8)*
5. **R2.5** Page rendering MUST be lazy — at most a small constant number of rendered pages on disk at once — and each page's storage reclaimed once recognized. *(A9)*
6. **R2.6** An observer MUST NOT block its host context unboundedly: per-cycle work MUST be capped, with expensive decoding performed off the UI context. *(A10)*

### R3 — Cancellation is real *(D2.3, S3.1, S3.3, Decision 3)*
1. **R3.1** A cancellation token MUST be observable at page **and** chunk granularity; the orchestrator MUST check it between units. *(A11)*
2. **R3.2** The shell MUST expose a cancel affordance whenever an operation is running. *(A11)*
3. **R3.3** Cancellation MUST transition through `CANCELLING` to `IDLE(cancelled)`, and MUST complete without a network response. *(A11)*
4. **R3.4** Shutdown MUST signal cancellation and **join with a timeout**. It MUST NOT rely on a language-level "abandon the thread" behavior. *(A12)*
5. **R3.5** **All transient storage MUST be reclaimed on cancellation and on shutdown**, by a path the shutdown sequence owns — never solely by a cleanup block inside an abandonable worker. *(A12)*
6. **R3.6** After cancellation or shutdown, **no artifact may appear**. *(A12)*

> **Why R3.4–R3.6 are worded as they are.** In the original, cleanup lives in a `finally` inside a daemon thread, and a probe confirmed that block **does not run** at interpreter exit: the render directory survives. A faithful port would inherit this. Cleanup ownership MUST sit with the shutdown sequence.

### R4 — Errors name the subsystem that actually failed *(D2.10, S5.1, S5.2, C03)*
1. **R4.1** Recognition failures MUST be typed by cause — `encode-failure` (local, e.g. the page image unreadable), `transport-failure`, `protocol-mismatch`, `empty-result`, `timeout` — and each MUST produce a distinct user-facing message. *(A13)*
2. **R4.2** A local failure MUST NOT be attributed to the server, and MUST NOT include the model tag as if the model were implicated. *(A13)*
3. **R4.3** An unrecognized response shape MUST fail loudly as `protocol-mismatch` and MUST NOT be silently coerced into an empty result. *(A14)*
4. **R4.4** Every message MUST identify the page as `n/N` and, for genuinely remote failures, the model tag. *(A13)*
5. **R4.5** User-facing documentation MUST enumerate every cause of a given symptom, not only the most memorable one.

### R5 — Verified input handling *(all verified contracts; preserve exactly)*
1. **R5.1** Accept `.pdf`, `.png`, `.jpg`, `.jpeg`, `.webp`, matched **case-insensitively**. Validate in order — exists, is a regular file, extension supported, readable — with a distinct message each, **first failure winning**. *(A15)*
2. **R5.2** Normalize the server URL by trimming whitespace and **all** trailing slashes, requiring an `http`/`https` scheme and a host, and **preserving any path prefix**. Never append an API path. *(A16)*
3. **R5.3** List model tags deduplicated, whitespace-trimmed, empties discarded, sorted case-insensitively. **An empty list is not an error.** A tag the user typed MUST survive a refresh. *(A17)*
4. **R5.4** Name the artifact `<input stem>_extracted.md` in the input's directory, preserving dotted stems. *(A18)*
5. **R5.5** Reject encrypted and zero-page documents **before rendering any page**, each with a distinct message. *(A19)*
6. **R5.6** A page-preview failure MUST be non-fatal: log it and continue recognizing. *(A20)*

### R6 — Data-loss guards actually guard *(S4.2, F4)*
1. **R6.1** Before publishing, the writer MUST re-check for an existing canonical artifact and MUST NOT overwrite one the user did not confirm — the confirmation MUST cover the state at **publish** time, not at start time. *(A21)*
2. **R6.2** Declining to overwrite MUST leave the artifact byte-unchanged and MUST report that the run was declined. *(A21)*

### R7 — Trust boundaries are explicit *(S4.1, S4.3, S4.6)*
1. **R7.1** The first use of a non-loopback server address MUST warn that document content will cross the network unencrypted and unauthenticated, and MUST require confirmation. *(A22)*
2. **R7.2** Requiring TLS MUST be configurable. *(A22)*
3. **R7.3** Document parsing is the primary untrusted-input boundary: parser dependencies MUST be version-pinned and locked, and R2.3's limits applied before parsing. *(A7)*
4. **R7.4** Recognized text MUST be treated as untrusted: any in-app preview MUST NOT render embedded HTML or load remote resources.

### R8 — Configuration is reachable and persistent *(D6.1, D6.2, D6.4)*
1. **R8.1** Settings MUST resolve from built-in defaults overridden by a user config file, with documented precedence. *(A23)*
2. **R8.2** The two prompt strings MUST be user-overridable and **versioned**, so a changed prompt is attributable. *(A23)*
3. **R8.3** Last-used server URL, model tag, and DPI MUST persist across launches. *(A24)*
4. **R8.4** Numeric settings MUST be range-checked at load, distinguishing absent from explicitly zero, and a malformed config MUST fail with a clear message rather than silently defaulting. *(A25)*
5. **R8.5** All dependencies MUST be pinned with an upper bound and a committed lockfile. The runtime version floor MUST be enforced by packaging metadata or an entry-point assertion. *(A26)*

### R9 — Observability *(D2.9, D2.7, D2.8)*
1. **R9.1** Diagnostics MUST be written to a durable per-user log, not only to an in-memory view. *(A27)*
2. **R9.2** Transient-storage cleanup failures MUST be logged, naming what could not be removed. *(A27)*
3. **R9.3** A platform action that cannot report failure MUST be pre-validated so a real failure surfaces. *(A28)*

### R10 — Guards precede what they guard *(D1.6, Decision 4)*
1. **R10.1** A capability-conditional test suite MUST skip cleanly when the capability is absent, and MUST NOT error at collection. Any guard MUST be evaluated **before** the dependency it protects is loaded. *(A29)*
2. **R10.2** The project MUST document one canonical test command, and it MUST report only passes and skips in an environment without a display or GUI toolkit. *(A29)*
3. **R10.3** Any behavior asserted only by a suite that cannot run in CI MUST be labelled as unverified in project documentation.

> **Why a spec constrains its own test harness.** In the original, the GUI suites' headless-skip guard sits *after* `import app`, so on a host without the toolkit all five error at collection. That single misplacement hid ~470 lines of never-executed contracts and is why 14 of the inherited acceptance scenarios carry weak evidence. R10 exists so the port cannot repeat it.

## Protocols and Persisted State

**Event contract** (closes the `porting-CF3` event-pinning obligation). Nine event kinds, extended per Decision 3. Every event MUST carry `{job id, sequence number, kind, payload}` — the original had none of the first two, which is why it is safe only while exactly one operation runs.

| Kind | Class | Payload | Delivery |
|---|---|---|---|
| `log` | observational | `{message}` | may drop/coalesce |
| `progress` | observational | `{phase: prepare\|recognize, completed: int, total: int}` — **`completed`, not `started`** (D1.1, Decision 2) | may coalesce |
| `page_image` | observational | `{page, total, image}` | may drop |
| `page_delta` | observational | `{page, text}` — MUST also carry `total` for schema uniformity (H10) | may coalesce |
| `page_result` | observational | `{page, total, status: recognized\|empty\|failed, text}` | may drop |
| `models_listed` | **state-bearing** | `{tags: [str]}` | guaranteed |
| `list_failed` | **state-bearing** | `{message, cause}` | guaranteed |
| `run_finished` | **state-bearing** | `{status: completed\|partial\|cancelled, artifact path, page summary}` | guaranteed |
| `run_failed` | **state-bearing** | `{message, cause, page?}` | guaranteed |

**Ordering (MUST preserve):** per page `page_image?` → `page_delta*` → `page_result`; all `prepare` progress precedes all `recognize` progress; exactly one terminal state-bearing event per run, emitted **after** all cleanup; page numbers ascend `1..N` without gaps. Payload schemas MUST be uniform and validated **once**, at the channel boundary — not variably by each handler (H9, S5.7).

**State machine (M5).** `IDLE(outcome?)` → `LISTING_MODELS` → `IDLE`; `IDLE` → `CONVERTING` → `IDLE(completed|partial)`; `CONVERTING` → `CANCELLING` → `IDLE(cancelled)`; any → `CLOSING`. **Two additions over the original, both mandatory:** a `CANCELLING` state, and an outcome carried on `IDLE` so retry can be offered. The original had neither, which is precisely why it can never offer retry.

**Provider protocol.** One `chat`-style streaming request per page. Messages: a system message carrying the system prompt, and a user message carrying the user prompt plus the page image. **The page image MUST be transmitted as base64-encoded file bytes in the request body** — established by inspecting and exercising the reference client; the path never crosses the wire. The file MUST remain readable and unmoved for the duration of its request. **Encode-time errors MUST be typed `encode-failure`, distinctly from transport errors** (R4.1). No generation parameters are sent; server defaults apply. No authentication exists in the reference protocol.

**Persisted artifact.** UTF-8, no BOM, **LF-only line endings on every platform** (a Windows port MUST NOT emit CRLF), pages joined by exactly one blank line, each page trimmed. No header, footer, or page markers — meaning **page boundaries are not recoverable from the artifact**; a port needing them MUST introduce an explicit marker and treat that as a format change. Publication: stage in the destination directory → flush → **fsync** → atomic same-filesystem rename. No history, no journal, no locking; concurrent writers resolve last-writer-wins. A `partial` artifact MUST be distinctly named so it can never be mistaken for canonical.

**Settings.** A user config file over built-in defaults, documented precedence, range-checked numerics, versioned prompts, and persisted last-used URL/model/DPI.

## External Dependencies

| Dependency | Stance | Rationale |
|---|---|---|
| Vision-model server (Ollama) | **wrap** | The protocol is simple and the product's identity. Wrap behind M3 so a second provider is additive. Base64 encoding is the adapter's job, not the kernel's. |
| PDF rasterizer (PyMuPDF equivalent) | **wrap** | Behind M2. Native, untrusted-input-facing, and the main CVE surface — keep it isolated, pinned, and replaceable. |
| Image library (Pillow equivalent) | **wrap** | Behind M2/M8 for thumbnails only. |
| GUI toolkit (customtkinter/Tk equivalent) | **replace** | Choose per target platform. The original's toolkit imposed the whole queue-plus-polling design; do not carry that inward. Note the original pins a major version because an older one rendered blank windows — expect toolkit-specific traps. |
| HTTP client | **wrap** | Behind M3, with the two independent timeouts of R2.1. |
| Clipboard | **replace** | Use a platform API with ownership transfer; the original's clipboard does not outlive the process on X11. |
| OS open/reveal | **wrap** | Thin, per-platform, argv-vector based (never a shell string). Pre-validate where the platform cannot report failure. |
| Test doubles for provider, page source, clock, cancellation | **emulate** | Required by §Implementation Sequence — the acceptance harness is built before any real adapter. |
| Concurrency runtime | **replace** | Re-derive from the target's model. Do **not** transliterate daemon threads plus 50 ms polling. |
| Persistence beyond the artifact | **postpone** | No database or session store is needed. |
| Batch/multi-document processing | **postpone** | Explicit non-goal; the event schema's job id leaves room. |

## Portability Hazards

Ranked by how likely each is to be reproduced by a faithful port.

1. **Cleanup inside an abandonable worker (high).** The original's `finally` provably does not run at interpreter exit. Any language with detached-thread or fire-and-forget task semantics can inherit this. → R3.4–R3.6.
2. **Image transport (high).** The reference client silently converts a path into base64 bytes. A port copying the *call shape* rather than the *wire format* will send a path string and fail confusingly. → §Protocols.
3. **UI-context affinity (high).** The original's queue and 50 ms poll exist solely to satisfy toolkit thread affinity. Transliterating them into an async runtime produces a needless timer; ignoring affinity entirely produces crashes. → M6, M8.
4. **A local error inside a remote error wrapper (medium).** Already live in the original. → R4.1–R4.2.
5. **Unbounded growth in three places at once (medium).** Deltas land in the channel, an accumulator, and the view simultaneously. Bounding only one leaves the pressure. → R2.2, R2.4, R2.6.
6. **Same-filesystem rename (medium).** Staging in a system temp directory silently loses atomicity. → M4.
7. **LF normalization (medium).** Deliberate and verified in the original; a Windows port "fixing" it changes the artifact. → §Protocols.
8. **Lock-free correctness by confinement (medium).** The original uses no locks and is *correct*, because all mutable UI state is confined to one thread. A port must re-establish confinement **explicitly** — the absence of locks is a consequence, not a substitute. → M5, M8.
9. **Provider response-shape coercion (medium).** `getattr(...) or ""` is right for a missing optional and wrong for a missing required field. → R4.3.
10. **A guard after its dependency (medium).** → R10.
11. **Positional indices over identity (low).** The original tracks the review position by list index; concurrent recognition would shift it under the user. Track page *number*. → M8.
12. **Zero-padded filenames, platform reveal conventions, `xdg-open` availability (low).** Incidental; reproduce or replace deliberately.

## Implementation Sequence

**Kernel-first with fake-driven acceptance tests.** The first artifact is a deterministic harness, not a working product. The parts most likely to be subtly wrong are built first, in isolation, under tests, before anything depends on them.

**Phase 0 — Acceptance harness (no real adapters).** Fake provider yielding scripted deltas, including empty, oversized, malformed-shape, mid-stream-error, and never-ending streams; fake page source with scripted page counts and render failures; fake clock and cancellation; temp-directory fixtures; an in-memory event observer that records order. **Nothing real is built until this is green.**

**Phase 1 — Artifact Writer (M4).** Crash-safe publication first, because it is the only irreversible operation. Includes fsync, staging placement, and the R1.4/R1.5 guards. Tests: A3, A4, A18, plus interrupted-write injection.

**Phase 2 — Event Sink (M6) and typed event schema.** Bounded channel, one validation point, the observational/state-bearing distinction, and guaranteed delivery for state-bearing events. Tests: A8, ordering, and per-handler failure isolation.

**Phase 3 — Operation State Machine (M5).** All four states including `CANCELLING`, plus the outcome on `IDLE`. Tests: A11, exclusion, and that no busy state lacks a network-independent path to `IDLE`.

**Phase 4 — Clock and Cancellation (M7).** Both independent limits, injectable. Tests: A5.

**Phase 5 — Recognition Orchestrator (M1) over fakes.** Retry, per-page outcomes, partial assembly, cancellation checks, typed error classification. This is the kernel; it MUST pass A1, A2, A5, A6, A11, A13, A14 with **no real adapter present**.

**Phase 6 — Real adapters, one at a time.** Page source (lazy rendering, R2.5), then recognizer (base64 encoding, both timeouts, typed errors). Each proven against the Phase 0 fakes' contract before substitution.

**Phase 7 — Settings Registry (M9).** Precedence, range checks, versioned prompts, persistence. Tests: A23–A26.

**Phase 8 — Presentation Shell (M8).** Last. Includes the single state→controls mapping, the cancel affordance, live text, and the corrected progress model. **The import-boundary check (kernel MUST NOT import the shell) is a build-time gate, not a review convention.**

**Phase 9 — Optional surfaces.** Review pane, preview, completion actions, clipboard.

### Scope Tiers

**Minimum viable port** — usable at all: R5 (validated input), M2 image + PDF paths, M3 with base64 encoding and both timeouts, M1 with retry and partial outcomes (R1), M4 with fsync-atomic publication, M6 bounded, M5 with cancellation (R3), R4 typed errors, and a shell with file selection, server/model/DPI entry, start, cancel, progress, and a terminal result. Explicitly **includes** cancellation and partial success — they are correctness, not polish.

**Major-workflow parity** — covers the primary use cases: live streamed text with the byte-identical-to-artifact invariant, per-page previews, the corrected progress model, overwrite confirmation at publish time (R6), persisted settings (R8), the durable log (R9), and the non-loopback warning (R7.1).

**Full parity** — everything the original does: the review pane with image↔text pairing and bounded image caching, the completion dialog with platform open/reveal, clipboard copy, and platform-specific naming of the reveal action.

## Acceptance Scenarios

Black-box, concrete, no reference to internals. The **Origin** column preserves the inherited evidence tiering per convention C04 — this is what closes `porting-CF3`: `V` = inherited from a *passing* original test; `A` = inherited but **asserted-not-verified** in the original (GUI-suite only, likely never executed); `NEW-FIX` = the original **demonstrably fails** this; `NEW` = new obligation from this spec.

| # | Scenario | Input | Expected output / side effect | Origin |
|---|---|---|---|---|
| A1 | Page failure yields partial success | 5-page document; server errors on page 3 past all retries | Run status `partial`; a distinctly-named partial artifact contains pages 1,2,4,5 with page 3 explicitly marked failed; canonical artifact untouched | NEW |
| A2 | Blank page is not an error | 3-page document; page 2 yields no text | Run status `completed`; artifact contains pages 1 and 3 with an empty section for 2; no error dialog | NEW-FIX |
| A3 | Interrupted publication never corrupts | Existing canonical artifact `OLD`; kill the process during the write | Artifact is byte-identical to `OLD`; no staging file remains | V |
| A4 | Total failure writes nothing | 2-page document; every page fails all retries | Status `failed`; no artifact created; any existing artifact byte-unchanged | V |
| A5 | Both timeouts fire independently | (a) stream stalls past the idle timeout; (b) stream drips slowly past the per-page ceiling | Both fail that page with distinct `timeout` messages; the run continues per R1 | NEW-FIX |
| A6 | Output caps enforced | Server streams beyond the per-page cap | That page fails with a distinct cap message; memory does not grow without bound; run continues per R1 | NEW |
| A7 | Input bounds enforced before parsing | Document exceeding the page-count or byte-size limit | Rejected with a clear message **before** any parsing; no transient storage created | NEW |
| A8 | Backpressure is real | Fake provider emitting deltas far faster than the observer consumes | Producer is slowed; channel occupancy stays bounded; no unbounded memory growth | NEW |
| A9 | Rendering is lazy | 50-page document at maximum DPI | At most a small constant number of rendered pages exist at once; peak transient storage is independent of page count | NEW |
| A10 | Observer never blocks unboundedly | Burst of many page-image events at once | UI stays responsive; per-cycle work is capped; decoding happens off the UI context | NEW |
| A11 | Cancellation is prompt and clean | Start a 20-page run; cancel at page 5 | State reaches `IDLE(cancelled)` without waiting for a network response; **no artifact appears**; all transient storage reclaimed | NEW-FIX |
| A12 | Shutdown means shutdown | Start a 20-page run; quit the application at page 5 | Process exits; **no transient storage survives**; **no artifact appears afterward** | NEW-FIX |
| A13 | Local vs remote errors are distinguished | Delete the page image between render and request | Error is typed `encode-failure`, names a local file problem, **does not mention the server, and does not include the model tag** | NEW-FIX |
| A14 | Unrecognized response shape fails loudly | Provider returns a response whose text field is renamed | Error is typed `protocol-mismatch` and says so; **it does not report "no text returned"** and does not implicate the model | NEW-FIX |
| A15 | Input validation, ordered, exact | `missing.pdf`; a directory named `d.pdf`; `notes.txt`; an unreadable `x.pdf`; `UPPER.PDF` | Four distinct messages, first-failure-wins; `UPPER.PDF` accepted | V |
| A16 | URL normalization | `  http://h:11434//  `; `https://s.lan/ollama/`; `ftp://h`; `""` | First two normalize (trailing slashes stripped, path prefix kept, no API path appended); last two rejected with distinct messages and **no network call** | V |
| A17 | Model listing | Server returns `zeta:7b, Alpha:12b, "  ", zeta:7b, null, "beta:2b "`; then an empty list; then with a user-typed tag present | `Alpha:12b, beta:2b, zeta:7b`; empty list is **not** an error; typed tag survives | V |
| A18 | Artifact naming | `report.v2.pdf`; `scan.png` | `report.v2_extracted.md`; `scan_extracted.md`, both beside the input | V |
| A19 | Malformed documents rejected early | Encrypted PDF; zero-page PDF | Distinct messages; **no page rendered**; no transient storage left | V |
| A20 | Preview failure is non-fatal | Thumbnail generation fails for page 2 of 3 | All three pages recognized; the failure is logged; page 2 still reviewable without an image | V |
| A21 | Overwrite guard covers publish time | No artifact at start; another process creates one during the run | User is asked before replacement, **or** the run reports it was declined; the file created mid-run is **not** silently destroyed | NEW-FIX |
| A22 | Non-loopback warning | Enter a non-loopback server address for the first time | Warning states document content crosses the network unencrypted and unauthenticated; requires confirmation; a TLS-required setting is available | NEW |
| A23 | Settings precedence and prompt override | Config file overriding the server URL and system prompt | Both take effect; precedence is documented; the prompt override is version-attributable | NEW |
| A24 | Settings persist | Set a non-default URL, model, and DPI; restart | All three are restored | NEW |
| A25 | Malformed config fails clearly | Config with a negative timeout, and one with a syntax error | Both fail with clear messages naming the offending key; neither silently defaults; absent is distinguishable from explicit zero | NEW |
| A26 | Dependencies and runtime pinned | Fresh install from the lockfile; run on an unsupported runtime version | Install resolves exactly; the unsupported runtime is refused with a clear message | NEW |
| A27 | Durable diagnostics | Complete one run, then force a failure, then quit | A per-user log file records both, including the failure cause and any cleanup failure | NEW |
| A28 | Platform action failures surface | Reveal-in-file-manager invoked on a path that no longer exists | An error is reported to the user and logged; it does **not** silently no-op | NEW-FIX |
| A29 | Headless test run is green | Run the canonical test command with no display and no GUI toolkit installed | Reports only passes and skips — **zero collection errors** | NEW-FIX |
| A30 | Live text matches the artifact | 3-page document, watching live output | Live text is byte-identical to the artifact, one blank line between pages, no duplicated page text | A |
| A31 | Progress reflects completed work | 4-page document; and a single image | Fraction never reaches 100% until the last page is **complete**; a single-image job does not sit at 100% for its duration | A → NEW-FIX |
| A32 | One operation at a time | Trigger a second operation while one runs | The second is refused or queued, never run concurrently; controls reflect the state via one mapping | A |
| A33 | Review pairs pages, preserves position | 3-page document; stay on page 1 while 2 and 3 arrive | Still showing page 1; forward navigation enables; each page shows its own image and text; decoded-image memory stays bounded | A |
| A34 | Terminal presentation | Complete a run; then force a failure | Success surfaces the result view and offers open/reveal; failure surfaces the log view with the typed cause | A |
| A35 | Fresh-launch defaults | First launch with no config | Documented defaults; no preselected model; no file selected | A |

**Nine scenarios (A2, A5, A11, A12, A13, A14, A21, A28, A29, and A31's second half) encode behavior the original demonstrably fails or that only execution revealed.** They are the highest-value rows: a faithful port would otherwise reproduce every one.

## Deliberate Non-Goals

- **Batch or multi-document conversion.** One document per run. The event schema's job id leaves room without committing.
- **Concurrent page recognition.** Explicitly out of the MVP. Decision 3's primitives (job id, sequence number) make it addable later without redesign; adding it *before* them reintroduces the races the original avoids only by being sequential.
- **Resume across restarts.** Partial artifacts (R1.3) cover the practical need without a journal.
- **Any model management.** No pulling, installing, starting, or health-checking. Preserved from the original deliberately.
- **Authentication to the provider.** The reference protocol has none. R7.1's warning is the honest response, not an invented auth layer.
- **Recovering page boundaries from an existing artifact.** Not possible in the inherited format; a port needing it must change the format explicitly.
- **Preserving the original's exact progress arithmetic.** Deliberately changed (Decision 2, R2/A31).
- **Preserving the all-or-nothing discard.** Deliberately changed (Decision 1, R1) — the one contract break in this spec, and it is stated as such.
- **Cosmetic fidelity.** Window geometry, theme, fonts, and placeholder model tags carry no behavior.
- **Stubbed first:** platform open/reveal and clipboard, behind interfaces from Phase 1 so the shell is testable before they exist.

## Coverage and limits

- **Inspected scope:** `findings/porting/reverse-engineering-bundle.md` in full as the compression boundary, plus **three targeted deep reads** recorded in §Orchestrator duties discharged (the acceptance list, the event catalog and SM1, and two defect rationales). All 41 defects carry forward into a normative rule or an explicit non-goal; all four bundle decisions are specified; all three routed carry-forwards closed.
- **Skipped scope:** the architecture map, contracts, protocols, and both scan reports were **not** reloaded wholesale — the bundle was self-sufficient except at the three named triggers, which is the compression boundary working as designed. No new source reading and no new defect analysis was performed.
- **Secondary outputs:** this phase declares none.
- **Evidence basis:** upstream findings, including runtime verification gathered in earlier phases (service-suite execution, the daemon-thread probe, the provider serializer inspection). Every `MUST` traces to a defect disposition or a verified contract.
- **Known blind spots:** (1) **no target stack is chosen**, so hazard mitigations are stated as requirements rather than concrete techniques, and the concurrency and UI sections cannot be made specific (`q-target-stack`); (2) **no live model was ever contacted in this pipeline**, so nothing here constrains output *quality* — the prompt pair is specified as preserve-verbatim precisely because its behavior is unmeasured; (3) **14 inherited scenarios (Origin `A`) rest on tests that error at collection in the original**, so they specify intent rather than demonstrated behavior — the Origin column preserves that distinction rather than hiding it; (4) the performance thresholds behind R2's caps are deliberately unspecified, since the original's real load characteristics were never measured (`q-queue-depth-under-load`); (5) gitignored `build-mac.sh` and `docs/` may contain packaging or design context the whole pipeline never saw.
- **Coverage disposition:** COMPLETE for the reimplementation-spec scope, with the caveat that "complete" here means every inherited finding is dispositioned and every rule is testable — **not** that the spec is stack-specific. An opinionated re-run is the right follow-up once a stack is committed, and is recorded as a post-pipeline amendment rather than a gap.

## Known Unknowns

| ID | Kind | Description | Deferred Reason |
|---|---|---|---|
| `q-target-stack` | needs-maintainer-decision | No target language, runtime, UI toolkit, project identity, or module naming was committed, so the Strategic Alignment Hook's inputs are absent and this spec is language-agnostic by default. Concurrency, UI, and packaging rules are stated as requirements rather than techniques. | A product decision the orchestrator owns. Once committed, re-run this phase against `templates/reimplementation-spec-opinionated.md`; see `pp-opinionated-spec-rerun`. |
| `q-scope-tier-commitment` | needs-maintainer-decision | Which scope tier is the actual target was never stated. The tiers are defined, but whether the first release is the minimum viable port or major-workflow parity changes sequencing after Phase 8. | Requires the same product decision as `q-target-stack`. |
| `q-gui-tests-ever-run` | needs-maintainer-decision | Established that the original's five GUI suites error at collection without the toolkit, so they cannot have passed headless. Whether they pass on a machine with a display — and therefore whether ~470 lines of contracts were ever demonstrated — is unknown, and it sets how far the 14 `A`-origin scenarios can be trusted. | Needs a maintainer statement or a host with a display. R10 and A29 make the port immune regardless. |
| `q-prompt-provenance` | needs-maintainer-decision | The system prompt drives output quality more than any other setting and is specified as preserve-verbatim, but nothing records how it was derived or against which models it was tuned. A port cannot tell which clauses are load-bearing, so it cannot safely simplify it. | Project history, not source. R8.2's versioning is the mitigation: changes become attributable even while provenance is unknown. |
| `q-queue-depth-under-load` | needs-runtime-test | Real steady-state channel occupancy and delta rates against a production-grade model were never measured, so R2's caps and bounds are specified as requirements without numeric thresholds. | Needs a GPU-backed model on a multi-page document. A prototype spike sets the numbers; see `pp-measure-queue-depth`. |
| `q-ctk-image-clear` | needs-runtime-test | In the original, clearing a displayed image relies on toolkit-version-dependent null-image behavior. Whether it actually clears was never verified. | Toolkit-specific and moot once a stack is chosen: M8 requires explicit clearing rather than a null assignment. |

## Carry-Forward

`reimplementation-spec` is the terminal phase, so nothing can be routed to a downstream phase. All three items below are **post-pipeline work**, recorded in the handoff's `post_pipeline` list; the "Target" column names the *kind* of follow-up rather than a pipeline phase.

| ID | Target | Description | Deferred Reason |
|---|---|---|---|
| `spec-CF1` | spike (post-pipeline) | Establish R2's numeric thresholds — per-page and per-document output caps, channel bound, per-cycle observer budget, and maximum input page count and byte size — by measuring a real model on real documents. The rules are normative; only the constants are open. | Requires executing against a production-grade model, which no pipeline phase performs. Guessing the constants would make R2 either toothless or wrong. |
| `spec-CF2` | amendment (post-pipeline) | Re-run this phase against the opinionated template once a target stack, project identity, module naming, and scope tier are committed, replacing requirement-shaped hazard mitigations with concrete techniques. | The Strategic Alignment Hook's inputs do not exist yet (`q-target-stack`, `q-scope-tier-commitment`). Producing a stack-specific spec now would invent commitments the orchestrator has not made. |
| `spec-CF3` | spike (post-pipeline) | Prototype the provider adapter against a live server to confirm the complete request envelope beyond image encoding — headers, sibling fields, and error-response shapes — so R4's typed error classification maps onto real failures rather than inferred ones. | Image encoding is established, but no traffic was ever captured. R4.1's taxonomy is derived from client code, and a live check is cheap insurance before Phase 6. |

## Spike List

1. **Threshold calibration** (`spec-CF1`) — measure delta rates, channel occupancy, per-page latency distribution, and peak transient storage at each DPI, on documents of 1, 10, and 500 pages. Output: the constants for R2.
2. **Provider envelope** (`spec-CF3`) — capture real requests and error responses; verify that every R4.1 error class is reachable and distinguishable in practice.
3. **Crash-safety validation** — inject process kills and simulated power loss around the M4 write window; confirm A3 holds and no staging file survives. The original's `fsync` gap is exactly what this catches.
4. **Cancellation latency** — measure worst-case time from cancel to `IDLE(cancelled)` with the largest page in flight; confirm A11's "without waiting for a network response" is achievable, and set a documented upper bound.
5. **Shutdown cleanup under adversity** — kill the process at ten points across a run; confirm A12 in every case. This is the spike that most directly guards against inheriting the original's proven leak.
6. **UI-context responsiveness** — burst page-image events at the shell on the slowest target platform; confirm A10's cap and off-context decoding hold.
7. **Import-boundary enforcement** — confirm the build-time gate actually fails when a kernel module imports the shell. The original's equivalent rule was convention-only, and its *test* suites are where that cost showed up.
8. **Toolkit capability guard** — verify A29 on a machine with no GUI toolkit at all, not merely no display. That distinction is precisely what the original got wrong.

---

## Validation

| # | Criterion | Result | Evidence |
|---|-----------|--------|----------|
| 1 | Concept-level modules are defined. | PASS | §Conceptual Module Model defines nine modules (M1–M9), each with responsibility, public inputs, public outputs, owned state, invariants, and collaborators; §Layer Split assigns every one to core semantics / adapter / delivery surface and states the kernel-buildability rule. |
| 2 | Required behaviors are stated. | PASS | §Required Behaviors gives 10 normative groups (R1–R10) comprising 44 numbered MUST rules, each traced to a defect disposition or verified contract and cross-referenced to the scenario that proves it. |
| 3 | Protocol and persisted state expectations are stated. | PASS | §Protocols and Persisted State pins the nine-event contract with payloads, delivery classes, and mandatory `job id` + `sequence number`; the four-state machine including the new `CANCELLING` state and outcome-carrying `IDLE`; the provider protocol including **base64 body encoding**; and the artifact format with its fsync-atomic publication rule. |
| 4 | Acceptance scenarios and known unknowns are included. | PASS | §Acceptance Scenarios — 35 black-box scenarios (A1–A35) with concrete inputs and observable outcomes, each tagged by evidence origin (`V` / `A` / `NEW` / `NEW-FIX`) per convention C04. §Known Unknowns lists six with kinds and deferral reasons; §Spike List gives eight prototypes. |
| 5 | Defects identified in either scan are explicitly designed-around or noted as "left behind", with the choice cited. | PASS | All 41 defects from both scans are dispositioned: the 4 high and 16 medium each become a numbered MUST rule (R1–R10) citing the defect ID; the 21 low are absorbed into R1–R10 or named in §Deliberate Non-Goals (D1.2, S5.5, D6.5, D6.7, S4.5 as left-behind). §Portability Hazards ranks the ten most likely to be re-inherited. |
| 6 | Findings are marked with evidence levels. | PASS | Inherited claims carry their upstream tier: `observed fact` for runtime-verified items (the proven cleanup leak, the base64 encoding, the passing service contracts), and the `asserted-not-verified` tier surfaced as `A` in the Origin column. Rules derived by synthesis are marked `strong inference` where stated in the bundle. |
| 7 | Coverage and limits name inspected scope, skipped scope, evidence basis, and blind spots. | PASS | §Coverage and limits names all four plus a COMPLETE disposition, and — per the skill — **records the three targeted deep reads and their triggers**, confirming the bundle was otherwise self-sufficient. Five blind spots are listed, led by the absent target stack. |
| 8 | Lower-level findings are deep-read only when the porting bundle identifies a gap, conflict, missing acceptance detail, or defect rationale. | PASS | Exactly three deep reads, each against a trigger the bundle named explicitly: the acceptance list and the event/state tables (both omitted by design and routed as `porting-CF3`), and two defect rationales behind rules that became MUSTs. No upstream report was reloaded wholesale; the architecture map and both scan reports were used only through the bundle. |

**Validated by:** 2026-08-18 (reimplementation-spec phase, MCP-driven session 2, framework v0.16.0, selection: language-agnostic)
**Overall:** PASS
