# Mechanical Defects Report — Local-OCR

## Scan Context

- **Source:** `../` (repository root), commit `8e7388c`
- **Architecture reference:** `findings/architecture/architecture-map.md`
- **Pipeline:** `full-with-deep-audit`
- **Date:** 2026-08-16
- **Scope:** Mechanical passes only (1 logic, 2 error handling, 6 configuration). Semantic passes (3 concurrency, 4 security, 5 contract violations) deferred to `defect-scan-semantic` after protocols.
- **Action set:** pre-porting (`fix before porting` / `port differently` / `leave behind`), per the `full-with-deep-audit` pipeline.
- **Routed items closed here:** `arch-CF1` (see **D2.3**), `arch-CF2` (see **D2.5**).

Emphasis follows the architecture map: the app has a real error model and a real config surface, so passes 2 and 6 carried the weight; pass 1 is comparatively thin because the codebase is small, linear, and heavily commented.

---

## Pass 1: Logic and Correctness

| # | Location | Defect | Severity | Evidence Level | Action |
|---|----------|--------|----------|----------------|--------|
| D1.1 | `app.py:505-524` `on_progress` | Progress reports a page as done when it has only *started*, so the bar reaches 100% before the last page is recognized. | medium | observed fact | port differently |
| D1.2 | `app.py:526-548` `on_page_text` | The non-streaming fallback append branch is unreachable in practice — dead defensive code. | low | strong inference | leave behind |
| D1.3 | `app.py:421-425` `_apply_refresh_busy_state` | Three controls stay visually enabled during a model refresh but silently no-op. | low | observed fact | port differently |
| D1.4 | `app.py:329`, `app.py:341` `_clear_preview` / `_clear_review` | Image clearing relies on `CTkLabel.configure(image=None)`, whose clearing behavior is version-dependent. | low | open question | port differently |
| D1.5 | `app.py:349-359` `_register_review_page` | `bisect.insort` can shift the page the user is viewing; safe only because pages happen to arrive in ascending order. | low | strong inference | port differently |

### D1.1 — Progress fraction is off by one page (medium)

`on_progress` is driven by `progress_callback("ocr", number, total)` fired at the **top** of each loop iteration, *before* the page is sent (`ocr_service.py:179-180`; same pattern for `"render"` at `ocr_service.py:123-124`). The handler treats `current` as work completed:

```python
fraction = 0.2 + 0.8 * current / total     # app.py:515
```

With `current == total` on the final page, the bar sits at 100% for the entire duration of the slowest single operation in the job. For a **single image** the effect is starker: `_render_phase_seen` is `False`, so `fraction = current / total = 1/1 = 1.0` (`app.py:517`) the instant recognition begins — the bar is full for the whole job. The `Page N / N` label is simultaneously correct-looking, which makes the app appear hung. **Evidence:** the callback sites emit `number` before `client.chat` is called; the handler applies no `- 1`. **Action:** compute the fraction from completed work (`(current - 1) / total`), or emit progress after each page completes.

### D1.2 — Unreachable fallback branch in `on_page_text` (low)

`on_page_text` guards with `if page == self._result_page: return` and otherwise appends the page text (`app.py:543-548`). That append is unreachable: `recognize_images` raises `OCRServiceError` when `"".join(chunks).strip()` is empty (`ocr_service.py:224-229`), so any page that reaches a `page_text` event emitted at least one non-empty delta, which already set `_result_page = page` in `on_stream_chunk` (`app.py:559`). The docstring acknowledges this ("cannot happen for a non-empty page, but keeps the panel correct should streaming ever be bypassed"), so this is deliberate defensive code, not an oversight. Recorded because it is genuinely dead on every current path and a port should not treat it as a required behavior. **Action:** leave behind unless the port supports a non-streaming mode, in which case it becomes live and must be tested.

### D1.3 — Controls enabled but inert during model refresh (low)

