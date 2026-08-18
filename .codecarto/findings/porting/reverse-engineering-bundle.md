# Reverse-Engineering Bundle — Local-OCR

Source: `../`, commit `8e7388c` · Pipeline `full-with-deep-audit` · Framework v0.16.0 · Date 2026-08-18
**Closes `contracts-CF1`, `contracts-CF2`, `contracts-CF4`, `protocols-CF1`.**

This bundle is the pipeline's compression boundary. `reimplementation-spec` should start here and deep-read upstream only where the Source Index names a trigger.

### Orchestrator duties discharged

**Open-question re-triage** (5 inherited). All five labels re-tested and **confirmed**; each needs a resource this environment lacks. `q-gui-tests-ever-run`, `q-test-invocation`, `q-prompt-provenance` → a maintainer statement. `q-ctk-image-clear` → a `tkinter` build. `q-queue-depth-under-load` → a GPU-backed model. Two are material to the port and restated below; three describe source-implementation behavior a port will not reproduce and are resolved by dispositions instead. Convention C01 was applied to each and reaches none.

**Contradiction sweep.** No new contradictions. The one the semantic phase found — measured client compatibility versus the "reachable today" premise behind the provider-drift severity — is already reflected: S5.1 carries `medium` here, not `high`.

## System Summary

Local OCR converts one document at a time — a PDF or a single image — into a Markdown file, by sending each page as an image to a vision-language model on an Ollama server the user names, and concatenating the results. It is a **single-user desktop application with no server, no CLI, no accounts, and no persistence beyond the output file.** Its product promise is privacy: the only outbound destination is the URL the user types, and the app never installs, pulls, or starts a model itself.

The system is small — 1,178 lines across four modules — and unusually disciplined for its size. A service layer holds all logic and is deliberately free of UI imports; a GUI layer owns every widget and all mutable state; one unbounded queue carries `(kind, payload)` tuples from a short-lived daemon worker to the UI thread, drained every 50 ms. Roughly 1,540 lines of tests pin behavior precisely enough that **the tests, not the README, are the specification** — with one crucial caveat below.

The audit found **no critical defects** and nothing that silently produces wrong OCR output. It found a system that is correct on the happy path and **unbounded in every other direction**, with one dominant design decision behind most of its worst behavior: *the run is an all-or-nothing transaction over N independent network calls.* One failed page, one blank page, or one mid-run quit discards every page already recognized. A reimplementation that changes only one thing should change that.

**The caveat, and it shapes how this bundle should be trusted:** the specification is two-tier. The 1,069-line service suite was **executed and passes** (72/1-skipped), so contracts it pins are verified. The five GUI suites (~470 lines) **error at collection** on a host without `tkinter` — their skip guard sits after the import it guards — so every contract resting only on them is *asserted, not verified*, and has very likely never run anywhere headless. The GUI is simultaneously the only interactive surface and the least-evidenced half of the spec.

## Source Index

| Area | Canonical upstream section | Carried forward | Deep-read trigger |
|---|---|---|---|
| Architecture | `architecture-map.md` §Layer Map, §Concurrency Model | Four-module acyclic stack; `config` the stable base; Tk boundary verified by execution; one queue, no locks, one worker | Only for the full 16-row porting-priority table or the durable-state inventory verbatim |
| Contracts | `behavioral-contracts.md` §Feature Contracts | 13 contracts across 3 surfaces; exact validation strings | For a contract field not summarized in the table below |
| Contracts (acceptance) | same, §Black-Box Acceptance List | **42 scenarios with a verification-status column** — not reproduced here | **Deep-read when writing the spec's acceptance section.** This is the parity harness; the status column is load-bearing (C04) |
| Contracts (evidence tiering) | same, §Verification status of the evidence | Service suite verified; GUI suites asserted-not-verified | Whenever deciding how much weight a UI contract carries |
| Protocols | `protocols-and-state.md` §Event Catalog, §State Machine | 6 boundaries; 9-event catalog; 4 state machines; 14 hazards | **Deep-read §B1 payload schemas and SM1's 22 transitions when specifying the port's event contract** |
| Protocols (wire encoding) | same, §B3 wire encoding | **Resolved:** page images travel as base64 file bytes in the JSON body; path never sent | For the exact serializer branch behavior when writing the provider adapter |
| Protocols (persistence) | same, §Persistent Schema Notes | Mutable whole-file replace, no history, no locking, page boundaries unrecoverable | If the port adds versioning, resume, or multi-instance safety |
| Defects — mechanical | `mechanical-defects.md` | 23 findings (3 high, 8 medium, 12 low), passes 1/2/6 | For per-finding evidence behind any disposition below |
| Defects — semantic | `semantic-defects.md` | 18 findings (1 high, 8 medium, 9 low), passes 3/4/5; 7 routed items resolved, 1 refuted, 1 downgraded | For the thread-confinement proof (§Verified Safe) if the port changes threading model |

