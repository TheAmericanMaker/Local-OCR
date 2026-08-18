# Semantic Defects Report — Local-OCR

## Scan Context

- **Source:** `../` (repository root), commit `8e7388c`
- **Architecture reference:** `findings/architecture/architecture-map.md`
- **Contracts reference:** `findings/contracts/behavioral-contracts.md`
- **Protocols reference:** `findings/protocols/protocols-and-state.md`
- **Mechanical defects reference:** `findings/defect-scan-mechanical/mechanical-defects.md`
- **Pipeline:** `full-with-deep-audit`
- **Date:** 2026-08-16
- **Scope:** Semantic passes only (3 concurrency, 4 security, 5 contract violations). Mechanical passes (1 logic, 2 error handling, 6 configuration) were covered in `defect-scan-mechanical`.
- **Action set:** pre-porting (`fix before porting` / `port differently` / `leave behind`).
- **Routed items closed here:** `arch-CF5` (S3.2, S3.5), `mech-CF1` (S4.2, S4.5), `mech-CF2` (S4.3), `mech-CF3` (§Pass 3 — Verified Safe), `mech-CF4` (S5.2, S5.3), `protocols-CF2` (S5.1). All six resolved; none re-routed.

One routed item — `mech-CF3` — resolved as **not a defect**. That outcome is reported in full rather than dropped, because the invariant it questioned is load-bearing for any port.

---

## Pass 3: Concurrency and Resource Management

| # | Location | Defect | Severity | Evidence Level | Action |
|---|----------|--------|----------|----------------|--------|
| S3.1 | `ocr_service.py:197-208`, `config.py:24-28` | No overall job timeout and no cancellation: a slow-but-alive stream can hang the app indefinitely with no recourse but force-quit. | high | observed fact | fix before porting |
| S3.2 | `app.py:250-256`, `app.py:317-324`, `app.py:368-374` | The drain loop is unbounded and calls expensive image-decode handlers on the Tk main thread. **Closes `arch-CF5`.** | medium | strong inference | fix before porting |
| S3.3 | `app.py:743-752` + `ocr_service.py:352-363` | Quitting does not abort the worker, so the output file can be written *after* the window is gone. | medium | strong inference | fix before porting |
| S3.4 | `ocr_service.py:88`, `ocr_service.py:343` | `ollama.Client` instances are constructed per operation and never closed; each leaves an unclosed connection pool. | low | strong inference | port differently |
| S3.5 | `ocr_service.py:121-137` | All N page images are held for the whole job when the pipeline consumes exactly one at a time. **Closes `arch-CF5`.** | low | observed fact | fix before porting |

### S3.1 — No overall timeout, no cancellation: an unbounded hang (high)

`OCR_STREAM_IDLE_TIMEOUT = 120` is passed as the client timeout and, because `stream=True`, applies to **gaps between chunks** rather than to the whole response (`config.py:24-28`, protocols §B3). That choice is correct for its stated purpose — a genuinely slow page must not be killed — but it means **no upper bound on a page exists, and no upper bound on the document exists**. A model emitting one token every 119 seconds never trips the idle timer; a 50-page document in that state runs effectively forever.

Protocols SM1 makes the consequence exact: PROCESSING_OCR has **no cancellation transition**. The only exits are a terminal event that will not arrive and window destruction. So the user's sole recourse is to quit — which triggers S3.3 and D2.3 (temp-dir leak). The three combine into a single bad outcome: *the app can enter a state it cannot leave, and the only escape leaks resources and may still write a file.*

This is `high` rather than `medium` because it is user-reachable without anything malfunctioning — an oversized page and an underpowered model are enough. **Action:** `fix before porting`. A port needs a per-page wall-clock ceiling in addition to the idle timeout, plus a cooperative cancellation token checked between pages and between chunks.

### S3.2 — Unbounded main-thread drain over expensive handlers (medium) — closes `arch-CF5`

`drain_ui_events` drains the **entire** queue in a single main-thread pass:

```python
while True:
    try:
        kind, payload = self.event_queue.get_nowait()
    except queue.Empty:
        break
    self.handle_event(kind, payload)      # app.py:250-256
```

There is no per-pass budget and no yield back to Tk. Most handlers are trivial, but two are not: `on_page_image` runs `Image.open` plus `CTkImage` construction (`app.py:317-324`), and `_review_image_for` decodes and rescales on navigation (`app.py:368-374`). Both execute on the Tk main thread, so the window is unresponsive for their duration.

