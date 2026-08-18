# Semantic Defects Report — Local-OCR

## Scan Context

- **Source:** `../`, commit `8e7388c` · **Framework:** v0.16.0 · **Date:** 2026-08-18
- **References:** `architecture-map.md`, `behavioral-contracts.md`, `protocols-and-state.md`, `mechanical-defects.md`
- **Scope:** Semantic passes only (3 concurrency, 4 security, 5 contract violations). Mechanical passes covered in `defect-scan-mechanical`.
- **Action set:** pre-porting (`fix before porting` / `port differently` / `leave behind`).
- **Routed items closed here (7):** `arch-CF5` (S3.2, S3.5) · `mech-CF1` (S4.2, S4.5) · `mech-CF2` (S4.3) · `mech-CF3` (§Pass 3 — Verified Safe) · `mech-CF4` (S5.2, S5.3) · `mech-CF5` + `protocols-CF2` (S5.1). All resolved; none re-routed.
- **Conventions honored:** C01/C05 (execution and dependency-reading resolved the encoding question upstream, which sharpens S4.3 and S4.5 here); C02/C04 (evidence tiering); C03 (S5.2 extends the misattribution catalogue).

### Orchestrator duties discharged

**Open-question re-triage** (5 inherited). All five labels re-tested and **confirmed**: `q-test-invocation`, `q-prompt-provenance`, `q-gui-tests-ever-run` (`needs-maintainer-decision` — project history or a maintainer statement, none derivable by reading); `q-ctk-image-clear`, `q-queue-depth-under-load` (`needs-runtime-test` — both need a resource this host lacks: a `tkinter` build and a GPU-backed model respectively). C01 was applied to each and reaches none of them.

**Contradiction sweep.** One item, and it **changes a finding's severity rather than being smoothed over**: the protocols phase measured that the installed `ollama` client *still exposes* `ChatResponse.message.content` and `ListResponse.models[].model`. The previous pass rated the provider-drift finding `high` partly because it was "reachable today through ordinary dependency resolution." Measurement contradicts that premise — the newest resolvable client satisfies the chains. S5.1 is therefore rated **medium**, not high, with the reasoning stated. Downgrading on evidence is the honest move even though it weakens a headline finding.

---

## Pass 3: Concurrency and Resource Management

| # | Location | Defect | Severity | Evidence Level | Action |
|---|---|---|---|---|---|
| S3.1 | `ocr_service.py:197-208`, `config.py:24-28` | No overall job timeout and no cancellation: a slow-but-alive stream hangs the app indefinitely with no recourse but force-quit. | high | observed fact | fix before porting |
| S3.2 | `app.py:250-256`, `317-324`, `368-374` | The drain loop is unbounded and runs image decodes on the Tk main thread. **Closes `arch-CF5`.** | medium | strong inference | fix before porting |
| S3.3 | `app.py:743-752` + `ocr_service.py:352-363` | Quitting does not abort the worker, so the output file can be published *after* the window is gone. | medium | strong inference | fix before porting |
| S3.4 | `ocr_service.py:88`, `343` | `ollama.Client` instances are built per operation and never closed. | low | observed fact | port differently |
| S3.5 | `ocr_service.py:121-137` | All N page images are held for the whole job when exactly one is needed at a time. **Closes `arch-CF5`.** | low | observed fact | fix before porting |

### S3.1 — No overall timeout, no cancellation: an unbounded hang (high)

`OCR_STREAM_IDLE_TIMEOUT = 120` is the client timeout and, because `stream=True`, applies to **gaps between chunks** rather than the whole response (`config.py:24-28`, protocols §B3). Correct for its stated purpose — a genuinely slow page must not be killed — but it means **no upper bound exists on a page, or on the document**. A model emitting one token every 119 s never trips the idle timer; a 50-page document in that state runs effectively forever.

Protocols SM1 makes the consequence exact: PROCESSING_OCR has **no cancellation transition**. The only exits are a terminal event that will not arrive, or window destruction — which triggers S3.3 and, as D2.3 **proved by probe**, leaks the render directory. The three compose into one bad outcome: *the app can enter a state it cannot leave, and the only escape leaks resources and may still write a file.*

`high` because it is user-reachable with nothing malfunctioning — an oversized page and an underpowered model suffice. **Action:** `fix before porting`. A per-page wall-clock ceiling **in addition to** the idle timeout, plus a cooperative cancellation token checked between pages and between chunks.