`_apply_refresh_busy_state` disables only `url_entry`, `refresh_button`, and `start_button` (`app.py:421-425`), while `_apply_ocr_busy_state` additionally disables `select_button`, `model_combobox`, and `dpi_combobox` (`app.py:426-442`). During a refresh, `Select File` therefore renders as enabled, but `select_file` returns immediately because of its `operation_state is not IDLE` guard (`app.py:461-462`). The click is swallowed with no dialog and no log line. `_restore_idle` then re-enables `select_button` unconditionally (`app.py:445`), which is harmless but confirms the asymmetry was unintentional. **Action:** in a port, derive control enablement from a single state→enabled-set mapping rather than from two hand-maintained methods.

### D1.4 — Image clearing depends on toolkit-specific `configure(image=None)` semantics (low, open question)

`_clear_preview` (`app.py:329`) and `_clear_review` (`app.py:341`) both clear a displayed image by passing `image=None` to `CTkLabel.configure`. Whether customtkinter removes the underlying Tk image or retains the previous one varies across versions, and the repo pins a range (`customtkinter>=6.0.0,<7`) rather than an exact version. If it retains, the previous run's last page stays visible behind a new job's cleared panel. Confirming requires running the GUI, so this is an `open question` rather than an asserted defect; recorded as `q-ctk-image-clear`. **Action:** in a port, clear by assigning an explicit empty image or by hiding the widget, not by passing null.

### D1.5 — Review index is positional and can be shifted by out-of-order arrivals (low)

`_review_index` indexes into `_review_order`, and `_register_review_page` inserts with `bisect.insort` (`app.py:355`). If a page ever arrives with a number lower than one already registered, the insert lands *before* the user's current position and `_review_index` silently now points at a different page — the displayed content changes under the user without navigation. This cannot happen today: `recognize_images` iterates `enumerate(image_paths, start=1)` strictly in order (`ocr_service.py:178`), so insort always appends. The defect is latent, and it becomes live the moment a port introduces concurrent page recognition — a plausible optimization given each page is an independent request. **Action:** track the current page *number*, not its list position, and recompute the index after each insert.

---

## Pass 2: Error Handling and Resilience

| # | Location | Defect | Severity | Evidence Level | Action |
|---|----------|--------|----------|----------------|--------|
| D2.1 | `ocr_service.py:158-236` `recognize_images` | One failed page aborts the whole document and discards every page already recognized; no retry, no partial save. | high | observed fact | fix before porting |
| D2.2 | `ocr_service.py:224-229` `recognize_images` | A legitimately blank page is treated as a fatal error, aborting the document. | high | observed fact | fix before porting |
| D2.3 | `app.py:743-752` `on_close` + `ocr_service.py:302-366` `process_ocr` | Shutdown neither joins nor cancels the daemon worker; temp-dir cleanup is not guaranteed to run. **Closes `arch-CF1`.** | high | strong inference | fix before porting |
| D2.4 | `ocr_service.py:239-263` `save_markdown_atomic` | No `fsync` before `os.replace`, and the `delete=False` temp file lives in the user's document folder. | medium | observed fact | fix before porting |
| D2.5 | `app.py:59`, `app.py:308-315` | Unbounded event queue and never-evicted per-page PNG bytes. **Closes `arch-CF2`.** | medium | strong inference | fix before porting |
| D2.6 | `app.py:247-261` `drain_ui_events` | A raising handler discards the events already dequeued in that pass; losing a terminal event strands the UI in its busy state. | medium | strong inference | fix before porting |
| D2.7 | `ocr_service.py:366` | `shutil.rmtree(..., ignore_errors=True)` silently swallows cleanup failures. | low | observed fact | port differently |
| D2.8 | `ocr_service.py:295` `reveal_in_file_manager` | Windows branch omits `check=True`, making genuine failure indistinguishable from success. | low | observed fact | port differently |
| D2.9 | `app.py:287-291` `append_log` | Diagnostics exist only in an in-memory Tk textbox; a crash loses them entirely. | low | observed fact | port differently |

### D2.1 — A single page failure destroys the entire job's work (high)