The interaction with D2.5 is what makes this more than a micro-optimization: the queue is unbounded, so a worker running ahead of the UI can accumulate many events, and the next drain processes **all** of them back-to-back — every queued `page_image` decode in one uninterruptible block. Architecture flagged the main-thread decode; protocols established that the drain is unbounded; together they give the real shape, which is why this belongs to the semantic pass rather than the mechanical one.

Note the asymmetry that limits severity: `stream_chunk` handling is cheap (string append plus a scheduled flush), so the highest-frequency event is not the expensive one. Decodes are once per page. **Action:** `fix before porting`. Decode off the main thread and hand over ready-to-display bytes, or cap events processed per drain pass and reschedule immediately when the cap is hit.

### S3.3 — The worker outlives the window and can still write output (medium)

`on_close` sets `closing` and calls `destroy()` without joining or signalling the worker (`app.py:743-752`). The worker is mid-`process_ocr`. If it happens to be at stage `[3/3]`, `save_markdown_atomic` may complete **after** the window is gone, publishing a file the user believes they cancelled (`ocr_service.py:352-363`).

Three distinct outcomes, depending on where the interpreter shutdown lands:

1. Before `os.replace` → nothing published, possibly an orphaned `.{stem}_*.tmp` (D2.4).
2. After `os.replace` → **the output file exists**, created after the user quit.
3. Mid-render → the temp directory leaks (D2.3).

Outcome 2 is the semantic one and is not covered by any mechanical finding: quitting is not an abort, and the app gives no indication that work may still land. Contracts F10 documents the confirmation dialog but promises nothing about what happens to in-flight work — because nothing is promised in code either. `os.replace` is atomic, so no *corrupt* file results; the defect is the surprise, not corruption. **Action:** `fix before porting`. Shutdown should signal cancellation and join with a timeout, so quitting means quitting.

### S3.4 — Ollama clients are never closed (low)

A fresh `ollama.Client` is built per operation — `list_models` (`ocr_service.py:88`) and `process_ocr` (`ocr_service.py:343`) — and neither is closed, nor used as a context manager. The underlying httpx client owns a connection pool with keep-alive sockets, released only when GC finalizes the object. Protocols §B3 records that there is no pooling or reuse across operations by design.

Impact is bounded: at most one operation runs at a time, so at most a couple of pools are outstanding, and a desktop app's process lifetime is short. It is nonetheless an unreleased OS resource on every run, and it will matter more in a port that keeps a long-lived process or adds batching. **Action:** `port differently` — own the client's lifetime explicitly (context manager or an app-scoped client).

### S3.5 — Every page image retained for the whole job (low) — closes `arch-CF5`

`render_pdf` writes all N PNGs before returning, and `recognize_images` then consumes them one at a time (`ocr_service.py:121-137`, sequenced at `ocr_service.py:331-352`). Peak resource usage is N pages; steady-state need is one. Nothing deletes a page's PNG after recognition, so temp usage only grows until the whole directory is removed at the end.

This is the resource-management face of D6.3 (which rated the disk-exhaustion risk) and the second half of `arch-CF5`. Recorded here separately because the fix is the same one that shrinks D2.3's leak and improves time-to-first-token: **stream the pipeline**. Rendering one page ahead of recognition bounds temp usage to a constant, deletes each page as it is consumed, and lets the first page reach the model without waiting for the last one to rasterize. **Action:** `fix before porting`.

### Verified safe — closes `mech-CF3`

`mech-CF3` asked whether the lock-free `OperationState` mutual exclusion is actually sound, and whether `self.closing` is a cross-thread race. Both were checked by enumerating every read and write site and the thread each runs on. **Neither is a defect** (`observed fact`).

**`operation_state`** — all sites are on the Tk main thread:

| Site | Access | Thread | Why |
|---|---|---|---|
| `select_file` (`app.py:461`) | read | main | Tk button command |
| `refresh_models` (`app.py:483`, `490`) | read, write | main | Tk button command |
| `start_ocr` (`app.py:596`, `650`) | read, write | main | Tk button command |
| `_restore_idle` (`app.py:456`) | write | main | called only from `on_models_loaded`, `on_refresh_error`, `on_ocr_success`, `on_ocr_error` — all reached via `handle_event` from the `after()` pump |
| `on_close` (`app.py:744`) | read | main | Tk protocol handler |