### S3.2 — Unbounded main-thread drain over expensive handlers (medium) — closes `arch-CF5`

`drain_ui_events` drains the **entire** queue in one main-thread pass with no per-pass budget and no yield back to Tk (`app.py:250-256`). Most handlers are trivial; two are not — `on_page_image` runs `Image.open` plus `CTkImage` construction (`317-324`), and `_review_image_for` decodes and rescales on navigation (`368-374`). Both execute on the Tk main thread, so the window is unresponsive for their duration.

The interaction with D2.5 is what elevates this above a micro-optimization: the queue is unbounded, so a worker running ahead accumulates events, and the next drain processes **all** of them back-to-back — every queued `page_image` decode in one uninterruptible block. Architecture flagged the main-thread decode; protocols established the drain is unbounded; together they give the real shape, which is why it belongs to the semantic pass.

One asymmetry limits severity: `stream_chunk` handling is cheap (string append plus a scheduled flush), so the highest-frequency event is not the expensive one — decodes are once per page. **Action:** `fix before porting`. Decode off the main thread and hand over ready-to-display bytes, or cap events per drain pass and reschedule immediately at the cap.

### S3.3 — The worker outlives the window and can still publish output (medium)

`on_close` sets `closing` and calls `destroy()` without joining or signalling the worker (`app.py:743-752`), which is mid-`process_ocr`. If it happens to be at stage `[3/3]`, `save_markdown_atomic` may complete **after** the window is gone, publishing a file the user believes they cancelled (`ocr_service.py:352-363`).

Three outcomes depending on where interpreter shutdown lands:

1. Before `os.replace` → nothing published, possibly an orphaned `.{stem}_*.tmp` (D2.4).
2. After `os.replace` → **the output file exists**, created after the user quit.
3. Mid-render → the temp directory leaks — **no longer hypothetical**, since D2.3's probe showed the cleanup `finally` does not run.

Outcome 2 is the semantic finding and no mechanical finding covers it: quitting is not an abort, and the app gives no indication that work may still land. Contracts F10 documents the confirmation dialog but promises nothing about in-flight work — because nothing is promised in code. `os.replace` is atomic, so no *corrupt* file results; the defect is the surprise. Note the probe strengthens this: since cleanup demonstrably does not run at exit, the interpreter is clearly not waiting on the worker, so whichever statement the worker is executing when the main thread returns is simply abandoned mid-flight. **Action:** `fix before porting`. Shutdown should signal cancellation and join with a timeout.

### S3.4 — Ollama clients never closed (low)

A fresh `ollama.Client` is built per operation — `list_models` (`88`) and `process_ocr` (`343`) — and neither is closed nor used as a context manager. The underlying httpx client owns a keep-alive connection pool released only on finalization; protocols §B3 records that there is no pooling or reuse by design. Impact is bounded: one operation at a time, short process lifetime. It is still an unreleased OS resource per run, and it matters more in a port with a long-lived process or batching. **Action:** own the client's lifetime explicitly.

### S3.5 — Every page image retained for the whole job (low) — closes `arch-CF5`

`render_pdf` writes all N PNGs before returning; `recognize_images` consumes them one at a time (`121-137`, sequenced at `331-352`). Peak usage is N pages; steady-state need is one, and nothing deletes a page's PNG after recognition. This is the resource-management face of D6.3 and the second half of `arch-CF5`, recorded separately because the fix is the same one that shrinks D2.3's proven leak and improves time-to-first-token: **stream the pipeline**. Rendering one page ahead bounds temp usage to a constant, deletes each page as consumed, and lets page 1 reach the model without waiting for page N to rasterize. **Action:** `fix before porting`.

### Verified safe — closes `mech-CF3`

`mech-CF3` asked whether the lock-free `OperationState` exclusion is sound and whether `self.closing` races. Both were checked by enumerating **every** read and write site with its owning thread. **Neither is a defect** (`observed fact`).

`operation_state` — all sites on the Tk main thread:

| Site | Access | Thread | Why |
|---|---|---|---|
| `select_file` (`app.py:461`) | read | main | Tk button command |
| `refresh_models` (`483`, `490`) | read, write | main | Tk button command |
| `start_ocr` (`596`, `650`) | read, write | main | Tk button command |
| `_restore_idle` (`456`) | write | main | reached only from the four terminal handlers, all dispatched by the `after()` pump |
| `on_close` (`744`) | read | main | Tk protocol handler |

No worker path touches it: `_refresh_worker` and `_ocr_worker` only call `event_queue.put` (`497-503`, `658-670`). Tk serializes command and `after()` callbacks on one thread, so every check-then-act is atomic with respect to the others. **The lock-free design is correct; a lock would add nothing.**

`self.closing` — written at `745` and `751` (`on_close`, main thread), read at `248` (`drain_ui_events`, main thread) and `744`. Both sides main-thread; no cross-thread access, so no visibility or ordering concern. `mech-CF3`'s suspicion was reasonable from the mechanical pass's context-light vantage but does not survive thread analysis.

**Why this matters for the port:** the invariant is *"all mutable UI state is confined to one thread; the queue is the only shared object."* That is what makes the design safe without locks. A port to a runtime with different affinity rules must re-establish the confinement **explicitly** — the absence of locks is a consequence of the invariant, not a substitute for one (protocols H2).

---

## Pass 4: Security and Trust Boundaries

Contracts §Security establishes the intended model: no authentication, no authorization, no secrets, one local user, one trusted outbound destination. Findings are measured against that, not a server-application model.

| # | Location | Defect | Severity | Evidence Level | Action |
|---|---|---|---|---|---|
| S4.1 | `ocr_service.py:109`, `150`; `requirements.txt` | Untrusted binary documents are parsed by two native libraries, one uncapped, with no size limit — the real attack surface. | medium | strong inference | port differently |
| S4.2 | `app.py:632-641` → `ocr_service.py:362` | Overwrite-check TOCTOU silently defeats the F4 data-loss guard. **Closes `mech-CF1`.** | medium | observed fact | fix before porting |
| S4.3 | `ocr_service.py:45-62`, `config.py:3` | Full document content is base64'd and sent over unauthenticated plaintext HTTP, with no guardrail on non-loopback hosts. **Closes `mech-CF2`.** | medium | observed fact | port differently |
| S4.4 | `ocr_service.py:209-230` | No bound on accumulated model output; a hostile or degenerate server exhausts memory and disk. | medium | observed fact | fix before porting |
| S4.5 | `app.py:606` → client encode time | Input-path TOCTOU, with a window now known to extend to base64 encode time. **Closes `mech-CF1`.** | low | observed fact | leave behind |
| S4.6 | `ocr_service.py:266-280` + `app.py:705-710` | Untrusted model output is written verbatim, then handed to the OS default application. | low | strong inference | port differently |

### S4.1 — Untrusted binary parsing is the real attack surface (medium)

The most exposed trust boundary is not the network — it is `pymupdf.open(pdf_path)` (`109`) and `Image.open(image_path)` (`150`), both handed arbitrary user-selected files. Contracts marks the input document untrusted; MuPDF and Pillow are large native codebases with a steady history of parser CVEs. A malicious PDF reaches native code before any of the app's own logic runs.

Two aggravating facts: **PyMuPDF is not version-capped** and there is **no lockfile**, so the installed parser is whatever `pip` resolved that day (D6.2) — this run resolved 1.28.2; and **no size or page-count limit** precedes parsing (`65-77` checks existence, type, extension, readability — never size), so a decompression-bomb PDF is accepted. Per the pass's own scope note CVE scanning is excluded, so this is the *structural* exposure and the absence of any bound, not a named vulnerability. **Action:** `port differently` — pin and track the parsers, add input size/page-count ceilings, treat parsing as the component most worth isolating.

### S4.2 — Overwrite-check TOCTOU silently defeats the data-loss guard (medium) — closes `mech-CF1`

Contracts F4 promises: *if an extraction already exists, the user is asked before it is replaced.* Check and write are far apart — `output_path.exists()` at `app.py:633`, `save_markdown_atomic` at `ocr_service.py:362`, **minutes later on a large document**. If `doc_extracted.md` did not exist at check time but is created during the run (another app, a sync client, a second copy of Local OCR, the user themselves), it is **overwritten with no prompt at all**. The guard simply does not apply to anything created after the check.

