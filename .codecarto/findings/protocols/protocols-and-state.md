# Protocols and State — Local-OCR

Source: `../` (repository root), commit `8e7388c` · Framework v0.16.0 · Date 2026-08-18
Upstream: `architecture-map.md`, `mechanical-defects.md`, `behavioral-contracts.md`
**Closes `arch-CF3` and `contracts-CF3`.**

### Orchestrator duties discharged

**Open-question re-triage** (4 inherited). `q-test-invocation` and `q-prompt-provenance` (`needs-maintainer-decision`) — labels re-tested, **confirmed**; both are project-history questions no reading resolves. `q-ctk-image-clear` and `q-gui-tests-ever-run` (`needs-runtime-test` / `needs-maintainer-decision`) — **confirmed**; both need a host with `tkinter`, which C01 cannot reach here.

**Additionally, and per C01, this phase resolved a question the previous pass had left open as its single largest unknown** — how page images actually reach the server. It is answered below as `observed fact` (§B3), so no equivalent open question is carried.

**Contradiction sweep.** No measured fact contradicts a completed phase's owner_notes. The contracts phase's two-tier evidence framing is honored here: SM2/SM3/SM4 are pinned only by the GUI suites, so their transitions are marked `asserted-not-verified` rather than `observed fact`.

## Boundaries Identified

| # | Boundary | Kind | Carrier |
|---|---|---|---|
| B1 | Worker thread → Tk main thread | core to UI | `queue.Queue` of `(kind, payload)` tuples |
| B2 | `ocr_service` internals → its caller | core to UI, callback form | Injected callables (`LogCallback`, `ProgressCallback`, `EventCallback`) |
| B3 | `ocr_service` → Ollama server | core to provider | HTTP via the `ollama` client (`list`, streaming `chat`) |
| B4 | `ocr_service` → durable output | runtime to persistence | `NamedTemporaryFile` + `os.replace` |
| B5 | `render_pdf` → `recognize_images` | runtime to transient filesystem | PNG files in a `mkdtemp` directory, referenced by path |
| B6 | `app` → operating system | process to process | `subprocess` argv vectors / `os.startfile` |

**Not protocol boundaries:** process-to-process IPC (none), a tool layer (none), and configuration propagation — `config.py` is imported directly by every consumer with no serialization, precedence chain, or reload path, so nothing crosses (`observed fact`). Hence no `findings/config-model/config-model.md`; see Coverage and limits.

---

## Event Catalog

### B1 — The worker→UI event queue (the primary protocol)

| Field | Value |
|---|---|
| **Producer** | Three sites, never concurrently: `_refresh_worker` (`app.py:497-503`), `_ocr_worker` (`658-670`), and the three closures `process_ocr` installs (`ocr_service.py:312-319`). Mutual exclusion comes from `OperationState`, not from the queue. |
| **Consumer** | `drain_ui_events` → `handle_event` (`app.py:247-285`), exclusively on the Tk main thread. |
| **Transport** | In-process `queue.Queue`, **unbounded** (no `maxsize`), FIFO, thread-safe. Producers only `put`; the consumer only `get_nowait` until `queue.Empty` (`observed fact`). |
| **Ordering guarantees** | FIFO within the queue; because one producer thread runs at a time, FIFO is also global. Four higher-level orderings hold, **all verified by the passing service suite** (`observed fact`): (1) **per page** — `page_image`? → `log "Sending page n/N"` → `stream_chunk`* → `page_text`; (2) **per phase** — every `render` progress event precedes every `ocr` one; (3) **terminal** — exactly one `ocr_success` XOR `ocr_error`, last, enqueued *after* cleanup completes; (4) **page monotonicity** — 1..N, no gaps, no repeats. |
| **Required fields** | Every event is a 2-tuple `(kind: str, payload: Any)`. Per-kind schemas below. |
| **Optional fields** | None formally, but consumer strictness is **inconsistent**: `on_page_text` reads `payload.get("total", self._review_total)` (tolerant, `app.py:539`) while `on_page_image` reads `payload["total"]` (strict, `312`). Same key, two contracts (`observed fact`). |
| **Identifiers and timestamps** | **None — no timestamps, no job/run id, no sequence numbers.** The page number is the only identifier and is scoped to the current run. Correlation relies entirely on there being at most one run (`observed fact`). |
| **Error cases** | An unknown `kind` logs `[Warn] Unhandled event kind: {kind!r}` rather than dropping (`282-285`). A malformed payload raises inside `handle_event`, which is **not** individually guarded — the exception abandons the rest of that drain pass (D2.6). The pump survives because the reschedule sits in a `finally`. |
| **Restart or resume** | **None.** In-memory and unpersisted. `closing = True` makes `drain_ui_events` return without draining or rescheduling (`248-249`), so queued events at shutdown are discarded silently. |