No worker path touches it: `_refresh_worker` and `_ocr_worker` only call `event_queue.put` (`app.py:497-503`, `658-670`). Tk serializes command callbacks and `after()` callbacks on one thread, so every check-then-act sequence is atomic with respect to the others. **The lock-free design is correct, and adding a lock would add nothing.**

**`self.closing`** — written at `app.py:745` and `751` (`on_close`, main thread), read at `app.py:248` (`drain_ui_events`, main thread) and `app.py:744`. Both sides are main-thread; there is no cross-thread access and therefore no visibility or ordering concern. `mech-CF3`'s suspicion was reasonable from the mechanical pass's context-light vantage but does not survive the thread analysis.

**Why this matters for the port:** the invariant is *"all mutable UI state is confined to a single thread; the queue is the only shared object."* That is what makes the whole design safe without locks. A port to a runtime with different thread-affinity rules must re-establish the confinement explicitly rather than assume the absence of locks means the absence of a concurrency contract (protocols H2).

---

## Pass 4: Security and Trust Boundaries

Contracts §Security and Authorization establishes the intended model: no authentication, no authorization, no secrets, a single local user, and one trusted outbound destination. Findings below are measured against that model rather than against a server-application model.

| # | Location | Defect | Severity | Evidence Level | Action |
|---|----------|--------|----------|----------------|--------|
| S4.1 | `ocr_service.py:109`, `ocr_service.py:150`, `requirements.txt` | Untrusted binary documents are parsed by two uncapped native libraries — the app's real attack surface. | medium | strong inference | port differently |
| S4.2 | `app.py:632-641` → `ocr_service.py:362` | Overwrite-check TOCTOU silently defeats the F4 data-loss guard. **Closes `mech-CF1`.** | medium | observed fact | fix before porting |
| S4.3 | `ocr_service.py:45-62`, `config.py:3` | Document images are sent over unauthenticated plaintext HTTP with no guardrail on non-loopback hosts. **Closes `mech-CF2`.** | medium | observed fact | port differently |
| S4.4 | `ocr_service.py:209-230` | No bound on accumulated model output; a hostile or malfunctioning server can exhaust memory and disk. | medium | observed fact | fix before porting |
| S4.5 | `app.py:606` → `ocr_service.py:331,340` | Input-path TOCTOU between validation and use. **Closes `mech-CF1`.** | low | observed fact | leave behind |
| S4.6 | `ocr_service.py:266-280` + `app.py:705-710` | Untrusted model output is written verbatim and then handed to the OS default application. | low | strong inference | port differently |

### S4.1 — Untrusted binary parsing is the real attack surface (medium)

The app's most exposed trust boundary is not the network — it is `pymupdf.open(pdf_path)` (`ocr_service.py:109`) and `Image.open(image_path)` (`ocr_service.py:150`), both handed arbitrary user-selected files. Contracts identifies the input document as untrusted; PyMuPDF (MuPDF) and Pillow are large native codebases with a steady history of parser CVEs. A malicious PDF is the one input that reaches native code before any of the app's own logic runs.

Two things make this worse than baseline:

- **Neither library is version-capped** (`PyMuPDF>=1.24.0`, no upper bound; Pillow is capped at `<12`) and there is **no lockfile**, so the installed parser version is whatever `pip` resolved that day (D6.2).
- **No size or page-count limit** is applied before parsing, so a decompression-bomb PDF is accepted (`ocr_service.py:65-77` checks existence, type, extension, and readability — never size).

Per the pass's own scope note, CVE scanning is out of scope; the finding is the *structural* exposure and the absence of any bound, not a specific vulnerability. **Action:** `port differently` — pin and track the parsers deliberately, add an input size/page-count ceiling, and treat document parsing as the component most worth isolating.

### S4.2 — Overwrite-check TOCTOU silently defeats the data-loss guard (medium) — closes `mech-CF1`

Contracts F4 promises a specific protection: *if an extraction already exists, the user is asked before it is replaced.* The check and the write are far apart:

```python
if output_path.exists():                      # app.py:633
    overwrite = messagebox.askyesno(...)      # app.py:634-639
...
save_markdown_atomic(request.output_path, ...)  # ocr_service.py:362 — minutes later
```