`recognize_images` wraps each page's `client.chat` in `try/except Exception` and **re-raises immediately** as `OCRServiceError` (`ocr_service.py:219-223`). That propagates out of `process_ocr` before `save_markdown_atomic` is ever reached (`ocr_service.py:352-362`), so nothing is written. There is no retry, no backoff, and no partial-result path anywhere in the module.

The consequence scales with document size: on a 200-page PDF, a transient network blip, a momentary Ollama restart, or one 120-second idle timeout on page 199 discards 198 successfully recognized pages. The user's only recovery is to re-run the entire document from page 1 — paying the full inference cost again. Given the architecture map's finding that per-page inference dominates wall-clock, this is the highest-cost failure mode in the system. **Action:** `fix before porting`. A port should retry each page with bounded attempts and backoff, and on unrecoverable failure still save the pages it has (with an explicit marker for the failed page) rather than discarding everything.

### D2.2 — A blank page is a fatal error (high)

```python
content = "".join(chunks).strip()
if not content:
    raise OCRServiceError(f"Ollama returned no text for page {number}/{total} ...")
```
(`ocr_service.py:224-229`)

The check cannot distinguish "the model failed" from "this page is genuinely blank." Blank pages are ordinary in real documents — chapter versos, scanned separator sheets, intentionally empty back pages. Any one of them aborts the whole document via the D2.1 path. The README frames the empty-output case purely as a symptom of a non-vision model ("Empty or garbage output / 'returned no text' → the selected model has no vision support"), which shows the blank-page case was not considered. **Action:** `fix before porting`. Treat an empty page as an empty string in the output with a warning, and reserve hard failure for a transport-level error.

### D2.3 — Shutdown abandons the worker without cleanup or cancellation (high) — closes `arch-CF1`

`on_close` sets `self.closing = True` and calls `self.destroy()` without joining or signalling the OCR worker (`app.py:743-752`). Workers are started with `daemon=True` (`app.py:654-656`, `app.py:493-495`). After `destroy()` ends the mainloop, `main()` returns and the interpreter shuts down; CPython does not run `finally` blocks in daemon threads at that point. The cleanup that removes the render directory lives exactly there:

```python
finally:
    if temp_dir is not None:
        shutil.rmtree(temp_dir, ignore_errors=True)
```
(`ocr_service.py:364-366`)

So quitting mid-job can leave a `local_ocr_*` directory holding a PNG for **every page rendered so far** — at 300 DPI, potentially gigabytes — with nothing to ever clean it up but the OS temp reaper. `drain_ui_events` also returns early once `closing` is set (`app.py:248-249`), so any diagnostic the worker emits on its way out is discarded.

Compounding this, **there is no cancellation mechanism at all**: no stop button, no `threading.Event`, no check of a cancel flag inside `render_pdf` or `recognize_images`. A user who starts a 500-page job at the wrong DPI can only quit the app (leaking the temp dir) or wait it out. The confirmation dialog ("An operation is still running. Close anyway?") asks the user to accept a consequence the app then handles badly.

Whether the `finally` runs is timing-dependent, which is why the residual runtime question `q-shutdown-tempdir` stays open; the *code hazard* — no join, no signal, no cancellation — is an observed structural fact and is what this finding closes. **Action:** `fix before porting`. A port needs a cooperative cancellation token checked between pages, and cleanup owned by the shutdown path rather than by a `finally` in an abandoned thread.

### D2.4 — "Atomic" save is not durable, and its temp file lands in the user's folder (medium)

Two distinct gaps in `save_markdown_atomic` (`ocr_service.py:239-263`):

