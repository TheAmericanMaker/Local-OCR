# Architecture Map

Target: `Local-OCR` (repository root, `../` relative to `.codecarto/`)
Commit analyzed: `8e7388c` · Framework: CodeCartographer v0.16.0

## System Intent

Local OCR is a single-user, privacy-focused **desktop GUI application** that converts one PDF or image at a time into structured Markdown by delegating recognition to a Vision-Language model served by [Ollama](https://ollama.com) (`observed fact` — `README.md:1-8`). Its defining product constraint is that no document data leaves the user's machine or network: the only outbound egress is the Ollama base URL the user types, and the app never installs, pulls, or starts a model itself (`observed fact` — `README.md:5-8,41-46`; the only network clients are built at `ocr_service.py:88` and `ocr_service.py:343`). The audience is someone who has already provisioned Ollama and a vision-capable model and wants document-to-Markdown conversion without a cloud OCR service (`strong inference`).

The unit of work is deliberately narrow: **one input file per run**, rendered and recognized page-by-page in document order, then concatenated and written beside the source as `<stem>_extracted.md` (`observed fact` — `ocr_service.py:82`, `352-362`).

## Layer Map

Four first-party modules at the repository root form a strict four-tier stack. No packages, no `src/`, no namespacing (`observed fact`).

- **`config.py`** — *core semantics (constants only)*. Tunables, extension allowlists, timeouts, and the two model prompts. **Contains no `import` statement at all**; everything else imports it. The stable base (`observed fact` — `config.py:1-44`).
- **`ocr_service.py`** — *integration adapter + core semantics*. Validation, URL normalization, PDF rasterization, thumbnailing, streaming recognition, atomic save, OS file-manager integration. **Deliberately Tk-free**, stated as a contract in its docstring: "This module must stay free of Tk imports so every function can be tested headlessly and no worker can accidentally touch the GUI" (`observed fact` — `ocr_service.py:1-5`).
- **`app.py`** — *UI or rendering + product shell*. One `LocalOCRApp(ctk.CTk)` class holding every widget, all mutable UI state, the worker launchers, and the main-thread event pump (`observed fact` — `app.py:8-25`).
- **`main.py`** — *product shell*. 16 lines: dark appearance, blue theme, instantiate, `mainloop()`.

### Package Inventory

| Package / Module | Role | Public Entrypoints | Key Dependencies | Runtime Surface |
|---|---|---|---|---|
| `main.py` | product shell | `main()`, `__main__` guard | `customtkinter`, `app` | Process entrypoint (`python main.py`) |
| `app.py` | UI or rendering / product shell | `LocalOCRApp`, `OperationState` | `config`, `ocr_service`, `customtkinter`, `PIL.Image`, `tkinter.filedialog`, `tkinter.messagebox` | Tk main thread; all widgets |
| `ocr_service.py` | integration adapter / core semantics | `OCRRequest`, `OCRServiceError`, `normalize_ollama_url`, `validate_input_path`, `build_output_path`, `list_models`, `render_pdf`, `make_thumbnail_png`, `recognize_images`, `save_markdown_atomic`, `open_in_default_app`, `reveal_in_file_manager`, `process_ocr` | `config`, `ollama`, `pymupdf`, `PIL.Image` (lazy) | Worker threads; HTTP; filesystem; subprocess |
| `config.py` | core semantics (constants) | module-level constants | *(none)* | Import-time only |
| `tests/` | test harness | `unittest` cases | `unittest`, `unittest.mock`, modules above | Dev only; not shipped |

### Dependency Direction

```
main.py  ──►  app.py  ──►  ocr_service.py  ──►  config.py
   │            │               │                  ▲
   └────────────┴───────────────┴──────────────────┘
                (all three import config)
```

- **Stable base:** `config.py`. Imports nothing; imported by everything (`observed fact`).
- **No cycles.** `ocr_service` never imports `app`; the arrow is strictly downward (`observed fact` — verified against `app.py:8-25` and `ocr_service.py:7-23`).
- **The Tk boundary is the load-bearing architectural line, and it holds — verified by execution.** This run installed only `Pillow`, `PyMuPDF`, and `ollama` on a host **with no `tkinter` module at all**, and ran the full service suite: **72 tests passed, 1 skipped, 0 failures** (`observed fact` — `python3 -m unittest tests.test_ocr_service`, this session). The boundary is therefore not merely asserted in a docstring; the 1,069-line service suite genuinely executes with the GUI toolkit absent. Note the rule is still enforced only by convention — no import lint, no test asserts it — so it can regress silently.
- **No module is a wrapper around shared internals**; each of the four has a distinct role. No plugin architecture, no service boundaries, no shared provider layer, and only one delivery surface (`observed fact`).

`ocr_service` reaches back toward the UI only through injected callables — `LogCallback`, `ProgressCallback`, `EventCallback` (`ocr_service.py:25-27`) — and the `event_queue` parameter of `process_ocr`, which carries **no type annotation**, making that contract structural (`observed fact` — `ocr_service.py:302`).

## Public Surfaces

No CLI beyond `python main.py` (no `argparse`, no `sys.argv` read), no network listener, no exported library API, no IPC (`observed fact`).

**1. GUI workflows** (primary surface — `app.py:85-243`):

| Control | Handler | Effect |
|---|---|---|
| `Select File` | `select_file` (`app.py:460`) | Filtered open dialog; validates and stores selection |
| Ollama server URL entry | read by `refresh_models` / `start_ocr` | Default `http://localhost:11434` |
| `Refresh Models` | `refresh_models` (`app.py:482`) | Worker lists model tags |
| Model combobox | read by `start_ocr` | Free-text or picked; seeded with non-authoritative suggestions |
| PDF DPI combobox (`readonly`) | read by `start_ocr` | One of 100/150/200/300 |
| `Start OCR` | `start_ocr` (`app.py:595`) | Validates all inputs, launches worker |
| Log / Result / Review tabs | `tabview` (`app.py:183-188`) | Status / live Markdown / per-page image↔text |
| `Copy` | `copy_result` (`app.py:571`) | Result panel to clipboard |
| `◀` / `▶` | `review_prev` / `review_next` | Step completed pages |
| Completion dialog: Open / reveal / OK | `_show_completion_dialog` (`app.py:687`) | Launch / reveal / dismiss |
| Window close | `on_close` (`app.py:743`) | Confirms if an operation is running |

**2. Outbound Ollama HTTP** (`observed fact`): `client.list()` (timeout 10 s, `ocr_service.py:88`); `client.chat(..., stream=True)` once per page (timeout 120 s applied to inter-chunk gaps, `ocr_service.py:197-208`). Response shapes are read via defensive `getattr` chains (`ocr_service.py:91`, `210-211`) rather than typed models. **Verified this session:** the installed `ollama` client still exposes exactly those shapes — `ChatResponse.message` → `Message` with a `content` field, and `ListResponse.models` → `Sequence[ListResponse.Model]` with a `model` field (`observed fact`, by introspection of `ollama._types`).

**3. File formats** (`observed fact`): inputs `.pdf`, `.png`, `.jpg`, `.jpeg`, `.webp` (`config.py:11-13`), matched case-insensitively (`ocr_service.py:71`). Output UTF-8 Markdown, LF-normalized, at `<input_dir>/<stem>_extracted.md`, pages joined `"\n\n"`. Transient: `local_ocr_*` temp dir with `page_0001.png`-style renders.

**4. OS integration** (`observed fact` — `ocr_service.py:266-299`): `open` / `open -R` on macOS, `os.startfile` / `explorer /select,` on Windows, `xdg-open` elsewhere. All argv vectors, never `shell=True`.

## Runtime Lifecycle

**Boot** (`observed fact` — `main.py:9-13`): appearance mode → color theme → construct `LocalOCRApp` → `mainloop()`. Construction sets `operation_state = IDLE`, creates the queue, builds all widgets eagerly, registers `WM_DELETE_WINDOW`, and schedules the first `drain_ui_events` tick (`app.py:50-81`). **No config read, no environment probing, no network call at startup** — the app is inert until the user acts (`observed fact`).

**Steady state:** the Tk event loop plus one self-rescheduling timer. `drain_ui_events` runs every `UI_POLL_INTERVAL_MS = 50ms`, drains the queue non-blockingly until empty, and dispatches each `(kind, payload)` to `handle_event` (`app.py:247-285`). The reschedule sits in a `finally`, so a raising handler cannot break the pump — deliberate and commented (`observed fact` — `app.py:257-261`). Unknown kinds surface as `[Warn]` rather than being dropped (`app.py:282-285`).

**Job lifecycle:** `start_ocr` validates everything on the main thread and snapshots it into a frozen `OCRRequest` before any thread starts (`app.py:598-649`) — this is what keeps workers from reading mutable widget state (`strong inference`). Then `_apply_ocr_busy_state` disables controls and clears panels, and a **daemon** thread runs `_ocr_worker`. Inside, `process_ocr` walks `[1/3]` prepare, `[2/3]` recognize, `[3/3]` save (`ocr_service.py:322-363`). Exactly one terminal event is enqueued by the wrapper **after** `process_ocr` returns or raises, so temp-dir cleanup always precedes it — an invariant documented in both docstrings (`observed fact` — `ocr_service.py:302-310`, `app.py:658-670`).

**Shutdown** (`observed fact` — `app.py:743-752`): idle (or already closing) → immediate `destroy()`; otherwise a yes/no confirmation gates it. Workers are daemon threads, so an in-flight job is abandoned rather than joined. `portability hazard` / `open question`: nothing joins or signals the worker, so `process_ocr`'s `finally` may not run on a mid-job quit and the temp directory can survive. Recorded as `q-shutdown-tempdir`; routed to the mechanical scan as `arch-CF1`.

**Background work:** only the 50 ms drain tick and the 100 ms one-shot `after()` used to throttle stream flushes (`app.py:561-569`). **No scheduler, no retry loop, and no cancellation path** — once `Start OCR` is pressed the job runs to completion or failure (`observed fact` — no cancel control exists in `_build_layout`).

## Concurrency Model

**Threading model:** Tk main thread + at most one short-lived **daemon** worker, mutual exclusion by the `OperationState` enum (`IDLE` / `REFRESHING_MODELS` / `PROCESSING_OCR`) rather than by a lock (`observed fact` — `app.py:28-31`). Every work-starting entrypoint returns early unless IDLE (`app.py:461`, `483`, `596`). Safe **only because all three checks run on the Tk main thread**, which serializes them (`strong inference`).

**Cross-thread channel:** exactly one — a `queue.Queue` of `(kind, payload)` tuples (`app.py:59`). Workers only `put`; the main thread only `get_nowait`. The contract is stated at the top of `app.py`: "workers never touch Tk" (`observed fact` — `app.py:3-6`). Payloads are plain values — `str`, `dict`, `list[str]`, `bytes` — never widgets (`observed fact`, every `put` site).

**Shared mutable state:** the queue is the only object both sides touch. Everything else (`review_pages`, `_review_order`, `_review_image_cache`, `_stream_buffer`, `_result_page`, `_render_phase_seen`, `operation_state`, `selected_path`) is main-thread-only (`strong inference` — workers hold only the frozen request and the queue).

**Synchronization primitives:** none beyond `queue.Queue`'s internal lock. No `Lock`, `Event`, `Semaphore`, or `Condition` anywhere (`observed fact`). No connection pool — a fresh `ollama.Client` per operation.

**Backpressure:** the queue is **unbounded**, so a fast-streaming model can outpace the 50 ms drain without limit (`observed fact`; consequence `strong inference`). Two mechanisms mitigate UI cost, not memory: stream deltas coalesce into `_stream_buffer` flushed at most every 100 ms (`app.py:550-569`), and decoded review images sit in a 5-entry LRU (`app.py:361-378`). The LRU bounds *decoded* images only — raw PNG bytes for every page accumulate in `review_pages` for the life of the run with no eviction (`observed fact` — `app.py:313-315`).

**Performance-critical paths** (`strong inference`): (a) per-page inference dominates wall-clock and is server-side; (b) `render_pdf` rasterizes **every** page before any recognition (`ocr_service.py:121-137`), so a large PDF pays full render latency and temp-disk cost before the first token; (c) PNG decode and `CTkImage` construction happen **on the Tk main thread** (`app.py:317-324`, `368-374`).

**Portability hazards** (all `portability hazard`): daemon-thread abandonment at interpreter exit has no equivalent in most target runtimes; Tk's thread-affinity rule is the reason the queue exists; `after()`-based 50 ms polling is a Tk idiom an async port should replace with awaited channel reads; the GIL makes `_stream_buffer` concatenation incidentally atomic, but it runs main-thread-only so nothing relies on it.

## Build and Packaging

Minimal to the point of absence (`observed fact` — no `pyproject.toml`, `setup.py`, `setup.cfg`, `Makefile`, `Dockerfile`, `tox.ini`, or `.github/`):

- **Dependencies:** `requirements.txt`, four entries — `customtkinter>=6.0.0,<7`, `Pillow>=10,<12`, `PyMuPDF>=1.24.0`, `ollama>=0.4.0`. Note two are capped and two are not.
- **Install and run:** venv + `pip install -r requirements.txt` + `python main.py` (`README.md:18-32`).
- **Build artifacts:** none. Run from source; no wheel, binary, or container.
- **CI/CD:** none visible.
- **Tests:** 6 `unittest` files, ~1,540 lines against ~1,180 lines of source. **Executed this session:** `python3 -m unittest tests.test_ocr_service` → **72 passed, 1 skipped** (the skip is `test_unreadable_file_rejected`, which self-skips as root because `chmod 0` does not block root reads). Ran in 0.52 s against **PyMuPDF 1.28.2 and Pillow 11.3.0** — far newer than the declared floor, so the uncapped range has not broken yet (`observed fact`).
- **Test invocation:** `python3 -m unittest tests.test_ocr_service` works; so does `unittest discover` for the service module. `q-test-invocation` is **re-triaged and narrowed** — what *works* is now observed fact; only the project's preferred runner remains unstated (`.gitignore` mentions `.pytest_cache/`, the tests are `unittest`-based).
- **Platform packaging:** `.gitignore` lists `build-mac.sh`, so a macOS packaging script exists **outside** version control (`observed fact` — `.gitignore:31`). Also gitignored and invisible here: `docs/`, `CLAUDE.md`, `AGENTS.md`, `.claude/`.
- **Version constraint of note:** the README pins customtkinter 6.x because "older 5.2.x renders blank windows under Tk 9.0 on macOS" (`observed fact` — `README.md:11-14`).

The build surface is four lines of `requirements.txt` and one command, so the secondary output `findings/build-and-deploy/build-and-deploy.md` was **not** written — the skill directs using it only when the pipeline is complex. Accounted for in Coverage and limits.

## Porting Priorities

| Component | Priority | Rationale |
|---|---|---|
| `recognize_images` per-page streaming loop | **core** | The product *is* this loop: one-request-per-page, stream-delta surfacing, empty-result rejection, error wrapping |
| `render_pdf` (PDF → PNG at DPI) | **core** | Without rasterization the PDF path does not exist; PyMuPDF-specific |
| `config.SYSTEM_PROMPT` / `USER_PROMPT` | **core** | Output quality is a direct function of these strings. Port verbatim; treat as behavioral |
| Worker→UI event contract | **core** | The seam between compute and presentation; wire shapes are the protocols phase's rubric |
| `save_markdown_atomic` | **core** | Temp-then-rename is the stated durability guarantee (`README.md:82-84`) |
| `normalize_ollama_url` / `validate_input_path` | **important** | The complete input-validation contract; each message is user-visible and test-asserted |
| `OperationState` exclusion + busy/idle gating | **important** | Prevents concurrent jobs and inconsistent widget state. Mechanism is Tk-specific; the invariant is not |
| Progress fraction model | **important** | Directly user-visible; the `_render_phase_seen` branch is easy to get subtly wrong (`app.py:510-517`) |
| Result streaming + `_result_page` de-duplication | **important** | Non-obvious interaction preventing double-written pages (`app.py:526-563`) |
| Overwrite-confirmation prompt | **important** | Data-loss guard on an existing extraction |
| Review tab (pairing, bisect nav, LRU) | optional | Real QA value, but conversion works without it |
| Live preview thumbnail | optional | Feedback affordance |
| Completion dialog with Open / reveal | optional | OS glue |
| `Copy` button | incidental | One clipboard call |
| Dark theme / geometry / `Courier New` | incidental | customtkinter ergonomics; no behavioral content |
| `EXAMPLE_MODELS` suggestions | incidental | Explicitly non-authoritative placeholders (`config.py:5-7`) |

## Durable State

| Kind | Present? | Detail |
|---|---|---|
| Config files | **No** | All configuration is hardcoded in `config.py`; nothing read from disk at startup (`observed fact`) |
| Environment variables | **No** | No `os.environ` / `os.getenv` in first-party code (`observed fact`) |
| Auth material | **No** | Ollama has no built-in auth and the app sends no credentials; README flags this (`README.md:48-53`) |
| Session / preferences | **No** | URL, model, and DPI are **not** persisted; every launch resets to `DEFAULT_OLLAMA_URL`, empty model, DPI 150 (`app.py:109`, `122`, `136`) |
| Logs | **In-memory only** | The Log tab is a Tk textbox; nothing written to a file (`observed fact`) |
| Caches | **In-memory only** | 5-entry LRU of decoded images; raw page PNG bytes retained for the run |
| Databases | **No** | None |
| Generated artifacts | **Yes** | `<stem>_extracted.md` beside the input — the single durable output |
| Temporary files | **Yes** | `local_ocr_*` mkdtemp with `page_NNNN.png`, removed in `process_ocr`'s `finally` on every normal path (`ocr_service.py:364-366`) — but see `q-shutdown-tempdir`. Also the `.{stem}_*.tmp` staging file, unlinked on failure |

The configuration model is a single flat set of module constants with no file, no env vars, and no precedence chain, so the secondary output `findings/config-model/config-model.md` was **not** written; the table above and §Build and Packaging carry the whole picture. Accounted for in Coverage and limits.

## Coverage and limits

- **Inspected scope:** all four first-party modules read in full (`main.py` 16, `config.py` 44, `ocr_service.py` 366, `app.py` 752 — 1,178 lines); `README.md`; `requirements.txt`; `.gitignore`; the `tests/` inventory with class-level structure. **Additionally, and new to this run: the service test suite was executed** (72 passed / 1 skipped) and the installed `ollama` client's response types were introspected.
- **Skipped scope:** test *bodies* were not read line-by-line (class names and sizes only) — mining them as behavioral evidence is the contracts phase's rubric, routed as `arch-CF4`. Third-party internals (`customtkinter`, `pymupdf`, `PIL`) treated as shaping forces, not architecture, per the skill. Binary fixtures in `examples/` not opened. Files excluded by `.gitignore` (`build-mac.sh`, `docs/`) are absent from the clone.
- **Secondary outputs — explicit accounting** (all five declared by this phase):
  - `findings/public-surfaces/public-surfaces.md` — **written** (append mode); seeded from §Public Surfaces.
  - `findings/runtime-lifecycle/runtime-lifecycle.md` — **written**; seeded from §Runtime Lifecycle and §Concurrency Model.
  - `findings/state-and-storage/state-and-storage.md` — **written**; seeded from §Durable State.
  - `findings/build-and-deploy/build-and-deploy.md` — **deliberately not written.** The build surface is four dependency lines and one command, below the skill's complexity threshold; §Build and Packaging carries it in full, now including executed test results. Recorded as decision `arch-D1`.
  - `findings/config-model/config-model.md` — **deliberately not written.** The config model is one flat set of module constants with no file, env var, or precedence chain; §Durable State and §Build and Packaging carry it. Recorded as decision `arch-D2`.
- **Evidence basis:** source inspection; project documentation; **runtime verification** (service suite execution, dependency introspection) — the last is new to this run and upgrades three previously-inferred claims to observed fact.
- **Known blind spots:** (1) the **GUI half of the suite could not be executed** — this host has no `tkinter`, and the attempt revealed the suites *error at import* rather than skipping (routed as `arch-CF6`); (2) Ollama wire payloads were introspected at the client type level, not captured from a live server, so no end-to-end request was observed; (3) real behavior under a large PDF (memory, temp disk, queue depth) is modeled analytically, not measured; (4) `customtkinter` widget semantics are assumed from its API; (5) gitignored `build-mac.sh` and `docs/` could carry packaging or design context invisible to this pipeline.
- **Coverage disposition:** COMPLETE for architecture-phase purposes. Every first-party module was read in full, the dependency graph is closed, and the central structural claim (the Tk boundary) is now empirically verified rather than asserted.

## Open Questions

| ID | Kind | Description | Deferred Reason |
|---|---|---|---|
| `q-shutdown-tempdir` | needs-runtime-test | Quitting mid-OCR destroys the Tk root while a daemon worker is inside `process_ocr`. Whether the outer `finally` at `ocr_service.py:364` runs before interpreter exit — and therefore whether the `local_ocr_*` directory leaks — is not determinable by reading. | Requires a GUI session: launch, start a multi-page job, quit mid-run, inspect the temp directory. This host has no `tkinter`, so it cannot be tested here even though the service layer can. |
| `q-test-invocation` | needs-maintainer-decision | **Re-triaged and narrowed this run.** `python3 -m unittest tests.test_ocr_service` is now *confirmed working* (72 passed). What remains unknown is only the project's intended runner: `.gitignore` references `.pytest_cache/` while the tests are written against `unittest`, and no runner config is committed. | The residue is a maintainer convention, not a code fact — and it now blocks nothing, since a working command is established. |

## Carry-Forward

| ID | Target Phase | Description | Deferred Reason |
|---|---|---|---|
| `arch-CF1` | defect-scan-mechanical | Shutdown (`app.py:743-752`) neither joins nor signals the daemon OCR worker; `process_ocr`'s cleanup `finally` may be skipped at interpreter exit, and no cancellation mechanism exists at all. | Resource-cleanup and error-handling hazards are the mechanical scan's rubric (pass 2). Architecture records the structural fact; the scan assigns severity and action. |
| `arch-CF2` | defect-scan-mechanical | The unbounded `queue.Queue` (`app.py:59`) has no `maxsize` and no backpressure, and per-page PNG bytes in `review_pages` are never evicted, so memory growth is unbounded in document length. | Resource-exhaustion assessment with severity and a recommended bound belongs to the mechanical scan, not the structural map. |
| `arch-CF3` | protocols | The worker→UI event catalog (`log`, `progress`, `page_image`, `page_text`, `stream_chunk`, `models_loaded`, `refresh_error`, `ocr_success`, `ocr_error`) is enumerated by name and dispatch site, but payload schemas, ordering guarantees, and the `OperationState` transition table are not formalized. | Event catalogs, wire shapes, and state machines are precisely the protocols phase's rubric. |
| `arch-CF4` | contracts | Per-surface behavioral contracts — every validation message, the overwrite flow, defaults, and a black-box acceptance list — are visible but not recovered as contracts. The ~1,540 lines of tests are the richest evidence source and were deliberately not mined. | Trigger/defaults/side-effects/error-behavior recovery is the contracts phase's rubric, and its skill directs mining tests as behavioral evidence. |
| `arch-CF5` | defect-scan-semantic | PNG decode and `CTkImage` construction run on the Tk main thread (`app.py:317-324`, `368-374`), and `render_pdf` rasterizes all pages before recognition starts (`ocr_service.py:121-137`). Both are main-thread/latency concerns rather than pure logic bugs. | Concurrency and responsiveness analysis needs the state machine and event ordering from protocols in hand; the semantic pass runs after protocols for that reason. |
| `arch-CF6` | defect-scan-mechanical | **New this run, found by execution.** The five GUI test files document that they "are skipped automatically when no display is available (CI, headless containers)", and each guards `LocalOCRApp()` in `setUp` with a `skipTest` fallback. But `import app as app_module` sits at **module scope**, so on a host without `tkinter` the import raises `ModuleNotFoundError` before any skip logic runs: `unittest discover` reports **5 errors**, not 5 skips, while the service suite passes cleanly. | Whether a documented skip that actually errors is a defect, and at what severity, is the mechanical scan's rubric (pass 2, error handling / pass 1, dead-or-wrong guard). Architecture records the observation; the scan rates it. |

---

## Validation

| # | Criterion | Result | Evidence |
|---|-----------|--------|----------|
| 1 | The system intent is documented. | PASS | §System Intent — purpose, audience, privacy constraint, unit of work, each cited. |
| 2 | The layer map and dependency direction are documented. | PASS | §Layer Map with per-module roles, §Package Inventory (5 rows), §Dependency Direction with the graph, the stable base, an explicit no-cycles finding, and the Tk boundary **empirically verified by running 72 service tests with `tkinter` absent**. |
| 3 | Public surfaces are identified. | PASS | §Public Surfaces — 11-row GUI control table, outbound Ollama calls with timeouts **and client response types verified by introspection**, file formats, OS integration. Absence of CLI/listener/library API stated explicitly. |
| 4 | Runtime lifecycle, concurrency model, and porting priorities are summarized. | PASS | §Runtime Lifecycle (boot, steady state, job lifecycle, shutdown, background work), §Concurrency Model (threading, single channel, shared state, no locks, backpressure, hot paths, 4 hazards), §Porting Priorities (16 rows across four tiers). |
| 5 | Findings are marked with evidence levels. | PASS | Every substantive conclusion carries `observed fact`, `strong inference`, `portability hazard`, or `open question`, with file:line citations or an executed-command reference. |
| 6 | Coverage and limits name inspected scope, skipped scope, evidence basis, and blind spots. | PASS | §Coverage and limits names all four plus a COMPLETE disposition, and **explicitly accounts for all five declared secondary outputs** (three written, two deliberately not, with rationale). Gaps routed: 2 open questions and 6 carry-forwards, all mirrored in `scratch/handoffs/architecture.yaml`. |

**Validated by:** 2026-08-18 (architecture phase, MCP-driven session 2, framework v0.16.0)
**Overall:** PASS