The window spans the **entire OCR run** — potentially many minutes on a large document. If `doc_extracted.md` did not exist at check time but is created during the run (another app, a sync client, a second copy of Local OCR, the user themselves), it is **overwritten with no prompt at all**. The guard the user was promised simply does not apply to anything created after the check.

This is a data-loss path rather than an attack: it needs no adversary, only ordinary concurrent file activity over a multi-minute window. It is the more consequential half of `mech-CF1`. **Action:** `fix before porting` — re-check immediately before publishing, or create the destination with `O_EXCL` at start to reserve the name.

### S4.3 — Plaintext, unauthenticated transport of document content (medium) — closes `mech-CF2`

`normalize_ollama_url` accepts any `http://` or `https://` host with a netloc (`ocr_service.py:55-61`). It enforces no TLS, warns on no host, and treats `http://localhost:11434` and `http://203.0.113.10:11434` identically. Ollama has no built-in authentication (contracts §Security), so page images — the full content of the user's document, which is the most sensitive data this app touches — are transmitted **unencrypted and unauthenticated** to whatever address is typed.

The README is honest about the risk in prose ("exposing Ollama beyond localhost makes it reachable by anyone who can connect to that port… Ollama has no built-in authentication"), and in the intended localhost deployment there is no real exposure. The defect is that **the code offers no guardrail**: the unsafe configuration is exactly as easy as the safe one, with no warning when the host is non-loopback and no way to require HTTPS. The product's central claim — "Nothing leaves your machine or network" — is therefore enforced entirely by user discipline (`observed fact`).

Rated `medium`, not `high`: the default is safe, the risk requires deliberate reconfiguration, and it is documented. **Action:** `port differently` — warn (or require confirmation) the first time a non-loopback host is used, and allow requiring TLS.

### S4.4 — No bound on accumulated model output (medium)

```python
for chunk in stream:
    delta = ...
    if delta:
        chunks.append(delta)          # ocr_service.py:209-218
content = "".join(chunks).strip()     # ocr_service.py:224
```

Nothing caps the number of chunks, the length of a page's text, or the total document size. A server that streams indefinitely — malfunctioning, a model in a degenerate repetition loop, or hostile — grows `chunks` without limit; every delta is *also* enqueued as a `stream_chunk` event into the unbounded queue (D2.5) and appended to the Tk textbox. Memory pressure therefore lands in three places at once, and whatever survives is written to the user's disk.

The idle timeout does not help: a server emitting steadily is never idle. Contracts §Security marks the Ollama server as *implicitly* trusted — this finding is what that implicit trust costs. Degenerate repetition loops are a well-known VLM failure mode, so this does not require an adversary. **Action:** `fix before porting` — cap per-page and per-document output size and fail with a clear message when exceeded.

### S4.5 — Input-path TOCTOU (low) — closes `mech-CF1`

`validate_input_path` runs on the main thread at `app.py:606`; the worker opens the file at `ocr_service.py:331` (PDF) or passes the path to the client at `ocr_service.py:340` (image). Between them sit the overwrite dialog and thread startup. The file can be deleted, replaced, or swapped for a symlink in that window.

Consequences are mild by construction: a deleted file yields a wrapped `Could not open PDF` error, and a replaced file is simply the file that gets processed. There is no privilege boundary to cross — the app runs as the user, reads a file the user chose, and writes beside it. An "attacker" would need write access to the user's own directory, at which point the TOCTOU is not the interesting capability. Recorded for completeness and rated honestly. **Action:** `leave behind` — the re-validation at `app.py:606` is already more than most desktop apps do; a port need not chase the residual window.

### S4.6 — Untrusted output handed to the OS default application (low)

The saved Markdown is model output derived from an untrusted document, written **verbatim** with no escaping or sanitization (`ocr_service.py:362`). The completion dialog then offers `Open`, which launches it via `open` / `os.startfile` / `xdg-open` (`app.py:705-710`).

The exposure is real but narrow: the file always carries a `.md` extension, so it is never executed; Markdown is inert; and the handler is whatever the user configured. The residual risk is a viewer that renders embedded raw HTML or auto-loads remote resources, turning OCR'd text into a tracking beacon or a rendered-content vector. A document crafted to make a VLM emit `<img src="http://attacker/...">` is a plausible if contrived chain. **Action:** `port differently` — nothing needs fixing in the file, but a port that adds an in-app preview must not render HTML.

### Verified safe

Two areas checked and found clean (`observed fact`):