Everything load-bearing is carried below. The two genuine omissions — the 42 acceptance scenarios and the full event/state tables — are named above with explicit triggers; reproducing them would duplicate rather than compress.

## Layer Map With Ownership

Concept names, not source names. A port should not assume the same file layout.

| Layer / Module | Role | Owns |
|---|---|---|
| **Settings registry** (`config.py`) | core semantics (constants) | Every tunable: extension allowlist, DPI set, two timeouts, three UI intervals, and — critically — the **two prompt strings**. Imports nothing; nothing is below it. |
| **Conversion engine** (`ocr_service.py`) | core semantics + integration adapter | Validation, URL normalization, rasterization, thumbnailing, the per-page recognition loop, atomic save, OS integration. **Owns every rule; owns no UI.** Reaches the UI only through injected callables. |
| **Presentation shell** (`app.py`) | UI / product shell | All widgets, all mutable UI state, worker launch, the main-thread event pump. Owns `OperationState`, the streaming de-duplication, the review navigation model. |
| **Entrypoint** (`main.py`) | product shell | Theme selection and `mainloop()`. 16 lines. |

**The one structural invariant to preserve by name:** the conversion engine must not import the UI toolkit. That rule is what makes the service suite headlessly executable — **demonstrated this run**, by running it on a host with no `tkinter` at all. A port should enforce it **mechanically** (import lint, module boundary, separate package) rather than by convention, which is all the source has, and which is exactly how the *test* suites lost the same property (D1.6).

## Feature Contract Table

| Feature | Surface | Priority | Key Contracts | Notes / Defects |
|---|---|---|---|---|
| Per-page recognition loop | Engine ↔ Ollama | **core** | One independent `stream=True` request per page; exact messages; text stripped; empty rejected; no cross-page context | The product *is* this loop. D2.1, D2.2, S3.1, S4.4, S5.1 land here |
| Prompt pair | Engine ↔ Ollama | **core** | Asserted verbatim by a **passing** test; instructs Markdown output, structure preservation, no preamble | Behavioral asset, not a literal. Port verbatim, version it. `q-prompt-provenance` open |
| PDF rasterization | Engine | **core** | `dpi`, `csRGB`, `alpha=False`; `page_%04d.png`; password-protected and zero-page rejected **before** any render | D6.3, S3.5 — eager today; should stream |
| Provider adapter (image encoding) | Engine ↔ Ollama | **core** | **Base64 file bytes in the JSON body**; path never sent; file must be readable at request time, per page | Resolved this run. H1, H14 |
| Output artifact | Storage | **core** | `<stem>_extracted.md` beside input; UTF-8; LF-only; pages joined `"\n\n"`; no header/footer/markers | D2.4, S5.3. Page boundaries unrecoverable from the file |
| Atomic publish | Storage | **core** | Temp-then-rename, same directory; failure preserves existing and leaves no `.tmp` | Keep the mechanism; add `fsync` (D2.4) |
| Worker→UI event contract | Engine ↔ Shell | **core** | 9 kinds; 4 orderings; exactly one terminal event, after cleanup | Only 4 events are state-bearing — see Protocol Notes |
| Input validation | Shell + Engine | **important** | 4 ordered checks with exact messages; case-insensitive extensions | All **verified**; every message user-visible |
| URL normalization | Engine | **important** | Trim whitespace and all trailing slashes; preserve path prefix; require http/https + host; never append `/api` | **Verified.** S4.3 — no guardrail on non-loopback hosts |
| Model discovery | Engine ↔ Ollama | **important** | Dedup, strip, drop empties, case-insensitive sort; empty list **not** an error; typed tag survives refresh | **Verified.** S5.1 shares the defensive-read hazard |
| Single-operation exclusion | Shell | **important** | `OperationState`; every entrypoint no-ops unless IDLE | Lock-free and **verified correct** (semantic §Verified Safe). D1.3/S5.6 are presentation gaps only |
| Overwrite confirmation | Shell | **important** | Prompt when output exists; declining is a silent no-op | S4.2 — check is minutes before the write |
| Progress model | Shell | **important** | render `0.2·c/t`; ocr-after-render `0.2+0.8·c/t`; ocr-only `c/t` | D1.1 / Decision 2. **Asserted-not-verified** (GUI suite) |
| Live streaming + de-duplication | Shell | **important** | Panel byte-identical to the saved file; separator embedded at the page boundary in the buffer | Subtle and correct; preserve the *invariant*. **Asserted-not-verified** |
| Review tab | Shell | optional | Image↔text pairs; navigable once text ready; position preserved; 5-entry decoded LRU | D1.5 latent; D2.5 retains raw bytes unbounded. **Asserted-not-verified** |
| Live preview thumbnail | Shell | optional | 900 px longest side; never upscaled; **failure is non-fatal and logged** (**verified**) | The non-fatal rule is what matters |
| Completion dialog | Shell + OS | optional | Non-blocking toplevel; Open / platform reveal / OK | D2.8, S4.6. Dialog itself asserted-not-verified; the six OS branches **verified** |
| Copy button | Shell | incidental | Whole-panel clipboard copy | D6.6 — X11 clipboard dies with the process |
| Theme, geometry, fonts, `EXAMPLE_MODELS` | Shell | incidental | Dark mode, `780x680`, Courier New, two placeholder tags | No behavioral content. D6.5 |