1. **No `fsync`.** The code calls `tmp_file.flush()` (`ocr_service.py:255`) then `os.replace` (`ocr_service.py:256`). `flush()` pushes to the OS page cache, not to disk. The rename is atomic against *process* death, but on power loss or kernel panic the result can be a zero-length or truncated file at the destination. The README's claim — "The file is written atomically, so a failed run never leaves a partial result" (`README.md:82-84`) — is therefore true for the failure mode the code guards and overstated for the one users usually mean by "crash."
2. **Temp file in the user's document directory.** `NamedTemporaryFile(..., delete=False, dir=output_path.parent, prefix=f".{output_path.stem}_", suffix=".tmp")` deliberately writes beside the input so the rename stays same-filesystem — correct for atomicity — but with `delete=False` and cleanup only in the `except` branch (`ocr_service.py:258-262`), a hard kill between creation and `os.replace` leaves a hidden `.report_*.tmp` file permanently in the user's folder. Combined with D2.3 (quit mid-job), this is reachable in normal use.

**Action:** `fix before porting`. Add `os.fsync(tmp_file.fileno())` before the close, and sweep stale `.{stem}_*.tmp` siblings on startup or before writing.

### D2.5 — Unbounded queue and unbounded retained page bytes (medium) — closes `arch-CF2`

Two unbounded growth paths, both driven by document length rather than by anything the app controls:

1. **The event queue.** `self.event_queue: queue.Queue = queue.Queue()` (`app.py:59`) has no `maxsize`, so `put` never blocks. The producer is a streaming model emitting one `stream_chunk` event per token delta (`ocr_service.py:214-218`); the consumer drains every 50 ms (`config.py:41`). A fast model on a local GPU can emit deltas faster than the drain retires them, and nothing applies backpressure — the worker never slows down.
2. **Retained thumbnail bytes.** `on_page_image` stores the full PNG for every page in `review_pages` (`app.py:313-315`) and nothing ever evicts. The 5-entry LRU at `app.py:376-377` bounds *decoded* `CTkImage` objects only — the raw `bytes` are a separate, unbounded dict. At `THUMBNAIL_MAX_SIDE = 900`, a text page is roughly 100-300 KB, so a 500-page document retains on the order of 50-150 MB for the life of the run, on top of the queue.

Neither is a leak in the strict sense — both are freed when the run's state is cleared by `_clear_review` on the next job (`app.py:335-340`) — but both are unbounded in document length, which is exactly the dimension users scale. **Action:** `fix before porting`. Give the queue a `maxsize` so a saturated UI applies real backpressure, and either evict page bytes on an LRU alongside the decoded images or spool them to the (already existing) temp directory.

### D2.6 — A raising handler drops the events already dequeued (medium)

`drain_ui_events` wraps only the *reschedule* in `finally`, not the dispatch:

```python
try:
    while True:
        try:
            kind, payload = self.event_queue.get_nowait()
        except queue.Empty:
            break
        self.handle_event(kind, payload)          # not individually guarded
finally:
    self.after(config.UI_POLL_INTERVAL_MS, self.drain_ui_events)
```
(`app.py:250-261`)

The comment explains the `finally` correctly — a broken `after()` chain would stop all event processing — but the guard is incomplete. If `handle_event` raises, the `while` loop is abandoned: the event being handled is lost, and so is any event already in the queue that this pass would have drained. The pump survives, but the exception surfaces only as a Tk traceback on stderr, invisible to a user running the app from a launcher.

The worst case is narrow but severe: if the lost event is the terminal `ocr_success` or `ocr_error`, `_restore_idle` never runs (`app.py:672-677`, `app.py:734-739`) and every control stays disabled with `Start OCR` reading "Processing, please wait..." forever. Because exactly one terminal event is ever enqueued (`app.py:658-670`), there is no retry — the only recovery is restarting the app. Plausible triggers are limited (`on_page_image` decoding bytes that `make_thumbnail_png` already produced successfully is low-risk), which is why this is medium rather than high. **Action:** `fix before porting`. Wrap each `handle_event` call in its own `try/except`, log the failure into the Log tab, and continue the drain loop.

### D2.7 — Cleanup failures are silently swallowed (low)

`shutil.rmtree(temp_dir, ignore_errors=True)` (`ocr_service.py:366`) discards every error. On Windows, a still-open handle on a rendered PNG makes removal fail routinely, and the user is never told that a directory of page images was left behind. The `ignore_errors=True` choice is defensible — cleanup must not mask the real exception in a `finally` — but the information should not vanish. **Action:** use `onerror`/`onexc` to log what could not be removed.