- **No secrets of any kind.** No API keys, tokens, passwords, or connection strings in source, and no `.env` or credential files in the tree. Nothing is logged that could contain one — the only interpolated values in log lines are the URL, the model tag, and file paths (`app.py:652-653`). There is nothing to rotate because there is nothing to store.
- **No command injection.** All three OS-integration paths pass argv vectors with no `shell=True` (`ocr_service.py:274-297`). The Windows `f"/select,{path}"` interpolation is a single argv element, not a shell string, so a filename containing `&`, `;`, quotes, or spaces cannot break out. Verified against every `subprocess` call site.

Not applicable: authentication, authorization, session management, multi-tenancy, CORS/CSP/XSS/CSRF, and rate limiting — the app has no auth model, no server, and no web surface (contracts §Security).

---

## Pass 5: API Contract Violations

| # | Location | Defect | Severity | Evidence Level | Action | Spec Reference |
|---|----------|--------|----------|----------------|--------|----------------|
| S5.1 | `ocr_service.py:210-211`, `ocr_service.py:91` | Provider schema drift is silently converted into a total failure reported with the wrong cause. **Closes `protocols-CF2`.** | high | strong inference | fix before porting | protocols §B3 (defensive reads, H8); contracts F12 error behavior |
| S5.2 | `README.md` troubleshooting table vs `ocr_service.py:224-229` | "Returned no text" is documented as having one cause; it has at least three. **Closes `mech-CF4`.** | medium | observed fact | fix before porting | contracts F12; Doc/Test Conflict #2 |
| S5.3 | `README.md:82-84` vs `ocr_service.py:239-263` | The atomicity claim promises durability the implementation does not provide. **Closes `mech-CF4`.** | low | observed fact | fix before porting | contracts F11; protocols §B4; Doc/Test Conflict #1 |
| S5.4 | `ocr_service.py:302-310` | `process_ocr`'s docstring documents one event kind; the function emits five. | low | observed fact | port differently | protocols §B1 event catalog |
| S5.5 | `ocr_service.py:166-175` | `recognize_images`' docstring describes a non-streaming mode that does not exist. | low | observed fact | leave behind | protocols §B3 (`stream=True` hardcoded) |
| S5.6 | `app.py:421-425` | SM1 says REFRESHING_MODELS ignores every button; three controls remain visually enabled. | low | observed fact | port differently | protocols SM1; mechanical D1.3 |
| S5.7 | `app.py:312` vs `app.py:539` | The same payload key is required by one handler and optional in another. | low | observed fact | port differently | protocols §B1 optional fields, H9 |

### S5.1 — Provider drift becomes a silent, misdiagnosed total failure (high) — closes `protocols-CF2`

**Spec:** protocols §B3 records that Ollama responses are read defensively rather than typed, and hazard H8 predicts the failure mode. Contracts F12 specifies that a request failure surfaces an error naming the page, the model, and the underlying cause.

**Divergence:** the defensive read swallows the case it most needs to report.

```python
delta = (getattr(getattr(chunk, "message", None), "content", None) or "")   # ocr_service.py:210-211
```

If a future `ollama` release renames `message.content` — or a proxy returns a differently shaped object — every `getattr` yields `None`, every delta becomes `""`, no chunk is appended, and `content` is empty. The code then raises:

```
Ollama returned no text for page 1/N (model 'x').
```

That message is not merely unhelpful, it is **actively misleading**: the README's troubleshooting table maps exactly this string to "The selected model has no vision support. Choose a vision-capable model." A user hitting a client-library incompatibility is told to change models, which will not help, and every model they try will fail identically. `list_models` has the same shape at `ocr_service.py:91`, where a renamed `item.model` would silently yield an empty list reported as "No models found on the server."

Rated `high`: the trigger is realistic (`ollama>=0.4.0` is uncapped with no lockfile — D6.2, so an ordinary `pip install` can introduce it), the blast radius is total (every page of every document), and the diagnostic points the wrong way. The `or ""` fallback is right for a *missing optional* field and wrong for a *required* one. **Action:** `fix before porting` — distinguish "chunk carried no content" from "chunk had an unrecognized shape," and fail loudly on the latter.

### S5.2 — "Returned no text" has three causes, one documented (medium) — closes `mech-CF4`

**Spec:** the README troubleshooting table attributes `returned no text` solely to a model without vision support. Contracts F12 records the error string.