## Protocol and State Notes

**The event contract** (protocols §B1). Nine kinds over one unbounded FIFO queue. Four orderings hold and must be preserved, **all verified**: per-page (`page_image`? → send log → `stream_chunk`* → `page_text`), per-phase (all render before all OCR), terminal (exactly one success-or-error, last, after cleanup), and page monotonicity.

**The distinction that matters most for a port:** events split three ways.

- **Observational** — `log`, `progress`, `page_image`, `stream_chunk`, `page_text`. The saved Markdown is built from the recognition function's **return value**, not the event stream, so losing every one still yields a byte-identical file. Best-effort.
- **Synchronous barriers** — three, internal to the worker: render-all before recognize-any; recognize-all before save; cleanup before the terminal event.
- **State-bearing** — `models_loaded`, `refresh_error`, `ocr_success`, `ocr_error`. **Only these four drive a transition, and losing one strands the machine permanently** (D2.6). Guaranteed delivery.

**The state machine** (SM1). Three states, 22 transitions. Three properties define it more than the transitions: **no cancellation transition**, **no error state** (failure returns straight to IDLE with no memory of the outcome — why the app can never offer "retry"), and **exactly four exits from a busy state**. Note SM2/SM3/SM4 (progress mode, result streaming, review nav) are real machines pinned **only** by the unexecuted GUI suites.

**Wire encoding** (protocols §B3, resolved). Page images are handed over as path strings; the client **base64-encodes the file into the JSON body** — the path never reaches the server. A port must base64 the bytes, keep the file readable and unmoved for the whole request, and **classify encode-time errors separately from transport errors** (H14: today a missing render is reported as `Ollama request failed`).

**Persistence** (§B4). One durable artifact, mutable whole-file replace, no history, no journal, no locking, no dedup. Two instances racing → last-writer-wins. Page boundaries unrecoverable from the output.

**Concurrency, verified.** All mutable UI state is confined to the Tk main thread; the queue is the only shared object; workers only `put`. The semantic scan enumerated every access site and confirmed the lock-free design correct. A port must **re-establish that confinement explicitly** — the absence of locks is a consequence of the invariant, not a substitute for one.

## Portability Hazards