This is data loss, not an attack: it needs no adversary, only ordinary concurrent file activity across a multi-minute window. It is the more consequential half of `mech-CF1`. **Action:** `fix before porting` — re-check immediately before publishing, or reserve the name with `O_EXCL` at start.

### S4.3 — Full document content over unauthenticated plaintext (medium) — closes `mech-CF2`

`normalize_ollama_url` accepts any `http://` or `https://` host with a netloc (`55-61`). It enforces no TLS, warns on no host, and treats `http://localhost:11434` and `http://203.0.113.10:11434` identically. Ollama has no built-in authentication (contracts §Security).

**The protocols phase sharpened exactly what is exposed.** It was previously unclear whether page bytes or merely a path reached the server; it is now established that the client **base64-encodes the file into the JSON body** (protocols §B3, `observed fact`). So what crosses the wire is **the complete pixel content of every page of the user's document** — the most sensitive data this app touches — unencrypted and unauthenticated, to whatever address was typed.

The README is honest in prose ("exposing Ollama beyond localhost makes it reachable by anyone who can connect to that port… Ollama has no built-in authentication"), and in the intended localhost deployment there is no real exposure. The defect is that **the code offers no guardrail**: the unsafe configuration is exactly as easy as the safe one, with no warning on a non-loopback host and no way to require HTTPS. The product's central claim — "Nothing leaves your machine or network" — is enforced entirely by user discipline (`observed fact`). Rated `medium`: the default is safe, the risk requires deliberate reconfiguration, and it is documented. **Action:** `port differently` — warn or require confirmation the first time a non-loopback host is used; allow requiring TLS.

### S4.4 — No bound on accumulated model output (medium)

```python
for chunk in stream:
    if delta: chunks.append(delta)      # ocr_service.py:209-218
content = "".join(chunks).strip()       # 224
```

Nothing caps chunk count, per-page length, or total document size. A server streaming indefinitely — malfunctioning, a model in a degenerate repetition loop, or hostile — grows `chunks` without limit; every delta is *also* enqueued into the unbounded queue (D2.5) **and** appended to the Tk textbox, so memory pressure lands in three places at once, and whatever survives is written to the user's disk. The idle timeout does not help: a steadily-emitting server is never idle. Contracts marks the server *implicitly* trusted — this is what that implicit trust costs. Degenerate repetition is a well-known VLM failure mode, so no adversary is required. **Action:** `fix before porting` — cap per-page and per-document output with a clear failure message.

### S4.5 — Input-path TOCTOU, window now known to extend to encode time (low) — closes `mech-CF1`

`validate_input_path` runs on the main thread at `app.py:606`; the worker later opens the file (`ocr_service.py:331`, PDF) or passes the path onward (`340`, image). Between them sit the overwrite dialog and thread startup.

The protocols phase extends the window further than the previous reading assumed: because the client **re-reads the file at base64 encode time**, the path must still resolve *at request time*, per page — so for a PDF the window stretches across the entire recognition loop, not just to the initial open. A file deleted mid-run yields the client's `ValueError('File ... does not exist')`, which surfaces as `Ollama request failed` (hazard H14, D2.10) — misattributed, though that is C03's concern rather than a security one.

Consequences remain mild by construction: there is no privilege boundary to cross. The app runs as the user, reads a file the user chose, and writes beside it; an "attacker" needs write access to the user's own directory, at which point the TOCTOU is not the interesting capability. Recorded honestly rather than inflated. **Action:** `leave behind` — the re-validation at `app.py:606` already exceeds most desktop apps; a port need not chase the residual window.

### S4.6 — Untrusted output handed to the OS default application (low)

The saved Markdown is model output derived from an untrusted document, written **verbatim** with no escaping (`362`), and the completion dialog offers `Open`, which launches it via `open` / `os.startfile` / `xdg-open` (`app.py:705-710`). Exposure is narrow: the file always carries a `.md` extension so it is never executed, Markdown is inert, and the handler is user-configured. The residual risk is a viewer that renders embedded raw HTML or auto-loads remote resources, turning OCR'd text into a tracking beacon. A document crafted to make a VLM emit `<img src="http://attacker/...">` is a plausible if contrived chain. **Action:** `port differently` — nothing needs fixing in the file, but an in-app preview must not render HTML.

### Verified safe

Two areas checked and clean (`observed fact`):