**Divergence:** the guard at `ocr_service.py:224-229` fires for *any* empty assembled text. At least three distinct causes reach it: (a) a non-vision model, as documented; (b) a legitimately blank page — a chapter verso or scanned separator sheet (D2.2); (c) a client/server schema mismatch (S5.1). Only (a) is documented, and only (a) is fixed by the suggested remedy.

Cause (b) is the common one in ordinary use, and it is severe out of proportion to its nature: one blank page aborts the entire document (D2.1). A user with a 200-page scan containing one empty page is told their model lacks vision support. **Action:** `fix before porting` — the documentation fix is trivial, but the real fix is upstream: stop treating an empty page as fatal.

### S5.3 — Atomicity claim overstates the guarantee (low) — closes `mech-CF4`

**Spec:** `README.md:82-84` — "The file is written atomically, so a failed run never leaves a partial result." Contracts F11 and protocols §B4 record the write path.

**Divergence:** the temp-then-`os.replace` sequence (`ocr_service.py:239-263`) is atomic against *process* failure — the rename either happens or it does not, and tests pin that the previous file survives a failed replace. It is **not** durable: `flush()` pushes to the page cache and there is no `fsync`, so an OS crash or power loss can leave a zero-length or truncated file at the destination (D2.4). "Atomically" is the word most readers will take to cover the crash case they actually fear.

Rated `low` because the implementation is correct for the failure class it targets and the tests verify it honestly; only the prose overreaches. **Action:** `fix before porting` — add the `fsync` so the claim becomes true, which is cheaper than qualifying the sentence.

### S5.4 — `process_ocr` documents one event kind and emits five (low)

**Spec:** the docstring reads "Run the full OCR pipeline; emit `('log', message)` events; return output" (`ocr_service.py:303`). Protocols §B1 catalogues what actually crosses the queue.

**Divergence:** `process_ocr` installs three closures and emits **five** kinds — `log`, `progress`, `page_image`, `stream_chunk`, `page_text` (`ocr_service.py:312-319`, plus `emit_event` forwarded into `recognize_images`). The `event_queue` parameter carries no type annotation either, so the contract is structural and undocumented. The return type is correctly documented.

An integrator reading only the docstring would build a consumer handling one event kind and hit the `[Warn] Unhandled event kind` path for four others. **Action:** `port differently` — make the event catalogue an explicit type at the boundary rather than a docstring aside.

### S5.5 — Docstring describes a mode that does not exist (low)

**Spec:** `recognize_images`' docstring states "The non-streaming result is identical — streaming only adds the live deltas" (`ocr_service.py:172-174`).

**Divergence:** there is no non-streaming path. `stream=True` is hardcoded at `ocr_service.py:207` with no parameter to change it (protocols §B3). The sentence describes a hypothetical equivalence, and it is the same assumption that keeps D1.2's unreachable fallback branch alive in `on_page_text`. Harmless in itself, but it documents a capability a reader may believe exists. **Action:** `leave behind` — if a port genuinely offers both modes the sentence becomes true; otherwise drop it.

### S5.6 — UI state does not reflect SM1's guards (low)

**Spec:** protocols SM1 — in REFRESHING_MODELS, *any button* is ignored via the state guard.

**Divergence:** `_apply_refresh_busy_state` disables only `url_entry`, `refresh_button`, and `start_button` (`app.py:421-425`), leaving `select_button`, the model combobox, and the DPI combobox visually enabled. `select_file` then returns immediately on its state guard (`app.py:461-462`) with no dialog and no log line. The state machine is enforced correctly; the *presentation* of it is not, so the affordance lies about what is available. Same underlying observation as mechanical D1.3, restated here as the spec divergence it is. **Action:** `port differently` — derive enablement from one state→control mapping.

### S5.7 — Inconsistent payload strictness for the same key (low)

**Spec:** protocols §B1 lists `total` as a required field of both `page_image` and `page_text`; hazard H9 records the inconsistency.

**Divergence:** `on_page_image` reads `payload["total"]` and raises `KeyError` if absent (`app.py:312`); `on_page_text` reads `payload.get("total", self._review_total)` and tolerates absence (`app.py:539`). One key, two contracts. Today both producers always supply it, so neither path is exercised — but a `KeyError` escaping `on_page_image` would abandon the rest of that drain pass (D2.6). Note `stream_chunk` omits `total` entirely (H10), so the event family is non-uniform in schema as well as in enforcement. **Action:** `port differently` — one schema, one validation point at the queue boundary.