| Hazard | Source Phase | Impact | Mitigation |
|---|---|---|---|
| Images cross as **base64 file bytes**, encoded by the client from a path | protocols H1 (**resolved**) | **high** | Base64 the page bytes in the port's own adapter. No path-passing shortcut exists; the file must be readable at request time, per page |
| Tk thread affinity + 50 ms `after()` polling | architecture, protocols H2 | **high** | Re-derive the boundary for the target runtime. An async port awaits a channel; it does not transliterate a timer |
| Daemon-thread abandonment at interpreter exit | protocols H3, D2.3 (**proven**) | **high** | A probe showed the cleanup `finally` does not run. Explicit cancellation token + joined shutdown. Never model "quit" as "abandon" |
| A **local** encode error raised inside the **remote** error wrapper | protocols H14, D2.10 | medium | Classify encode-time failures separately (convention C03). Today a missing file reads as a server failure with the model name attached |
| No job id, sequence number, or timestamp in any event | protocols H5 | medium | Extend the schema **before** adding concurrency, cancellation, or resume. See Decision 3 |
| Unbounded queue as the only backpressure mechanism | protocols H4, D2.5 | medium | Bound it deliberately and document that the worker now blocks |
| LF normalization on every platform | protocols H6 | medium | Deliberate and **verified**. A Windows port must not "fix" this to CRLF |
| `os.replace` requires same-filesystem source and destination | protocols H7 | medium | Keep staging in the output directory; system temp silently loses atomicity |
| Defensive `getattr` reads mask provider drift | protocols H8, S5.1 | medium | Fields currently match (measured), so unrealized. Distinguish absent-optional from unrecognized-shape; fail loudly on the latter |
| Untrusted native parsing, PyMuPDF uncapped, no size cap | S4.1, D6.2 | medium | Pin parsers, add input size/page-count limits, isolate parsing if feasible |
| **A guard placed after the dependency it guards** | D1.6 | medium | The GUI suites' skip guard sits below `import app`. In a port, guard imports with a capability check, not a `try` in setup |
| Inconsistent payload strictness; `stream_chunk` omits `total` | protocols H9/H10, S5.7 | low | One schema, one validation point at the boundary |
| Zero-padded `page_%04d` overflows past 9,999 pages | protocols H11 | low | Cosmetic; consumed list order stays correct |
| `explorer /select,{path}` comma convention | protocols H13, D2.8 | low | Reproduce exactly; safe (argv, no shell) but Explorer-specific |
| Tk clipboard ownership dies with the process on X11 | D6.6 | low | Platform clipboard API with ownership transfer |

Not applicable: ANSI/terminal semantics, IME and cursor positioning, OAuth refresh, shell quoting.

## Defect Synthesis

**41 findings: 0 critical, 4 high, 16 medium, 21 low.** All high and medium below; the 21 low are grouped after.

| ID | Src | Description | Sev | Disposition | Required design consequence |
|---|---|---|---|---|---|
| D2.1 | mech | One failed page discards every page already recognized | high | **fix before porting** | Model the job as N independently retryable units: bounded retry with backoff per page; on unrecoverable failure still persist recognized pages with an explicit marker. **Test:** fail page 3 of 5 → 1,2,4,5 present, 3 marked |
| D2.2 | mech | A legitimately blank page is fatal | high | **fix before porting** | An empty page MUST yield an empty section, not an error. Reserve failure for transport-level errors. **Test:** blank page mid-document → run completes, that page contributes no text |
| D2.3 | mech | Shutdown neither joins nor cancels; cleanup `finally` **proven** not to run | high | **fix before porting** | Cancellation token checked between pages and chunks; shutdown signals, joins with timeout, owns cleanup. **Test:** quit mid-job → no temp directory survives, no output appears afterward (acceptance row 41) |
| S3.1 | sem | No overall job timeout; a slow-but-live stream hangs forever with no cancel | high | **fix before porting** | Per-page wall-clock ceiling **in addition to** the idle timeout. **Test:** stall a page past the ceiling → that page fails distinctly, job continues per D2.1 |
| D1.1 | mech | Progress reports a page done when it has only started | medium | **port differently** | Compute from completed work. See **Decision 2** — rewrites a GUI test that may never have run |
| D1.6 | mech | Headless-skip guard sits after the import it guards; 5 suites error | medium | **fix before porting** | Guard imports with a capability check (`find_spec`) or move the import inside the guarded block. **Test:** acceptance row 42 — a headless run reports pass/skip, never collection errors |
| D2.4 | mech | No `fsync` before rename; `delete=False` temp in the user's folder | medium | **fix before porting** | `fsync` before close; sweep stale `.{stem}_*.tmp` siblings. Makes the README's atomicity claim true (S5.3) |
| D2.5 | mech | Unbounded queue; per-page PNG bytes never evicted | medium | **fix before porting** | Bounded channel (real backpressure) + LRU or spill-to-disk for page bytes |
| D2.6 | mech | An unguarded handler drops queued events; a lost terminal event strands the UI | medium | **fix before porting** | Wrap each dispatch individually; log and continue. Guaranteed delivery for the four state-bearing events |
| D6.1 | mech | No config file, no env vars, no persistence of URL/model/DPI | medium | **port differently** | User config file + persisted last-used settings. Make the prompts reachable |
| D6.2 | mech | PyMuPDF and ollama uncapped; no lockfile | medium | **fix before porting** | Pin and lock all dependencies. Measured currently-working, which is a reason to pin, not to relax |
| D6.3 | mech | Whole document rasterized up front; no free-space check | medium | **fix before porting** | Stream rendering one page ahead; delete each PNG after use. Bounds temp to a constant and shrinks D2.3's blast radius |
| S3.2 | sem | Unbounded drain loop runs image decodes on the UI thread | medium | **fix before porting** | Decode off the UI thread, or cap events per drain pass and reschedule |
| S3.3 | sem | The worker can publish output *after* the window is destroyed | medium | **fix before porting** | Same fix as D2.3 — quitting must mean quitting |
| S4.1 | sem | Untrusted native parsing is the real attack surface; no size cap | medium | **port differently** | Input size and page-count ceilings before parsing; pinned, tracked parsers |
| S4.2 | sem | Overwrite check happens minutes before the write; guard silently defeated | medium | **fix before porting** | Re-check immediately before publish, or reserve the name with `O_EXCL` at start |
| S4.3 | sem | Full page content base64'd over unauthenticated plaintext; no guardrail | medium | **port differently** | Warn or require confirmation on a non-loopback host; allow requiring TLS |
| S4.4 | sem | No bound on accumulated model output across three buffers | medium | **fix before porting** | Per-page and per-document output caps with a clear failure message |
| S5.1 | sem | Provider field rename → total failure reported as "no vision support" | medium | **fix before porting** | Distinguish absent-optional from unrecognized-shape; fail loudly with the real cause |
| S5.2 | sem | "Returned no text" documented as one cause; it has four | medium | **fix before porting** | Fixed upstream by D2.2, S5.1, and C03; the doc must then describe all remaining causes |