- **No secrets of any kind.** No API keys, tokens, passwords, or connection strings in source; no `.env` or credential files. Nothing logged could contain one — the only interpolated values in log lines are the URL, the model tag, and file paths (`app.py:652-653`). There is nothing to rotate because there is nothing to store.
- **No command injection.** All three OS paths pass argv vectors with no `shell=True` (`274-297`); the Windows `f"/select,{path}"` interpolation is a single argv element, so a filename containing `&`, `;`, quotes, or spaces cannot break out. Verified against every `subprocess` call site, and **all six branches are covered by passing service tests**.

Not applicable: authentication, authorization, session management, multi-tenancy, CORS/CSP/XSS/CSRF, rate limiting — no auth model, no server, no web surface.

---

## Pass 5: API Contract Violations

| # | Location | Defect | Severity | Evidence Level | Action | Spec Reference |
|---|---|---|---|---|---|---|
| S5.1 | `ocr_service.py:210-211`, `91` | Provider schema drift would become a total failure reported with the wrong cause. **Closes `mech-CF5` + `protocols-CF2`.** | medium | strong inference | fix before porting | protocols §B3, H8; contracts F12 error behavior |
| S5.2 | `README.md` troubleshooting vs `ocr_service.py:224-229` | "Returned no text" is documented as having one cause; it has **four**. **Closes `mech-CF4`.** | medium | observed fact | fix before porting | contracts F12; Doc/Test Conflict #2 |
| S5.3 | `README.md:82-84` vs `ocr_service.py:239-263` | The atomicity claim promises durability the implementation lacks. **Closes `mech-CF4`.** | low | observed fact | fix before porting | contracts F11; protocols §B4; Conflict #1 |
| S5.4 | `ocr_service.py:302-310` | `process_ocr`'s docstring documents one event kind; it emits five. | low | observed fact | port differently | protocols §B1 event catalog |
| S5.5 | `ocr_service.py:166-175` | `recognize_images`' docstring describes a non-streaming mode that does not exist. | low | observed fact | leave behind | protocols §B3 (`stream=True` hardcoded) |
| S5.6 | `app.py:421-425` | SM1 says REFRESHING_MODELS ignores every button; three controls stay visually enabled. | low | observed fact | port differently | protocols SM1; D1.3 |
| S5.7 | `app.py:312` vs `539` | The same payload key is required by one handler and optional in another. | low | observed fact | port differently | protocols §B1 optional fields, H9 |

### S5.1 — Provider drift would become a misdiagnosed total failure (medium) — closes `mech-CF5` + `protocols-CF2`

**Spec:** protocols §B3 records that Ollama responses are read defensively rather than typed, and hazard H8 predicts the failure mode. Contracts F12 specifies that a request failure surfaces an error naming page, model, and underlying cause.

**Divergence:** the defensive read swallows the case it most needs to report.

```python
delta = (getattr(getattr(chunk, "message", None), "content", None) or "")   # 210-211
```

If a future `ollama` release renames `message.content` — or a proxy returns a differently shaped object — every `getattr` yields `None`, every delta becomes `""`, no chunk is appended, and the code raises `Ollama returned no text for page 1/N (model 'x')`. That message is not merely unhelpful but **actively misleading**: the README maps exactly this string to "The selected model has no vision support. Choose a vision-capable model." A user hitting a client incompatibility is told to change models, and every model they try fails identically. `list_models` has the same shape at `91`, where a renamed `item.model` yields an empty list reported as "No models found on the server."

**Rated `medium`, not high — on measurement.** The previous pass rated this `high` partly because it was "reachable today through ordinary dependency resolution." The protocols phase **introspected the installed client and found the fields still present**: `ChatResponse.message` → `Message.content`, `ListResponse.models[].model`. So the newest resolvable client satisfies the chains, and the trigger is a *future* rename rather than a current one. Impact remains total and the diagnostic remains wrong, which keeps it above `low`; but the premise that justified `high` is contradicted by evidence, and honesty about that outweighs preserving a headline. The mild form **is** already live: hazard H14/D2.10, where a local missing-file `ValueError` is reported as an Ollama failure. **Action:** `fix before porting` — distinguish "chunk carried no content" from "chunk had an unrecognized shape," and fail loudly on the latter. `or ""` is right for a missing *optional* field and wrong for a required one.