#### Payload schemas

| Kind | Payload | Schema | Producer | Handler |
|---|---|---|---|---|
| `log` | `str` | the message, already stage-prefixed (`[1/3] ` etc.) by the caller's lambda | `process_ocr.log` | `append_log` |
| `progress` | `dict` | `{"phase": "render"\|"ocr", "current": int, "total": int}` — 1-based, `current ≤ total`, emitted **before** the work | `process_ocr.progress` | `on_progress` |
| `page_image` | `dict` | `{"page": int, "total": int, "png": bytes}` — PNG, longest side ≤ 900 px | `recognize_images` | `on_page_image` |
| `stream_chunk` | `dict` | `{"page": int, "text": str}` — **no `total` key** (the family's sole asymmetry) | `recognize_images` | `on_stream_chunk` |
| `page_text` | `dict` | `{"page": int, "total": int, "text": str}` — assembled, `.strip()`ed, guaranteed non-empty | `recognize_images` | `on_page_text` |
| `models_loaded` | `list[str]` | deduplicated, stripped, case-insensitively sorted; may be empty | `_refresh_worker` | `on_models_loaded` |
| `refresh_error` | `str` | `str(exc)` of the wrapped error, includes the URL | `_refresh_worker` | `on_refresh_error` |
| `ocr_success` | `str` | `str(saved_path)` — stringified, not a `Path` | `_ocr_worker` | `on_ocr_success` |
| `ocr_error` | `str` | `str(error)` | `_ocr_worker` | `on_ocr_error` |

Payloads are plain values only — `str`, `dict`, `list`, `bytes`. No widget, `Path`, or Tk object ever crosses; `_ocr_worker` stringifies the path specifically to hold that line (`app.py:670`) (`observed fact`).

#### The load-bearing distinction: observational vs. state-bearing

- **Observational** — `log`, `progress`, `page_image`, `stream_chunk`, `page_text`. **The saved Markdown is built from `recognize_images`' return value, not from the event stream** (`ocr_service.py:352-362`), so losing every one of them still produces a byte-identical file. Their loss degrades the display, never the artifact (`observed fact`, from the return-value data path plus the passing save tests).
- **Synchronous barriers** — three, internal to the worker and invisible to the queue: render-all before recognize-any; recognize-all before save; cleanup before the terminal event.
- **State-bearing** — `models_loaded`, `refresh_error`, `ocr_success`, `ocr_error`. **Only these four drive an `OperationState` transition; losing one strands the machine permanently** (D2.6).

A port may treat the first group as best-effort and must treat the last as guaranteed delivery.

### B2 — The service callback protocol

| Field | Value |
|---|---|
| **Producer** | `render_pdf`, `recognize_images`, `make_thumbnail_png` call sites. |
| **Consumer** | Whatever the caller injects — in production `process_ocr`'s three closures (adapting to B1); in tests plain lists and lambdas. **This substitutability is why the service suite is executable at all** (`observed fact`). |
| **Transport** | Direct synchronous calls on the worker thread. |
| **Ordering** | Strictly synchronous and in-order; a callback returns before the next line runs. |
| **Required fields** | `LogCallback = (str) -> None`; `ProgressCallback = (phase: str, current: int, total: int) -> None`; `EventCallback = (kind: str, payload: dict) -> None` (`ocr_service.py:25-27`). |
| **Optional fields** | `progress_callback` and `event_callback` are `None`-defaulted and guarded at every call site. `log_callback` is **required** — no `None` guard exists. |
| **Identifiers** | None. |
| **Error cases** | A raising callback propagates and aborts the operation — **except** the thumbnail path, where failure is caught, logged, and skipped (`182-193`). That is the only callback-adjacent error the service tolerates (**verified**). |
| **Restart or resume** | None. |

Two adaptations at the B2→B1 seam are easy to miss in a port (`observed fact`): the **positional** `ProgressCallback` signature is repacked into a **dict** payload (`315-316`), and log messages are **stage-prefixed by wrapper lambdas** at the `process_ocr` call sites (`335`, `356`), not by the emitting function — the service's own strings carry no stage marker.

### B3 — Ollama HTTP protocol

| Field | Value |
|---|---|
| **Producer** | `list_models` and `recognize_images`, via `ollama.Client`. |
| **Consumer** | The Ollama server at the user-supplied base URL. |
| **Transport** | HTTP(S) through the official client, which owns all API paths — the app never appends `/api` (`48-50`). Two independently constructed clients with different timeouts; **no pooling or reuse** across operations, and neither client is ever closed (`observed fact`). |
| **Ordering** | Strictly sequential: one page at a time, awaited to completion. **No pipelining, no concurrency, no cross-page context** — every page is an independent stateless request (`observed fact`). |
| **Required fields** | `chat(model, messages=[system, user], stream=True)`, messages exactly `[{"role":"system","content":SYSTEM_PROMPT},{"role":"user","content":"Recognize this document page.","images":[str(path)]}]`. `list()` takes no arguments. Both **verified field-for-field**. |
| **Optional fields** | None sent. No `options`, `format`, `keep_alive`, or `template` — every generation parameter is left at the server default (`observed fact`). |
| **Identifiers** | None sent. No request id, no correlation header, **no auth header**. |
| **Error cases** | `list` → `Could not fetch models from {url}: {cause}`. `chat` → `Ollama request failed on page {n}/{N} (model '{model}'): {cause}`, wrapping **both** the call and the stream iteration (**verified**). Empty assembled text → `Ollama returned no text for page {n}/{N} (model '{model}').` **Note:** this wrapper also swallows *local* failures — see the encoding finding below and D2.10. |
| **Restart or resume** | **None.** No retry, no backoff, no resumption from a partial stream. First failure aborts the document. |

#### Wire encoding of page images — resolved this run (`observed fact`)

The previous pass called this the single largest unknown for a cross-language port. Per C01 it was settled by inspecting and exercising the installed client rather than reasoning about it.

The app passes a **filesystem path string** (`"images": [str(image_path)]`, `ocr_service.py:204`). The client's `Image.serialize_model` then:

1. If the value is a `Path` or `bytes` → `b64encode(...)`.
2. If it is a `str` that names an existing file → **reads the file and base64-encodes it**.
3. If it is a `str` ending in `png`/`jpg`/`jpeg`/`webp` that does **not** exist → raises `ValueError(f'File {value} does not exist')`.
4. Otherwise it tries to treat the string as already-base64.

Exercised directly: `Image(value=str(path)).model_dump()` returns a **`str` beginning `iVBORw0KGgo`** (PNG magic in base64), and the path is **absent** from the output.

Three consequences, all now firm rather than speculative:

- **Wire format:** base64-encoded file bytes inside the JSON body. A port must base64 the page bytes; there is no path-passing shortcut.
- **A remote server never sees the path** — the client reads locally and uploads bytes. The privacy analysis therefore concerns *bytes in flight*, not path disclosure.
- **Branch 3 is a local error surfaced as a remote one.** A missing render (temp dir reaped, antivirus quarantine, filesystem error) raises client-side `ValueError` **inside** `recognize_images`' `try`, so the user is told their *Ollama request failed* and shown their model name for a purely local problem — D2.10, and the reason convention **C03** exists.

Response shapes are read defensively — `getattr(item, "model", None) or ""` (`91`) and `getattr(getattr(chunk,"message",None),"content",None) or ""` (`210-211`). **Verified this run:** the installed client still exposes `ChatResponse.message` → `Message.content` and `ListResponse.models[].model`, so the chains are currently satisfied rather than masking drift (`observed fact`). Hazard H8 covers what happens when that stops being true.

Timeout semantics are deliberate and unusual: `OCR_STREAM_IDLE_TIMEOUT = 120` is the client timeout, and because `stream=True` it applies to **gaps between chunks**, not the whole response — so a genuinely slow page never times out while a stalled connection does (`config.py:24-28`) (`observed fact`).

### B4 — Output persistence protocol

| Field | Value |
|---|---|
| **Producer** | `save_markdown_atomic` (`239-263`). |
| **Consumer** | The user's filesystem, and any editor opened via B6. |
| **Transport** | `NamedTemporaryFile(mode="w", encoding="utf-8", newline="\n", delete=False, dir=output.parent, prefix=".{stem}_", suffix=".tmp")` → `flush()` → `os.replace`. |
| **Ordering** | One write at the end of a run; publication is a same-directory rename, atomic against other processes. |
| **Required fields** | `"\n\n".join(page_texts)`, then `\r\n`→`\n` and bare `\r`→`\n`. UTF-8, no BOM. **No header, footer, page markers, or metadata** (`observed fact`, verified). |
| **Optional fields** | None — the format has no optional structure. |
| **Identifiers and timestamps** | None in content. The filename is the only identifier; filesystem mtime the only temporal record. |
| **Error cases** | Any failure → `Could not save output file: {cause}`, temp unlinked best-effort, **any pre-existing output left intact** (**verified**). |
| **Restart or resume** | None. No journal, no partial file, no resume marker. A failed run leaves the filesystem as it found it. |

### B5 — The render-directory protocol

| Field | Value |
|---|---|
| **Producer** | `render_pdf` writes `page_%04d.png` into `mkdtemp(prefix="local_ocr_")`. |
| **Consumer** | `recognize_images` (receives an ordered `list[Path]`, passes `str(path)` to the client, which base64s it) and `make_thumbnail_png`, which re-reads the same file. |
| **Transport** | The filesystem. Files are the message; the returned `list[Path]` is the manifest. |
| **Ordering** | List order is document order. Filenames are zero-padded to 4 digits so lexical order also matches numeric order past page 9 (**verified**) — belt and braces, since list order is what is consumed. |
| **Required fields** | RGB PNG, no alpha, at the requested DPI. |
| **Identifiers** | The 1-based page number, in the filename and independently as the loop index. |
| **Error cases** | `mkdtemp` failure → `Could not create temporary render directory: {cause}`. Per-page failure → `Failed to render page {n}/{N}: {cause}`. **A file missing at request time surfaces as an Ollama error, not a filesystem one** (B3 branch 3). |
| **Restart or resume** | None. Created per run, removed in `process_ocr`'s outer `finally` on every path — **except an abandoned daemon thread at interpreter exit, which a probe this run proved does happen** (D2.3). |

For image input this boundary **does not exist**: the original path passes straight through and no temp directory is created (`338-340`; **verified**) (`observed fact`).

### B6 — OS integration protocol

Argv vectors, never `shell=True`: macOS `["open", path]` / `["open","-R",path]`; Windows `os.startfile(path)` / `["explorer", f"/select,{path}"]`; other `["xdg-open", path]` / `["xdg-open", parent]`. Fire-and-forget; only the exit status is read. Failures wrap as `Could not open/reveal {path}`. **All six branches verified.** The Windows reveal branch omits `check=True` because `explorer` returns 1 on success, so it **cannot** report failure (D2.8).

---

## State Machine

### SM1 — `OperationState` (the primary machine)

Three states (`app.py:28-31`), all transitions on the Tk main thread.

| Current State | Event / Trigger | Guard | Next State | Side Effects |
|---|---|---|---|---|
| IDLE | `Select File` | passes `validate_input_path` | IDLE | set `selected_path`; label ← basename |
| IDLE | `Select File` | validation fails | IDLE | `showwarning`; **previous selection retained** |
| IDLE | `Select File` | dialog cancelled | IDLE | none |
| IDLE | `Refresh Models` | URL normalizes | **REFRESHING_MODELS** | disable url/refresh/start; log; spawn daemon thread |
| IDLE | `Refresh Models` | URL invalid | IDLE | `showerror`; **no thread, no network** |
| IDLE | `Start OCR` | no file selected | IDLE | `showerror` `No file` |
| IDLE | `Start OCR` | file now invalid (re-validated) | IDLE | `showerror` `Invalid file` |
| IDLE | `Start OCR` | URL invalid | IDLE | `showerror` `Invalid URL` |
| IDLE | `Start OCR` | model box empty | IDLE | `showerror` `No model` |
| IDLE | `Start OCR` | DPI not in `DPI_OPTIONS` | IDLE | `showerror` `Invalid DPI` |
| IDLE | `Start OCR` | output exists ∧ overwrite declined | IDLE | **silent** — nothing logged |
| IDLE | `Start OCR` | all guards pass | **PROCESSING_OCR** | freeze `OCRRequest`; disable all controls; clear panels; select **Log**; bar → indeterminate; 2 `[Start]` lines; spawn daemon thread |
| REFRESHING_MODELS | `models_loaded` | — | **IDLE** | `_restore_idle`; repopulate (**keeping a typed tag**); log count or hint |
| REFRESHING_MODELS | `refresh_error` | — | **IDLE** | `_restore_idle`; log `[Error]`; `showerror` |
| REFRESHING_MODELS | any button | — | REFRESHING_MODELS | **ignored** (state guard) |
| PROCESSING_OCR | `log`/`progress`/`page_image`/`stream_chunk`/`page_text` | — | PROCESSING_OCR | observational UI updates only; **no state change** |
| PROCESSING_OCR | `ocr_success` | — | **IDLE** | flush buffer; `_restore_idle`; log; select **Result**; completion dialog |
| PROCESSING_OCR | `ocr_error` | — | **IDLE** | flush buffer; `_restore_idle`; log `[Error]`; select **Log**; `showerror` |
| PROCESSING_OCR | any button | — | PROCESSING_OCR | **ignored** (also disabled) |
| IDLE | `WM_DELETE_WINDOW` | — | **CLOSED** | `closing = True`; `destroy()` |
| REFRESHING_MODELS / PROCESSING_OCR | `WM_DELETE_WINDOW` | user confirms | **CLOSED** | `destroy()`; **worker abandoned — and its cleanup provably does not run** (D2.3) |
| REFRESHING_MODELS / PROCESSING_OCR | `WM_DELETE_WINDOW` | user declines | unchanged | none |
| any | `WM_DELETE_WINDOW` | `closing` already true | CLOSED | `destroy()` (re-entry guard) |

Three properties matter more than the transitions (all `observed fact`):

1. **No cancellation transition exists.** Once in PROCESSING_OCR the only exits are a terminal event or window destruction — the state-machine expression of D2.3 and S3.1.
2. **No error state.** Failure returns directly to IDLE with no memory of the outcome, which is *why* the app can never offer "retry."
3. **Exactly four events can leave a busy state.** Lose one and the machine is stuck with no recovery path (D2.6).

### SM2 — Progress bar display mode (`asserted-not-verified` — GUI-suite only)

| Current State | Trigger | Next State | Side Effects |
|---|---|---|---|
| IDLE-INDETERMINATE (0, stopped) | `_apply_ocr_busy_state` | RUNNING-INDETERMINATE | mode ← indeterminate; `start()` |
| RUNNING-INDETERMINATE | first `progress` | DETERMINATE | `stop()`; mode ← determinate; `set(fraction)` |
| DETERMINATE | subsequent `progress` | DETERMINATE | `set(fraction)`; label ← `Page {c} / {t} ({Render\|OCR})` |
| DETERMINATE / RUNNING-INDETERMINATE | `_restore_idle` | IDLE-INDETERMINATE | `stop()`; indeterminate; `set(0)`; label ← `""` |

Fraction: render → `0.2·c/t`; ocr-after-render → `0.2 + 0.8·c/t`; ocr-without-render → `c/t`. `_render_phase_seen` latches on the first render event and resets in **both** `_apply_ocr_busy_state` and `_restore_idle`.

### SM3 — Result-panel streaming (`asserted-not-verified` — GUI-suite only)

| Current State | Trigger | Guard | Next State | Side Effects |
|---|---|---|---|---|
| `_result_page = 0`, buffer empty | `stream_chunk(p)` | — | `_result_page = p` | append delta; if unscheduled, `after(100ms, flush)` |
| `_result_page = p` | `stream_chunk(p)` | same page | unchanged | append delta only |
| `_result_page = p` | `stream_chunk(q)` | `q ≠ p ∧ p ≠ 0` | `_result_page = q` | **prepend `"\n\n"` to the buffer** before the delta |
| `_result_page = p` | `page_text(p)` | already streamed | unchanged | flush; register review page; **return without appending** |
| `_result_page = p` | `page_text(q)` | `q ≠ p` (unreachable — D1.2) | `_result_page = q` | flush; if `p ≠ 0` append `"\n\n"`; append full text |
| flush scheduled | 100 ms timer | — | flush cleared | write buffer; clear; clear flag |
| any | `ocr_success` / `ocr_error` | — | unchanged | **forced flush** before `_restore_idle` |
| any | `_apply_ocr_busy_state` | — | reset | clear textbox; buffer ← `""`; flag ← False; `_result_page = 0` |

The separator is written **into the buffer**, not the textbox, so a streamed run and a non-streamed one produce identical text (`app.py:553-558`).

### SM4 — Review navigation (`asserted-not-verified` — GUI-suite only)

| Current State | Trigger | Guard | Next State | Side Effects |
|---|---|---|---|---|
| empty | `page_text(p)` | first ready | 1 page, index 0 | `bisect.insort`; `show_review_page(0)` |
| n pages, index i | `page_text(q)` | `q ∉ order` | n+1, **index i unchanged** | insort; nav refresh — **user position preserved** |
| n pages, index i | `page_text(q)` | `q ∈ order` | unchanged | nav refresh (idempotent) |
| n pages, index i | `◀` / `▶` | — | `max(0,i-1)` / `min(n-1,i+1)` | re-render image + text; nav refresh |
| any | `_apply_ocr_busy_state` | — | empty | clear pages, order, index, total, LRU; label ← `No pages yet` |

Nav enablement is purely positional (`◀` iff `i > 0`, `▶` iff `i < n-1`); the label prefers the document total, falling back to the ready count (`404`). Decoded images are memoized in a 5-entry LRU. Note D1.5: `_review_index` is a **position**, not a page number, so an out-of-order arrival would shift the displayed page — safe today only because B3 is strictly sequential.

---

## Persistent Schema Notes

**The `_extracted.md` file (B4)** — the only durable artifact:

| Property | Value |
|---|---|
| Mutability | **Mutable, whole-file replace.** Not append-only; a re-run overwrites entirely. |
| History | **Linear, depth 1.** No versioning, no branching, no backup — the prior extraction is gone once overwritten (the overwrite prompt is the only guard). |
| Schema / framing | None. Plain Markdown, no envelope, no front-matter. **Page boundaries are unrecoverable** — a model-emitted `"\n\n"` is indistinguishable from a separator (`strong inference`). |
| Encoding | UTF-8, no BOM, LF-only on every platform (**verified**). |
| Compaction | None. |
| Replay or resume | **None.** No partial file, no journal; a failed run is invisible in the filesystem. |
| Locking | **None.** No lockfile, no advisory lock, no `O_EXCL`. Two instances racing produce last-writer-wins rather than interleaved corruption, because `os.replace` is atomic (`strong inference`). |
| Deduplication | Content never compared; an identical re-run rewrites the file. |
| Conflict handling | One `exists()` check at start, with a TOCTOU window spanning the whole run (`mech-CF1`). |

**The render directory (B5):** ephemeral, per-run, never read across runs, removed in a `finally` — **which a probe proved is skipped on quit** (D2.3). Names are positional and carry no run identifier; two concurrent runs are isolated only because `mkdtemp` returns distinct directories (`observed fact`).

**The `.{stem}_*.tmp` staging file (B4):** hidden, same directory as the output, `delete=False`, unlinked on the error path and consumed by `os.replace` on success. Orphaned only if the process dies inside the write window (D2.4) — reachable, given D2.3.

**Nothing else persists.** No settings file, cache, database, log, or session state (`observed fact`).

---

## Compatibility Hazards

| # | Hazard | Where | Severity | Notes |
|---|---|---|---|---|
| H1 | Page images are handed to the client as **path strings**, which it base64-encodes into the JSON body | `ocr_service.py:204`; client `Image.serialize_model` | **high** | **Resolved this run** (§B3): the wire format is base64 file bytes; the path never reaches the server. A port must base64 the bytes and keep the file readable and unmoved for the whole request. Also see H14. |
| H2 | Tk thread affinity + `after()` 50 ms polling | `app.py:81`, `247-261` | **high** | Every widget call must run on the interpreter-creating thread; the poll exists only to satisfy that. An async port should await a channel, not transliterate a timer. |
| H3 | Daemon-thread abandonment at interpreter exit | `app.py:493-495`, `654-656`; `ocr_service.py:364-366` | **high** | **Proven** this run: the `finally` does not run and the temp dir survives (D2.3). Most runtimes have no equivalent; a port needs explicit cancellation and a joined shutdown. |
| H4 | Unbounded queue as the only backpressure mechanism (i.e. none) | `app.py:59` | medium | A bounded channel changes producer behavior — an improvement (D2.5), but a deliberate behavior change. |
| H5 | **No job id, sequence number, or timestamp in any event** | entire B1 catalog | medium | The protocol is correct only because at most one operation runs. Concurrency, cancellation, or resume require extending the schema first. Routed as `protocols-CF1`. |
| H6 | LF normalization on every platform | `ocr_service.py:241` | medium | Deliberate and **verified**. A Windows port must not "fix" this to CRLF. |
| H7 | `os.replace` requires same-filesystem source and destination | `249`, `256` | medium | Satisfied by staging in `output_path.parent`. Staging in system temp silently loses atomicity. |
| H8 | Defensive `getattr` reads convert provider schema drift into a **misdiagnosed total failure** | `91`, `210-211` | medium | Fields currently match (verified), so unrealized — but a rename makes every delta empty and surfaces as `returned no text`, which the README blames on a non-vision model. Routed as `protocols-CF2`. |
| H9 | Payload strictness inconsistent for the same key | `312` vs `539` | low | `page_image` requires `total`; `page_text` tolerates its absence. |
| H10 | `stream_chunk` omits `total` while siblings carry it | schema table | low | Harmless today; a trap for a schema-generating port. |
| H11 | Zero-padded `page_%04d` overflows past 9,999 pages | `131` | low | Lexical ordering breaks; consumed list order stays correct, so impact is cosmetic. |
| H12 | Paths cross B5/B3 as `str(Path)` | `132`, `204` | low | Separator and encoding differences are platform-dependent; carry structured paths, stringify at the boundary. |
| H13 | `explorer /select,{path}` comma convention | `295` | low | Safe (argv, no shell) but Explorer-specific; reproduce exactly, do not "fix." |
| H14 | The provider client raises a **local** `ValueError` for a missing image path, inside the remote-error wrapper | client `Image.serialize_model` branch 3; `ocr_service.py:219-223` | medium | A filesystem problem is reported as `Ollama request failed … (model 'x')`. New this run; the reason convention **C03** exists (D2.10). A port that base64s inline must classify encode-time errors separately from transport errors. |

Not applicable: ANSI/terminal semantics, IME and cursor positioning, OAuth refresh, shell quoting (no `shell=True` anywhere).

---

## Coverage and limits

- **Inspected scope:** every `event_queue.put` site (7) and every `handle_event` branch (10 including the default) traced to a payload schema; all three callback aliases and their call sites; both client constructions and both request shapes; the full save path; the render-directory lifecycle; all six OS branches. Four state machines derived from source and cross-checked against tests. **Per C01, the `ollama` client's image serializer was read and exercised** (`Image(value=str(path)).model_dump()`), resolving the wire-encoding question. Both routed items closed.
- **Skipped scope:** the client's HTTP framing beyond image encoding — exact headers and the full JSON envelope — was not captured from a live server. PyMuPDF's PNG encoding beyond the three arguments passed. Tk's internal scheduler guarantees. Concurrency and security *judgments* about these protocols belong to `defect-scan-semantic` (`mech-CF1`–`mech-CF5`).
- **Secondary outputs:** this phase declares four. `public-surfaces`, `runtime-lifecycle`, and `state-and-storage` were **appended** with the protocol-level view. `findings/config-model/config-model.md` was **deliberately not written**: configuration is explicitly *not* a boundary here — `config.py` is imported directly with no serialization, precedence, or reload path — so there is no propagation protocol to document. Recorded as `protocols-D1`.
- **Evidence basis:** source inspection (primary); **runtime verification** (client serializer exercised, response types introspected, service suite results inherited); tests (used to confirm orderings and payload shapes); upstream findings.
- **Known blind spots:** (1) **no captured wire traffic** — image encoding is now established from the client's own serializer and a direct call, but the complete request envelope (headers, sibling fields) was never observed against a live server; (2) **SM2, SM3, and SM4 rest entirely on the GUI suites, which do not run here** — their transitions are `asserted-not-verified` per C02/C04, and this is the weakest part of the protocol description; (3) queue depth under a fast local model is reasoned, not measured (`q-queue-depth-under-load`); (4) Tk's ordering guarantees between `after()` callbacks and widget events are assumed from documentation; (5) H8's trigger is hypothetical — fields currently match, so no drift was observed.
- **Coverage disposition:** COMPLETE for the protocols scope. All six boundaries carry the nine required fields, the nine-event catalog has payload schemas, four state machines are tabulated with guards and side effects, and persistence and 14 hazards are documented — with the SM2–SM4 evidence tier stated rather than averaged away.

## Open Questions

| ID | Kind | Description | Deferred Reason |
|---|---|---|---|
| `q-queue-depth-under-load` | needs-runtime-test | With no `maxsize` and a 50 ms drain, steady-state queue depth against a fast local model is unknown. It determines whether D2.5's unbounded growth is theoretical or practical, and therefore how urgently a port needs bounded channels. | Needs measurement against a real GPU-backed model on a multi-page document. No static reading yields a number, and no model is available here. |

*(The previous run's `q-ollama-image-encoding` is not carried: it was resolved to `observed fact` in §B3 this phase.)*

## Carry-Forward

| ID | Target Phase | Description | Deferred Reason |
|---|---|---|---|
| `protocols-CF1` | porting | The B1 event schema carries no job id, sequence number, or timestamp, and SM1 has no cancellation transition and no error state. Any port adding cancellation, concurrent recognition, per-page retry, or resume must extend the event schema and the state machine **together** — these are one design change, not four. | Choosing the extended shape is a synthesis judgment depending on the defect synthesis (D2.1, D2.3, S3.1) and the tested-contract tradeoffs already routed as `contracts-CF1`. Deciding it here would fix a design before the port's goals are known. |
| `protocols-CF2` | defect-scan-semantic | The defensive `getattr` reads (`91`, `210-211`) **currently match** the installed client, but a rename makes every delta read empty and surfaces as `returned no text`, which the README attributes to a non-vision model. H14/D2.10 is the mild, already-live form of the same misattribution. | Provider-contract drift with a misleading diagnostic is pass 5's rubric, judged against the recovered contracts. The measured "currently matching" status is evidence the semantic pass should weigh, not a reason to drop the finding. |

---

## Validation

| # | Criterion | Result | Evidence |
|---|-----------|--------|----------|
| 1 | An event catalog is documented. | PASS | §Event Catalog covers all six boundaries (B1–B6) with the nine required fields each. B1's nine kinds have per-kind payload schemas with producer and handler, plus the observational / synchronous-barrier / state-bearing classification. Closes `arch-CF3` and `contracts-CF3`. |
| 2 | A state machine is documented. | PASS | Four machines — SM1 `OperationState` (22 transitions), SM2 progress mode, SM3 result streaming, SM4 review navigation — each tabulated with current state, trigger, guard, next state, and side effects, and each labelled with its evidence tier. |
| 3 | Persistent schema notes are documented. | PASS | §Persistent Schema Notes covers the `.md` artifact across mutability, history, framing, encoding, compaction, replay, locking, deduplication, and conflict handling, plus the render directory and staging file, and states that nothing else persists. |
| 4 | Compatibility hazards are documented. | PASS | 14 hazards (H1–H14) with location, severity, and porting notes; H1 now resolved rather than speculative and H14 new this run. Inapplicable hazard classes named explicitly. |
| 5 | Findings are marked with evidence levels. | PASS | `observed fact`, `strong inference`, `asserted-not-verified` (per C02, for the GUI-only state machines), and `open question` used throughout, with `file:line` citations, named passing tests, or the exercised command. |
| 6 | Coverage and limits name inspected scope, skipped scope, evidence basis, and blind spots. | PASS | All four named plus a COMPLETE disposition; **all four declared secondary outputs accounted for** (three appended, one deliberately not, with rationale); the SM2–SM4 evidence tier named as the weakest part. One open question and two carry-forwards routed. |

**Validated by:** 2026-08-18 (protocols phase, MCP-driven session 2, framework v0.16.0)
**Overall:** PASS