**The 21 low findings, grouped.** *Fix before porting:* S3.5 (eager page retention — same fix as D6.3), S5.3 (atomicity prose — resolved by D2.4). *Port differently:* D1.3/S5.6 (controls enabled but inert — derive enablement from one state→control map), D1.4 (clear images explicitly, not via a null image), D1.5 (track the current page **number**, not a list position — becomes live the moment pages recognize concurrently), D2.7 (report cleanup failures), D2.8 (verify the path before the Windows reveal), D2.9 (write a real log file), D2.10 (classify local vs remote errors — convention C03), D6.4 (assert the runtime version), D6.6 (platform clipboard with ownership transfer), S3.4 (own the provider client's lifetime), S4.6 (never render HTML in an in-app preview), S5.4/S5.7 (one typed event schema, one validation point). *Leave behind:* D1.2 and S5.5 (the unreachable non-streaming fallback and the docstring describing it), D6.5 (placeholder model tags), D6.7 (`xdg-open` fallback), S4.5 (input-path TOCTOU — no privilege boundary to cross).

### Four synthesis decisions

**Decision 1 — Break the all-or-nothing contract, deliberately (closes `contracts-CF1`).**
Two **passing** service-suite tests assert that a mid-run failure discards all recognized pages and preserves any prior output. The right move is to **split the invariant they conflate**:

- **PRESERVE, as a hard rule:** *never leave a corrupt, partial, or truncated file at the destination, and never destroy an existing output except on a successful complete write.* This is the valuable half; both tests should keep asserting it.
- **DROP:** *discard all completed work when any page fails.* This protects nothing — it is a side effect of saving once at the end.

A port satisfies the preserved half by writing recognized pages to a **new, clearly-named partial artifact** rather than overwriting the canonical output on failure. `test_late_page_failure_preserves_existing_output` then still passes unchanged; `test_late_page_failure_leaves_no_new_output` must be reworded to assert "no *canonical* output was created." A deliberate, documented contract change (`strong inference`).

**Decision 2 — Fix the progress formula and rewrite the test (closes `contracts-CF2`).**
`test_ocr_phase_without_render` asserts the bar reads 1.0 the instant a single-image job starts. Nothing external depends on the formula, it is not a compatibility surface, and the behavior makes a working app look hung. Compute from **completed** work and rewrite the three progress tests. The cost is lower than it first appeared: these are **GUI-suite tests that error at collection headlessly**, so they are very likely not currently passing anywhere in CI — rewriting them forfeits little.

**Decision 3 — Extend the event schema and the state machine together (closes `protocols-CF1`).**
Cancellation, per-page retry (D2.1), concurrent recognition, and resume are **one design change, not four**, because each needs the same missing primitives. Before implementing any of them, add: a **job identifier** on every event; a **monotonic sequence number** for ordering independent of arrival; a **CANCELLING** state between PROCESSING and IDLE; and an **error state** (or an outcome field on IDLE) so the machine remembers the last result and can offer retry. Adding retry without the sequence number, or cancellation without the CANCELLING state, produces exactly the race class this system currently avoids only by being strictly sequential (`strong inference`).

**Decision 4 — Carry the two known-failing scenarios as normative rules (closes `contracts-CF4`).**
Of the 42 acceptance scenarios, **exactly two describe behavior the current implementation demonstrably fails**, and both were found by execution rather than reading:

- **Row 41** — quitting mid-run must not leak the render directory. Proven failing by the daemon-thread probe (D2.3).
- **Row 42** — a headless test run must report pass/skip, never collection errors. Measured failing (D1.6).

These must appear in the spec as **MUST rules with their own scenarios**, not as observations, because a faithful port would otherwise reproduce both. Row 42 is unusual in that it constrains the *port's own test harness* rather than its runtime behavior — keep it anyway: the defect it encodes is what hid ~470 lines of unverified contracts.

## Observed Facts vs. Inferred Structure

### Observed Facts

- Four first-party modules, 1,178 lines, strictly acyclic: `main` → `app` → `ocr_service` → `config`. `config.py` contains no `import` statement.
- `ocr_service.py` imports no Tk symbol — **demonstrated** by running its 1,069-line suite on a host with no `tkinter`: 72 passed, 1 skipped, 0 failures, on PyMuPDF 1.28.2 / Pillow 11.3.0.
- The five GUI suites **error at collection** without `tkinter` (5 errors), because the skip guard sits after `import app`.
- A daemon thread's `finally` **does not run** at interpreter exit; a `mkdtemp` directory survives (probe, Python 3.11).
- Page images reach the provider as **base64 file bytes in the JSON body**; `Image(value=str(path)).model_dump()` returns a base64 `str` and the path is absent.
- The installed `ollama` client still exposes `ChatResponse.message.content` and `ListResponse.models[].model`.
- Exactly one cross-thread object: an unbounded `queue.Queue`. No `Lock`, `Event`, `Semaphore`, or `Condition` exists anywhere.
- Every `operation_state` and `closing` access is on the Tk main thread (exhaustive site enumeration).
- One `stream=True` chat request per page, no cross-page context; `stream=True` is hardcoded with no non-streaming path.
- Output is UTF-8, LF-only, pages joined `"\n\n"`, each stripped, no header/footer/page markers.
- Password-protected and zero-page PDFs are rejected before any page renders; the document closes on every path.
- A thumbnail failure is logged and skipped; it never aborts recognition.
- No secrets in the tree; no `shell=True`; no `os.environ` read in first-party code.
- No config file, no env vars, no persisted preferences, no CI, no packaging metadata, no lockfile.

### Inferred Structure

- **The Tk boundary is the load-bearing architectural line** (`strong inference` — the docstring plus the demonstrated headless run). Not documented as an architectural decision anywhere, yet everything testable depends on it. And it is enforced only by convention, which is precisely how the *test* suites lost it (D1.6).
- **The system has no notion of partial success** (`strong inference` — save-at-end sequencing plus two passing tests asserting the discard). This one decision generates D2.1, D2.2, and half of D2.3's cost.
- **Absent bounds are systemic, not incidental** (`strong inference` across D2.5, D6.3, S3.1, S3.2, S4.4): no job timeout, output cap, input cap, queue bound, per-drain budget, or free-space check.
- **Errors misdirect as a pattern** (`strong inference` across S5.1, S5.2, S5.3, D2.10): defensive reads and confident prose combine so the app asserts causes it has not established.
- **The prompt pair is the highest-leverage and least accessible setting** (`strong inference` — verbatim test assertion plus no override path).
- **The event protocol is safe only because one operation runs at a time** (`strong inference` — no identifier or sequence number exists).

## Domain Glossary

| Term | Definition | Where Used |
|---|---|---|
| **Page** | The atomic unit of work — one rendered image, one independent model request. 1-based. An image input is page 1 of 1 | Everywhere; the only identifier in the event protocol |
| **DPI** | Rasterization density for PDF pages (100/150/200/300, default 150). **PDFs only** | `config.DPI_OPTIONS`, `render_pdf` |
| **Model tag** | An Ollama model identifier such as `gemma4:12b`. Free-text; never validated client-side; never auto-pulled | Model combobox, `list_models`, chat requests |
| **Extraction** | The output Markdown file, always `<input stem>_extracted.md` beside the input | `build_output_path`, overwrite prompt |
| **Delta / chunk** | One incremental text fragment from a streaming response, concatenated into the page text | `recognize_images`, `stream_chunk` |
| **Terminal event** | The single `ocr_success` or `ocr_error` ending a run, emitted after cleanup | `_ocr_worker`, SM1 |
| **Observational event** | An event that drives UI only and cannot affect the saved artifact | Protocols §B1 |
| **Render phase / OCR phase** | The two stages of a PDF job, mapped to progress `[0, 0.2]` and `[0.2, 1.0]`. An image job has no render phase | `on_progress`, `_render_phase_seen` |
| **Idle timeout** | The 120 s limit on *gaps between chunks*, not total response time — so a slow page never trips it | `config.OCR_STREAM_IDLE_TIMEOUT` |
| **Review** | Post-recognition spot-checking UI pairing each page's thumbnail with its text | Review tab, SM4 |
| **Asserted-not-verified** | A contract pinned by a test that is not known to execute — here, the ~470 lines of GUI suites | Contracts §Verification status; conventions C02/C04 |

## Coverage and limits

- **Inspected scope:** all five upstream artifacts read in full and synthesized; all 41 defect findings assigned a disposition and a design consequence; all four routed carry-forwards resolved with explicit decisions. Source and tests were re-consulted only to confirm the four decisions — no new source reading was required, which is the compression boundary working as intended.
- **Skipped scope:** the 42 acceptance scenarios and the full 9-event payload / 22-transition tables are **deliberately not reproduced** — named in the Source Index with explicit triggers. Per-finding evidence prose stays in the two scan reports. No new defect analysis was performed; this phase synthesizes.
- **Secondary outputs — explicit accounting** (five declared): `public-surfaces`, `runtime-lifecycle`, and `state-and-storage` were **appended** by architecture, contracts, and protocols and need no porting-level addition — the synthesis view lives in this bundle, which is the compression boundary those files would otherwise duplicate. `build-and-deploy` and `config-model` were **deliberately not written** across the whole run (trivial build surface; flat constant config with no propagation protocol), as recorded in `arch-D1`/`arch-D2`, `contracts-D2`, and `protocols-D1`. Recorded here as `porting-D5`.
- **Evidence basis:** upstream findings (primary), including **runtime verification** gathered in earlier phases (service-suite execution, the daemon-thread probe, client serializer inspection and type introspection); plus targeted source and test confirmation for the four decisions.
- **Known blind spots:** (1) **no live model was ever contacted and no GUI was ever launched** — output quality is unmeasured and the ~470 lines of UI contracts remain unverified, so 14 of the 42 acceptance rows carry weak evidence; (2) S5.1's trigger is a future rename — today's client is measured compatible, so which future version breaks is unknown; (3) performance and memory figures are analytical estimates, never measured; (4) `q-queue-depth-under-load` and `q-ctk-image-clear` remain open and need a model and a `tkinter` build respectively; (5) gitignored `build-mac.sh` and `docs/` could contain packaging or design context invisible to the whole pipeline.
- **Coverage disposition:** COMPLETE for the porting scope. Every template section is populated, every defect carries a disposition and an acceptance-test implication, and the Source Index makes the bundle self-sufficient with two named exceptions.

## Open Questions

Five remain open pipeline-wide. Two are material to the reimplementation and restated; three (`q-ctk-image-clear`, `q-queue-depth-under-load`, `q-test-invocation`) describe source-implementation behavior a port will not reproduce and are resolved by the dispositions above.

| ID | Kind | Description | Deferred Reason |
|---|---|---|---|
| `q-gui-tests-ever-run` | needs-maintainer-decision | It is established that the five GUI suites error at collection without `tkinter`, so they cannot have passed headless. Whether they pass on a machine **with** `tkinter` and a display — and therefore whether ~470 lines of UI contracts have ever been demonstrated anywhere — is unknown. | Needs a maintainer statement or a host with a display. Bears directly on how far the acceptance harness can be trusted, which is why D1.6 is dispositioned `fix before porting`. |
| `q-prompt-provenance` | needs-maintainer-decision | `SYSTEM_PROMPT` drives output quality more than anything else and is asserted verbatim by a passing test, but nothing records how it was derived or against which models it was tuned. A port cannot tell which clauses are load-bearing. | Project history, not source. Unresolvable by any phase's rubric. |

## Carry-Forward

| ID | Target Phase | Description | Deferred Reason |
|---|---|---|---|
| `porting-CF1` | reimplementation-spec | The 4 high and 16 medium defects each carry a **required design consequence** and, for the high ones, a sketched acceptance test in §Defect Synthesis. Each must become a normative rule plus a concrete black-box scenario in the spec, not a summary. | The deep-audit guidance is explicit that a hazard with no test in the spec gets reintroduced by whoever implements it. Turning consequences into normative rules with scenarios is the spec's rubric. |
| `porting-CF2` | reimplementation-spec | Decisions 1–4 are decided but not specified. Each needs exact normative wording, the enumerated test changes, and its placement in the build order. | The decisions belong to synthesis; their precise wording, test-change lists, and sequencing belong to the spec. |
| `porting-CF3` | reimplementation-spec | This bundle omits the 42 acceptance scenarios and the full event/state tables, pointing at them instead. The spec must fold the acceptance list into its own acceptance section **carrying the verification-status column forward** (convention C04) and pin the event contract concretely. | Named in the Source Index as the two deep-read triggers. Reproducing them here would defeat the compression boundary; the spec is where they become normative — and dropping the status column would hand the implementer a harness whose weak rows look like its strong ones. |

---

## Validation

| # | Criterion | Result | Evidence |
|---|-----------|--------|----------|
| 1 | The system summary, layer map, contract table, protocol notes, and porting findings are synthesized. | PASS | §System Summary (3 paragraphs plus the two-tier-evidence caveat); §Layer Map With Ownership (4 concept-named layers with the Tk-boundary invariant); §Feature Contract Table (19 features with contracts and defect references); §Protocol and State Notes (event contract, three-way split, SM1 properties, resolved wire encoding, persistence, verified concurrency). |
| 2 | Portability hazards and open questions are separated from facts. | PASS | §Portability Hazards is a distinct 15-row table with impact and mitigation; §Open Questions is separate; §Observed Facts vs. Inferred Structure splits the two, with every inference carrying its derivation. |
| 3 | Feature importance is sorted for porting. | PASS | §Feature Contract Table sorts all 19 features core → important → optional → incidental (7 core, 7 important, 3 optional, 2 incidental). |
| 4 | Defect Synthesis consolidates mechanical-defects.md and semantic-defects.md with porting recommendations. | PASS | Draws from both reports (D- and S-prefixed IDs), tabulates all 4 high and 16 medium with disposition **and** a required design consequence (acceptance-test sketches for the high ones), and groups all 21 low findings by disposition. Totals reconcile to 41. |
| 5 | Findings are marked with evidence levels. | PASS | `observed fact`, `strong inference`, `portability hazard` (the hazards table), and `open question` used throughout; the `asserted-not-verified` tier from C02/C04 is carried into the contract table. §Observed Facts vs. Inferred Structure is the explicit separation. |
| 6 | Coverage and limits name inspected scope, skipped scope, evidence basis, and blind spots. | PASS | All four named plus a COMPLETE disposition; **all five declared secondary outputs accounted for** (`porting-D5`); five blind spots listed, led by the absence of any live model or GUI run. |
| 7 | The Source Index makes the bundle a self-contained compression boundary and identifies targeted deep-read triggers. | PASS | §Source Index maps 9 upstream areas to canonical sections, what is carried forward, and a specific trigger. The two genuine omissions (42 acceptance scenarios; full event/state tables) are named with triggers and routed as `porting-CF3`. Every load-bearing invariant, hazard, and disposition is carried inline. |

**Validated by:** 2026-08-18 (porting phase, MCP-driven session 2, framework v0.16.0)
**Overall:** PASS