### S5.2 — "Returned no text" has four causes, one documented (medium) — closes `mech-CF4`

**Spec:** the README troubleshooting table attributes `returned no text` solely to a model lacking vision support. Contracts F12 records the string.

**Divergence:** the guard at `224-229` fires for *any* empty assembled text. Four distinct causes reach it, and this run identified the fourth:

1. A non-vision model — as documented.
2. **A legitimately blank page** — a chapter verso or scanned separator (D2.2). Ordinary in real documents, and it aborts the entire document via D2.1.
3. **A client/server schema mismatch** — S5.1.
4. **A missing local render** — the client's own `ValueError` for an absent path, wrapped as an Ollama failure (protocols H14, D2.10). Strictly this surfaces as `Ollama request failed` rather than `returned no text`, but it lands in the same troubleshooting row from the user's point of view: the README sends them to the model.

Only cause 1 is documented, and only cause 1 is fixed by the suggested remedy. Cause 2 is the common one in ordinary use. **Action:** `fix before porting` — the documentation fix is trivial, but the real fixes are upstream (D2.2, S5.1, C03).

### S5.3 — Atomicity claim overstates the guarantee (low) — closes `mech-CF4`

**Spec:** `README.md:82-84` — "The file is written atomically, so a failed run never leaves a partial result." Contracts F11 and protocols §B4 record the write path.

**Divergence:** temp-then-`os.replace` (`239-263`) is atomic against *process* failure — the rename either happens or it does not, and **passing tests** confirm the previous file survives a failed replace. It is **not durable**: `flush()` reaches the page cache and there is no `fsync`, so an OS crash or power loss can leave a zero-length or truncated file at the destination (D2.4). "Atomically" is the word most readers take to cover the crash case they actually fear. Rated `low` because the implementation is correct for its target failure class and the tests verify it honestly; only the prose overreaches. **Action:** `fix before porting` — add the `fsync`, which makes the claim true more cheaply than qualifying the sentence.

### S5.4 — `process_ocr` documents one event kind and emits five (low)

**Spec:** the docstring reads "Run the full OCR pipeline; emit `('log', message)` events; return output" (`303`). Protocols §B1 catalogues what actually crosses.

**Divergence:** `process_ocr` installs three closures and emits **five** kinds — `log`, `progress`, `page_image`, `stream_chunk`, `page_text` (`312-319`, plus `emit_event` forwarded into `recognize_images`). The `event_queue` parameter carries no type annotation either, so the contract is structural and undocumented. An integrator reading only the docstring would build a consumer for one kind and hit `[Warn] Unhandled event kind` for four others. The return type is correctly documented. **Action:** `port differently` — make the event catalogue an explicit type at the boundary.

### S5.5 — Docstring describes a mode that does not exist (low)

**Spec:** `recognize_images`' docstring states "The non-streaming result is identical — streaming only adds the live deltas" (`172-174`). **Divergence:** there is no non-streaming path; `stream=True` is hardcoded at `207` with no parameter to change it (protocols §B3). The sentence describes a hypothetical equivalence — and it is the same assumption keeping D1.2's unreachable fallback alive in `on_page_text`. Harmless in itself, but it documents a capability a reader may believe exists. **Action:** `leave behind` — if a port genuinely offers both modes the sentence becomes true; otherwise drop it.

### S5.6 — UI state does not reflect SM1's guards (low)

**Spec:** protocols SM1 — in REFRESHING_MODELS, *any button* is ignored via the state guard. **Divergence:** `_apply_refresh_busy_state` disables only `url_entry`, `refresh_button`, and `start_button` (`421-425`), leaving `select_button`, the model combobox, and the DPI combobox visually enabled; `select_file` then returns immediately on its guard (`461-462`) with no dialog and no log line. The state machine is enforced correctly; its *presentation* is not, so the affordance lies about what is available. Same observation as D1.3, restated as the spec divergence it is. **Action:** `port differently` — derive enablement from one state→control mapping.

### S5.7 — Inconsistent payload strictness for the same key (low)