### D2.8 — Windows reveal cannot report failure (low)

```python
subprocess.run(["explorer", f"/select,{path}"])   # ocr_service.py:295
```

The comment correctly notes that `explorer` returns exit code 1 even on success, so `check=True` is unusable — but the consequence is that a real failure (path gone, Explorer unavailable) is indistinguishable from success and produces no error dialog. The `Show in Explorer` button silently does nothing. The macOS and Linux branches do use `check=True` (`ocr_service.py:292`, `ocr_service.py:297`), so error reporting is inconsistent across platforms. **Action:** verify the path exists before invoking, and report that failure directly.

### D2.9 — No persistent diagnostics (low)

`append_log` writes only into the Tk textbox (`app.py:287-291`); nothing is ever written to a file, and the architecture map confirms no logging configuration exists. Every message describing what happened — including the `[Error]` line for a failed run — is destroyed when the window closes. A user reporting a bug has nothing to attach unless they think to copy the tab's contents before quitting. **Action:** `port differently` — mirror log lines to a rotating file in a per-user data directory.

---

## Pass 6: Configuration and Environment Hazards

| # | Location | Defect | Severity | Evidence Level | Action |
|---|----------|--------|----------|----------------|--------|
| D6.1 | `config.py` (whole module), `app.py:109,122,136` | Every setting is a source constant; no config file, no env override, and user choices are discarded on exit. | medium | observed fact | port differently |
| D6.2 | `requirements.txt` | Inconsistent pinning — two dependencies capped, two open-ended, against APIs the code already treats as unstable. | medium | observed fact | fix before porting |
| D6.3 | `ocr_service.py:326`, `ocr_service.py:121-137` | All pages are rendered up front into system temp with no free-space check. | medium | strong inference | fix before porting |
| D6.4 | `README.md:9-14`, repo root | Python 3.10+ requirement is documented but enforced nowhere. | low | observed fact | port differently |
| D6.5 | `config.py:7` `EXAMPLE_MODELS` | Combobox seeded with model tags that are not guaranteed to exist. | low | observed fact | leave behind |
| D6.6 | `app.py:571-574` `copy_result` | Tk owns the clipboard; on X11 copied text vanishes when the app exits. | low | observed fact | port differently |
| D6.7 | `ocr_service.py:278,297` | Non-macOS/Windows branches assume `xdg-open` is on PATH. | low | observed fact | leave behind |

### D6.1 — No configuration surface at all, and no persistence (medium)