---

## Summary

### Findings by Severity

| Severity | Count |
|----------|-------|
| Critical | 0 |
| High | 2 |
| Medium | 7 |
| Low | 9 |
| **Total** | **18** |

### Findings by Pass

| Pass | Critical | High | Medium | Low | Total |
|------|----------|------|--------|-----|-------|
| 3. Concurrency and resources | 0 | 1 | 2 | 2 | 5 |
| 4. Security and trust | 0 | 0 | 4 | 2 | 6 |
| 5. API contract violations | 0 | 1 | 1 | 5 | 7 |
| **Total** | **0** | **2** | **7** | **9** | **18** |

Combined with the mechanical scan, the full audit stands at **39 findings — 0 critical, 5 high, 14 medium, 20 low.**

### Top Findings

1. **S5.1** (pass 5) — a provider field rename silently becomes a total failure reported as "no vision support," misdirecting every diagnostic attempt; reachable today because `ollama` is uncapped with no lockfile. `high` → `fix before porting`.
2. **S3.1** (pass 3) — no overall job timeout and no cancellation, so the app can enter a state it cannot leave; the only escape (quit) leaks the temp directory and may still write a file. `high` → `fix before porting`.
3. **S4.2** (pass 4) — the overwrite prompt guards a check made minutes before the write, so a file created during the run is destroyed without the promised confirmation. `medium` → `fix before porting`.
4. **S4.4** (pass 4) — unbounded accumulation of model output across three buffers at once; degenerate repetition loops are a normal VLM failure, not an exotic attack. `medium` → `fix before porting`.
5. **S3.2** (pass 3) — an unbounded drain loop running image decodes on the Tk main thread, amplified by the unbounded queue. `medium` → `fix before porting`.

Two structural themes for `porting` to carry:

- **Absent bounds.** No job timeout, no output size cap, no input size cap, no queue bound, no per-drain budget, no temp-space check. Every dimension that could grow, grows. The system is correct on the happy path and unbounded everywhere else.
- **Errors that misdirect.** S5.1, S5.2, and S5.3 all describe the same weakness: the app is confident about causes it has not established. Defensive reads and generous prose combine so that a user chasing a failure is pointed at the wrong thing.

### Carry-Forward Resolution

All six items routed to this phase are **resolved; none re-routed.**

| Routed ID | From | Resolved by | Outcome |
|---|---|---|---|
| `arch-CF5` | architecture | S3.2, S3.5 | Confirmed. Main-thread decode rated `medium` in combination with the unbounded drain; eager rasterization rated `low` as the resource face of D6.3. |
| `mech-CF1` | defect-scan-mechanical | S4.2, S4.5 | Split. The **output** TOCTOU is a real `medium` data-loss path defeating contract F4; the **input** TOCTOU is `low` with no privilege boundary to cross. |
| `mech-CF2` | defect-scan-mechanical | S4.3 | Confirmed `medium`. Default is safe; the defect is the absence of any guardrail on the unsafe configuration. |
| `mech-CF3` | defect-scan-mechanical | §Pass 3 — Verified Safe | **Refuted.** Every `operation_state` and `closing` access is main-thread; the lock-free design is correct. The confinement invariant is recorded for the port. |
| `mech-CF4` | defect-scan-mechanical | S5.2, S5.3 | Confirmed, split by severity: the misattributed "returned no text" cause is `medium`; the overstated atomicity claim is `low`. |
| `protocols-CF2` | protocols | S5.1 | Confirmed and **escalated to `high`** — protocols predicted the mechanism (H8); this pass established that the diagnostic actively misdirects and that the trigger is reachable through ordinary dependency resolution. |

---

## Coverage and limits