**Spec:** protocols §B1 lists `total` as required on both `page_image` and `page_text`; hazard H9 records the inconsistency. **Divergence:** `on_page_image` reads `payload["total"]` and raises `KeyError` if absent (`312`); `on_page_text` reads `payload.get("total", self._review_total)` and tolerates absence (`539`). One key, two contracts. Both producers always supply it today, so neither path is exercised — but a `KeyError` escaping `on_page_image` would abandon the rest of that drain pass (D2.6). Note `stream_chunk` omits `total` entirely (H10), so the family is non-uniform in schema as well as enforcement. **Action:** `port differently` — one schema, one validation point at the queue boundary.

---

## Summary

### Findings by Severity

| Severity | Count |
|---|---|
| Critical | 0 |
| High | 1 |
| Medium | 8 |
| Low | 9 |
| **Total** | **18** |

### Findings by Pass

| Pass | Critical | High | Medium | Low | Total |
|---|---|---|---|---|---|
| 3. Concurrency and resources | 0 | 1 | 2 | 2 | 5 |
| 4. Security and trust | 0 | 0 | 4 | 2 | 6 |
| 5. API contract violations | 0 | 0 | 2 | 5 | 7 |
| **Total** | **0** | **1** | **8** | **9** | **18** |

Combined with the mechanical scan, the full audit stands at **41 findings — 0 critical, 4 high, 16 medium, 21 low.**

### Top Findings

1. **S3.1** — no overall job timeout and no cancellation, so the app can enter a state it cannot leave; the only escape (quit) provably leaks the temp directory and may still write a file. `high` → `fix before porting`.
2. **S4.2** — the overwrite prompt guards a check made minutes before the write, so a file created during the run is destroyed without the promised confirmation. `medium` → `fix before porting`.
3. **S4.4** — unbounded accumulation of model output across three buffers at once; degenerate repetition is a normal VLM failure, not an exotic attack. `medium` → `fix before porting`.
4. **S4.3** — the complete pixel content of every page travels base64'd over unauthenticated plaintext HTTP, with no guardrail making the unsafe host harder to choose than the safe one. `medium` → `port differently`.
5. **S5.1** — a provider field rename would produce a total failure reported as "no vision support." `medium` (downgraded from the previous pass on measurement) → `fix before porting`.

Two structural themes for `porting`:

- **Absent bounds.** No job timeout, no output size cap, no input size cap, no queue bound, no per-drain budget, no temp-space check. The system is correct on the happy path and unbounded in every other direction.
- **Errors that misdirect.** S5.1, S5.2, S5.3, and (from the mechanical scan) D2.10 are one weakness in four places: defensive reads and confident prose combine so the app asserts causes it has not established. This is what convention **C03** was promoted to prevent.

### Carry-Forward Resolution

All seven routed items **resolved; none re-routed.**

| Routed ID | From | Resolved by | Outcome |
|---|---|---|---|
| `arch-CF5` | architecture | S3.2, S3.5 | Confirmed. Main-thread decode `medium` in combination with the unbounded drain; eager rasterization `low` as the resource face of D6.3. |
| `mech-CF1` | mechanical | S4.2, S4.5 | **Split.** The *output* TOCTOU is a real `medium` data-loss path defeating F4; the *input* TOCTOU is `low` with no privilege boundary — and its window is now known to extend to base64 encode time. |
| `mech-CF2` | mechanical | S4.3 | Confirmed `medium`, and **sharpened**: the protocols phase established that full page bytes, not a path, cross the wire. |
| `mech-CF3` | mechanical | §Verified Safe | **Refuted.** Every `operation_state` and `closing` access is main-thread; the lock-free design is correct. The confinement invariant is recorded for the port. |
| `mech-CF4` | mechanical | S5.2, S5.3 | Confirmed, split by severity, and **extended**: "returned no text" now has four documented causes rather than three. |
| `mech-CF5` | mechanical | S5.1 | Confirmed as a mechanism. |
| `protocols-CF2` | protocols | S5.1 | Confirmed **and downgraded to `medium`** — measurement showed the installed client still exposes the fields, contradicting the "reachable today" premise that justified `high`. |

---

## Coverage and limits