`config.py` is 44 lines of module-level constants with **zero** `os.environ` / `os.getenv` reads anywhere in first-party code, and no config file is read at startup (confirmed in the architecture map's Durable State table). Consequences:

- Changing `DEFAULT_OLLAMA_URL` (`config.py:3`), `MODEL_LIST_TIMEOUT`, `OCR_STREAM_IDLE_TIMEOUT`, `DPI_OPTIONS`, or either prompt requires editing source in an installed tree.
- The two prompt strings (`config.py:34-44`) drive output quality more than any other setting, and are the least accessible thing in the app.
- **User choices are not persisted.** Every launch resets the URL to `http://localhost:11434` (`app.py:109`), the model box to empty (`app.py:122`), and DPI to 150 (`app.py:136`). A user with Ollama on `http://192.168.1.50:11434` and a preferred model retypes both on every single launch.

The magic numbers themselves are *well handled* — centralized, named, and each carrying a rationale comment (`config.py:20-41`); this finding is about the absence of an override path, not about the values. **Action:** `port differently` — load defaults from a user config file and persist last-used URL/model/DPI.

### D6.2 — Inconsistent dependency pinning against unstable APIs (medium)

```
customtkinter>=6.0.0,<7      # capped
Pillow>=10,<12               # capped
PyMuPDF>=1.24.0              # open-ended
ollama>=0.4.0                # open-ended
```

The two uncapped dependencies are the two whose APIs the code is most exposed to. `ocr_service.py` already reads Ollama responses through defensive `getattr` chains — `getattr(item, "model", None)` (`ocr_service.py:91`) and `getattr(getattr(chunk, "message", None), "content", None)` (`ocr_service.py:210-211`) — which is direct evidence that response-shape drift is a known concern; yet nothing prevents `pip install` from resolving a future major version. PyMuPDF is used through `pymupdf.open`, `document.needs_pass`, `page.get_pixmap(dpi=..., colorspace=..., alpha=...)`, and `pymupdf.csRGB` (`ocr_service.py:109-132`) — a wide surface for a library with no upper bound. The project has no lockfile and no CI to catch a breaking release. **Action:** `fix before porting` — cap both, and record the tested versions.

### D6.3 — Eager full-document rasterization into unbounded temp space (medium)

`render_pdf` loops over **every** page and writes a PNG before `recognize_images` is called at all (`ocr_service.py:121-137`, sequenced at `ocr_service.py:331-352`). No free-space check precedes `tempfile.mkdtemp` (`ocr_service.py:326`). At 300 DPI, an A4 RGB page is roughly 2-8 MB as PNG, so a 500-page document needs 1-4 GB of temp space up front. Where `/tmp` is a tmpfs sized to a fraction of RAM — the default on many Linux distributions — this fails partway through rendering, after the user has already waited through most of it, and surfaces as a generic `Failed to render page N/M` (`ocr_service.py:133-136`). The eager design is also why D2.3's leak is expensive. **Action:** `fix before porting` — render lazily one page ahead of recognition, deleting each PNG once its page is recognized. That bounds temp usage to a constant and shrinks the abandoned-cleanup blast radius.

### D6.4 — Documented runtime floor is unenforced (low)

`README.md:9-11` requires "Python 3.10 or newer with Tkinter support," but there is no `pyproject.toml`, no `setup.py`, and therefore no `python_requires` gate, and no runtime version check in `main.py`. On an older interpreter the failure is an obscure syntax or typing error deep in a module rather than a clear message. Note the code may in fact tolerate 3.9 — every PEP 604 annotation is protected by `from __future__ import annotations` (`app.py:8`, `ocr_service.py:7`) — so the true floor is untested in either direction. **Action:** `port differently` — assert the interpreter version at entry, or declare it in packaging metadata.

### D6.5 — Non-existent model tags seeded into the picker (low)

`EXAMPLE_MODELS = ["gemma4:12b", "qwen3.6:27b"]` (`config.py:7`) populates the combobox's dropdown (`app.py:119-121`). Neither tag is guaranteed to exist on any server, and a user who picks one gets a `model not found` error from Ollama. This is well mitigated and honestly documented: the constant carries a comment saying the tags are never assumed to exist and never pulled, `app.py:122` calls `set("")` so nothing is preselected, and the README repeats the caveat. Recorded for completeness only. **Action:** `leave behind` — a port should populate the picker from `Refresh Models` alone.

### D6.6 — Clipboard content does not outlive the process (low)

`copy_result` uses Tk's `clipboard_clear` / `clipboard_append` (`app.py:573-574`). Under X11 the clipboard is owned by the source application, so on most Linux desktops (without a clipboard manager) the copied Markdown disappears the moment Local OCR exits. macOS and Windows use a system-owned clipboard and are unaffected — so the `Copy` button silently behaves differently per platform. **Action:** `port differently` — use a platform clipboard API with ownership transfer.

### D6.7 — `xdg-open` assumed present on non-macOS, non-Windows (low)

Both OS-integration helpers fall through to `xdg-open` (`ocr_service.py:278`, `ocr_service.py:297`), which is not guaranteed on minimal or headless Linux installs. The failure is contained — `FileNotFoundError` is caught and wrapped in `OCRServiceError` (`ocr_service.py:279-280`, `ocr_service.py:298-299`), then surfaced through `_run_file_action`'s dialog (`app.py:726-732`) — so this degrades to an error message rather than a crash. **Action:** `leave behind`; the fallback chain is a target-platform concern for the port.

---

## Summary

### Findings by Severity

| Severity | Count |
|----------|-------|
| Critical | 0 |
| High | 3 |
| Medium | 7 |
| Low | 11 |
| **Total** | **21** |

### Findings by Pass

| Pass | Critical | High | Medium | Low | Total |
|------|----------|------|--------|-----|-------|
| 1. Logic and correctness | 0 | 0 | 1 | 4 | 5 |
| 2. Error handling | 0 | 3 | 3 | 3 | 9 |
| 6. Config and environment | 0 | 0 | 3 | 4 | 7 |
| **Total** | **0** | **3** | **7** | **11** | **21** |

No critical findings: nothing in the mechanical passes produces silently wrong OCR output or destroys user data in normal operation. The three high findings all concern **losing completed work** rather than corrupting it.

### Top Findings

1. **D2.1** (pass 2, `ocr_service.py:158-236`) — one failed page discards every page already recognized; no retry, no partial save. `high` → `fix before porting`.
2. **D2.2** (pass 2, `ocr_service.py:224-229`) — a legitimately blank page is fatal, aborting the document via the D2.1 path. `high` → `fix before porting`.
3. **D2.3** (pass 2, `app.py:743-752` + `ocr_service.py:302-366`) — no join, no cancellation, cleanup `finally` unreachable at interpreter exit; leaks a full page-render directory. `high` → `fix before porting`.
4. **D6.3** (pass 6, `ocr_service.py:121-137,326`) — eager whole-document rasterization with no free-space check; gigabytes of temp on large PDFs and the reason D2.3 is expensive. `medium` → `fix before porting`.
5. **D2.6** (pass 2, `app.py:247-261`) — an unguarded handler drops queued events; losing a terminal event strands the UI permanently in its busy state. `medium` → `fix before porting`.

The first three share a root cause worth carrying into `porting`: **the pipeline has no notion of partial success.** It is written as one all-or-nothing transaction over N independent network calls, which is the wrong shape for the work it does.

### Routed To Semantic Phase

| ID | Description | Why Routed |
|----|-------------|-----------|
| `mech-CF1` | TOCTOU windows: `validate_input_path` runs at `app.py:606` and the overwrite check at `app.py:633`, both well before the worker reads the input (`ocr_service.py:331`) or writes the output (`ocr_service.py:362`). The file can be replaced, deleted, or created in between. | Time-of-check/time-of-use is a trust-boundary and race concern — pass 3/4 in `defect-scan-semantic`, not a local logic defect. |
| `mech-CF2` | Transport and trust: `normalize_ollama_url` accepts any `http://` host (`ocr_service.py:55`), the default is plaintext, Ollama has no authentication, and full document page images are transmitted to whatever host is entered. The README warns about exposure but the code enforces nothing. | Auth gaps, trust boundaries, and transport security are pass 4's rubric and need the contracts phase's security model for grounding. |
| `mech-CF3` | The `OperationState` mutual exclusion (`app.py:461,483,596`) uses no lock; correctness rests entirely on the unverified invariant that all three guards execute on the Tk main thread. Also `self.closing` is read by `drain_ui_events` (`app.py:248`) and written by `on_close` (`app.py:745`) with no synchronization. | Lock-free invariant verification and cross-thread visibility are pass 3's rubric and need the protocols phase's state machine to check exhaustively. |
| `mech-CF4` | Documentation-versus-implementation drift: the README's atomicity claim (`README.md:82-84`) overstates the guarantee (see D2.4), and its troubleshooting table attributes "returned no text" solely to a non-vision model, omitting the blank-page case (see D2.2). | Spec-versus-implementation drift is pass 5's rubric and requires the contracts phase's recovered contracts to compare against systematically. |

---

## Coverage and limits

- **Inspected scope:** all four first-party modules read in full for this pass (`main.py`, `config.py`, `ocr_service.py`, `app.py` — 1,178 lines), each against the three pass checklists (`01-logic-and-correctness.md`, `02-error-handling.md`, `06-config-and-environment.md`) in that order. `requirements.txt`, `.gitignore`, and `README.md` were re-read specifically for pass 6. Both routed items (`arch-CF1`, `arch-CF2`) were addressed and closed.
- **Skipped scope:** passes 3 (concurrency), 4 (security), and 5 (API contract violations) are out of scope by pipeline design and run in `defect-scan-semantic`; four items were routed there rather than judged here. Test bodies in `tests/` (~1,540 lines) were not read — they are the contracts phase's evidence source (`arch-CF4`), and reading them here would have blurred the mechanical/behavioral boundary. Third-party internals (`customtkinter`, `pymupdf`, `ollama`, `PIL`) were not audited.
- **Evidence basis:** source inspection, plus upstream findings from `findings/architecture/architecture-map.md` for the layer map, concurrency model, and durable-state inventory. Project documentation (`README.md`) was used as the stated-intent baseline for pass 6 and for D2.4/D2.2's drift observations.
- **Known blind spots:** (1) **no runtime verification** — the app was never launched, no Ollama server was contacted, and the test suite was not executed, so every timing-dependent conclusion (D2.3's `finally` behavior, D2.6's stuck-state) is reasoned rather than observed; (2) customtkinter widget semantics are assumed from its public API, which is why D1.4 is an `open question`; (3) memory figures in D2.5 and disk figures in D6.3 are order-of-magnitude estimates from typical PNG sizes, not measurements; (4) whether CPython runs the daemon thread's `finally` at exit is timing- and interpreter-dependent — the *code hazard* is asserted, the *observed outcome* remains open as `q-shutdown-tempdir`; (5) gitignored files (`build-mac.sh`, `docs/`) are absent from the clone and could carry configuration this pass could not see.
- **Coverage disposition:** COMPLETE for the mechanical scope. All three assigned passes ran to completion over the entire first-party source; gaps are either deliberate deferrals to the semantic phase (4 routed items) or the open questions listed above.

## Validation

| # | Criterion | Result | Evidence |
|---|-----------|--------|----------|
| 1 | At least two of the three mechanical passes (1, 2, 6) produced findings or documented "no defects found." | PASS | All three passes produced findings: pass 1 → 5, pass 2 → 9, pass 6 → 7. |
| 2 | Each finding has location, severity, evidence level, and recommended action. | PASS | Every row of the three pass tables carries all four columns (file:line location, severity, evidence level, pre-porting action), and each has a prose subsection expanding the evidence. |
| 3 | Findings are organized by pass and sorted by severity. | PASS | One section per pass; within each, rows descend high → medium → low (pass 1 has no high; passes 1 and 6 have no critical or high). |
| 4 | Summary tables are complete and counts match the detailed findings. | PASS | §Findings by Severity totals 21 (0/3/7/11); §Findings by Pass rows total 5 + 9 + 7 = 21 and column sums are 0/3/7/11. Both reconcile with the 21 detailed entries D1.1–D1.5, D2.1–D2.9, D6.1–D6.7. |
| 5 | Items spotted that are actually semantic in nature are routed onward via a carry_forward entry in the phase handoff targeting defect-scan-semantic. | PASS | `mech-CF1`–`mech-CF4` documented in §Routed To Semantic Phase and recorded as `carry_forward` entries with `target_phase: defect-scan-semantic` in `scratch/handoffs/defect-scan-mechanical.yaml`. |
| 6 | Findings are marked with evidence levels. | PASS | Each finding carries `observed fact` (16), `strong inference` (4), or `open question` (1 — D1.4, tracked as `q-ctk-image-clear`). |
| 7 | Coverage and limits name inspected scope, skipped scope, evidence basis, and blind spots. | PASS | §Coverage and limits names all four plus a COMPLETE disposition; the absence of runtime verification is stated as blind spot (1). |

**Validated by:** 2026-08-16 (defect-scan-mechanical phase, MCP-driven session 1)
**Overall:** PASS