- **Inspected scope:** all four first-party modules re-read against the three semantic checklists in order (3 → 4 → 5). Pass 3 enumerated every `operation_state`, `closing`, and `event_queue` access site with its owning thread, plus every resource acquisition (temp dir, temp file, PDF document, PIL image, Ollama client) against its release path. Pass 4 walked every external input (document bytes, URL, model tag, DPI, model output) and every outbound sink (HTTP, filesystem, subprocess) against the contracts' security model. Pass 5 compared implementation against three specs: the 13 contracts, the six protocol boundaries and four state machines, and the README plus in-source docstrings. All six routed carry-forwards addressed.
- **Skipped scope:** mechanical passes 1, 2, and 6 — covered in `defect-scan-mechanical` and not re-litigated; findings there are cross-referenced (D-prefixed) rather than restated. Dependency CVE scanning is explicitly out of scope per pass 4's own guidance, so S4.1 reports structural exposure rather than named vulnerabilities. Cryptographic algorithm review is not applicable (no cryptography). Third-party internals (`ollama` HTTP framing, MuPDF and Pillow parsers, Tk's scheduler) were not audited.
- **Evidence basis:** source inspection, upstream findings (architecture, contracts, protocols, mechanical defects — used as the spec for pass 5 and the threat model for pass 4), tests (used to confirm which behaviors are locked in), and project documentation as the pass-5 baseline.
- **Known blind spots:** (1) **no runtime verification** — nothing was executed, no server contacted, no traffic captured, so S3.1's hang, S3.2's stall duration, and S4.4's growth rate are reasoned rather than measured; (2) S5.1's trigger is hypothetical-but-realistic — no specific `ollama` release is known to have renamed the field, and confirming which versions are affected needs dependency testing this pipeline does not perform; (3) whether `pymupdf.open` or `Image.open` has an exploitable path against a crafted document is unknowable without fuzzing, so S4.1 is a structural finding only; (4) the thread-confinement proof in §Verified Safe rests on Tk serializing command and `after()` callbacks on one thread — documented behavior, not observed here; (5) `q-ollama-image-encoding` (protocols) remains open, so whether page bytes or only a path reach a remote server is still unconfirmed, which bounds how precisely S4.3's exposure can be stated.
- **Coverage disposition:** COMPLETE for the semantic scope. All three passes ran to completion over the full first-party source with the contracts and protocols outputs in hand; all six routed items are resolved with none re-routed; remaining gaps are the runtime blind spots above and one previously-opened protocol question.

## Validation

| # | Criterion | Result | Evidence |
|---|-----------|--------|----------|
| 1 | All three semantic passes (3, 4, 5) produced findings or documented "no defects found." | PASS | Pass 3 → 5 findings plus a Verified Safe subsection; pass 4 → 6 findings plus a Verified Safe subsection naming secrets and command injection as clean; pass 5 → 7 findings. |
| 2 | Each finding has location, severity, evidence level, and recommended action. | PASS | All 18 rows carry `file:line` location, severity, evidence level, and a pre-porting action; each has a prose subsection expanding evidence and rationale. |
| 3 | Pass 5 findings cite the contract or protocol reference they violate. | PASS | The pass 5 table has a dedicated **Spec Reference** column populated for all 7 rows (protocols §B1/§B3/§B4/SM1/H8/H9, contracts F11/F12, README lines, and the contracts phase's Doc/Test Conflict entries); each subsection opens with an explicit **Spec:** / **Divergence:** pair. |
| 4 | Findings are organized by pass and sorted by severity; summary tables match the detailed findings. | PASS | One section per pass, rows descending high → medium → low. §Findings by Severity totals 18 (0/2/7/9); §Findings by Pass rows total 5 + 6 + 7 = 18 with column sums 0/2/7/9. Both reconcile with S3.1–S3.5, S4.1–S4.6, S5.1–S5.7. |
| 5 | Any carry_forward entries that targeted defect-scan-semantic have been resolved or explicitly re-routed. | PASS | §Carry-Forward Resolution maps all six (`arch-CF5`, `mech-CF1`, `mech-CF2`, `mech-CF3`, `mech-CF4`, `protocols-CF2`) to the findings that resolve them, including one refutation (`mech-CF3`) and one escalation (`protocols-CF2` → high). None re-routed; all six listed in `carry_forward_closures` in the phase handoff. |
| 6 | Findings are marked with evidence levels. | PASS | 13 `observed fact`, 5 `strong inference`, 0 `open question` — every finding labelled, with the Verified Safe determinations marked `observed fact` from exhaustive site enumeration. |
| 7 | Coverage and limits name inspected scope, skipped scope, evidence basis, and blind spots. | PASS | §Coverage and limits names all four plus a COMPLETE disposition, and states per-pass enumeration methods along with five specific blind spots, including the absence of runtime verification. |

**Validated by:** 2026-08-16 (defect-scan-semantic phase, MCP-driven session 1)
**Overall:** PASS