- **Inspected scope:** all four modules re-read against the three semantic checklists in order (3 → 4 → 5). Pass 3 enumerated every `operation_state`, `closing`, and `event_queue` access site with its owning thread, and every resource acquisition (temp dir, temp file, PDF document, PIL image, Ollama client) against its release path. Pass 4 walked every external input (document bytes, URL, model tag, DPI, model output) and every outbound sink (HTTP, filesystem, subprocess) against the contracts' security model. Pass 5 compared implementation against three specs: the 13 contracts, the six boundaries and four state machines, and the README plus in-source docstrings. All seven routed items addressed.
- **Skipped scope:** mechanical passes 1/2/6 — covered earlier and cross-referenced (D-prefixed) rather than restated. Dependency CVE scanning is excluded by pass 4's own guidance, so S4.1 reports structural exposure. Cryptographic review is not applicable. Third-party internals were not audited for quality — though per C05 the `ollama` serializer was *read to establish the contract*, which is what sharpened S4.3 and S4.5.
- **Evidence basis:** source inspection; upstream findings (used as the pass-5 spec and the pass-4 threat model); **runtime verification inherited from earlier phases** (service suite results, the daemon-thread probe behind S3.3, client type introspection behind S5.1's downgrade); tests, to establish which behaviors are locked in and at which evidence tier.
- **Known blind spots:** (1) **no live model and no captured traffic** — S3.1's hang, S3.2's stall duration, and S4.4's growth rate are reasoned, not measured; (2) S5.1's trigger is a *future* rename — measurement establishes only that today's client is compatible, not which future version breaks; (3) whether `pymupdf.open` or `Image.open` has an exploitable path against a crafted document is unknowable without fuzzing, so S4.1 is structural only; (4) the thread-confinement proof rests on Tk serializing command and `after()` callbacks on one thread — documented behavior, not observed here, and unobservable without `tkinter`; (5) five open questions remain outstanding, all needing a resource this environment lacks (a `tkinter` build, a display, a GPU-backed model, or a maintainer).
- **Coverage disposition:** COMPLETE for the semantic scope. All three passes ran over the full first-party source with contracts and protocols in hand; all seven routed items resolved with none re-routed; remaining gaps are the blind spots above.

## Validation

| # | Criterion | Result | Evidence |
|---|-----------|--------|----------|
| 1 | All three semantic passes (3, 4, 5) produced findings or documented "no defects found." | PASS | Pass 3 → 5 findings plus a Verified Safe subsection; pass 4 → 6 findings plus a Verified Safe subsection naming secrets and command injection clean; pass 5 → 7 findings. |
| 2 | Each finding has location, severity, evidence level, and recommended action. | PASS | All 18 rows carry `file:line` location, severity, evidence level, and a pre-porting action, each with a prose subsection expanding evidence and rationale. |
| 3 | Pass 5 findings cite the contract or protocol reference they violate. | PASS | The pass 5 table carries a dedicated **Spec Reference** column populated for all 7 rows (protocols §B1/§B3/§B4/SM1/H8/H9, contracts F11/F12, README lines, contracts' Doc/Test Conflict entries); each subsection opens with an explicit **Spec:** / **Divergence:** pair. |
| 4 | Findings are organized by pass and sorted by severity; summary tables match the detailed findings. | PASS | One section per pass, rows descending high → medium → low. §Findings by Severity totals 18 (0/1/8/9); §Findings by Pass rows total 5 + 6 + 7 = 18 with column sums 0/1/8/9. Both reconcile with S3.1–S3.5, S4.1–S4.6, S5.1–S5.7. |
| 5 | Any carry_forward entries that targeted defect-scan-semantic have been resolved or explicitly re-routed. | PASS | §Carry-Forward Resolution maps all seven (`arch-CF5`, `mech-CF1`–`mech-CF5`, `protocols-CF2`) to resolving findings, including one refutation (`mech-CF3`) and one evidence-driven **downgrade** (`protocols-CF2` → medium). None re-routed; all seven in `carry_forward_closures`. |
| 6 | Findings are marked with evidence levels. | PASS | 15 `observed fact`, 3 `strong inference`, 0 `open question` — every finding labelled, with the Verified Safe determinations marked `observed fact` from exhaustive site enumeration. |
| 7 | Coverage and limits name inspected scope, skipped scope, evidence basis, and blind spots. | PASS | All four named plus a COMPLETE disposition, with per-pass enumeration methods and five blind spots. Orchestrator duties (re-triage of five questions; a contradiction sweep that changed S5.1's severity) are discharged in §Scan Context. |

**Validated by:** 2026-08-18 (defect-scan-semantic phase, MCP-driven session 2, framework v0.16.0)
**Overall:** PASS
