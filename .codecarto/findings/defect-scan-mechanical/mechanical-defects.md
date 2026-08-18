# Mechanical Defects Report — Local-OCR

## Scan Context

- **Source:** `../` (repository root), commit `8e7388c`
- **Architecture reference:** `findings/architecture/architecture-map.md`
- **Pipeline:** `full-with-deep-audit` · **Framework:** v0.16.0 · **Date:** 2026-08-18
- **Scope:** Mechanical passes only (1 logic, 2 error handling, 6 configuration). Semantic passes (3 concurrency, 4 security, 5 contract violations) deferred to `defect-scan-semantic`.
- **Action set:** pre-porting (`fix before porting` / `port differently` / `leave behind`).
- **Routed items closed here:** `arch-CF1` (D2.3), `arch-CF2` (D2.5), `arch-CF6` (D1.6).
- **Conventions honored:** C01 (execute the headless layer) — two probes were run rather than reasoned about, resolving one open question and upgrading one finding to `observed fact`. C02 (a test's existence is not evidence it runs) — governs how D1.6 and D6.2 are stated.

### Orchestrator duties discharged

**Open-question re-triage.** Both inherited labels were re-tested rather than accepted:

- `q-shutdown-tempdir` (was `needs-runtime-test`) — **now answerable, and answered.** Its Tk half needs a GUI this host lacks, but the *load-bearing mechanism* is toolkit-independent: does a daemon thread's `finally` run when the main thread returns? A 15-line probe reproducing `process_ocr`'s shape (`mkdtemp` … `finally: rmtree`) shows it **does not** — the directory survived interpreter exit on Python 3.11. **Closed as `observed fact`** via `open_question_closures`; see D2.3.
- `q-test-invocation` (`needs-maintainer-decision`) — label re-tested and **confirmed correct**. A working command is established, so nothing is blocked; the residue is genuinely a maintainer preference (`.gitignore` implies pytest, the tests are `unittest`). Retained unchanged.

**Contradiction sweep.** One contradiction found between measured fact and a summarized claim, and it is routed rather than smoothed: the architecture phase's own earlier reading held that the GUI suites "self-skip without a display." Execution shows they **error at import** on a host without `tkinter`. This was already routed as `arch-CF6` and is closed here as D1.6. No other required-read claim conflicts with measurement; notably, D6.2's uncapped-dependency risk is *tempered* by measurement (the suite passes on PyMuPDF 1.28.2) rather than contradicted.

---

## Pass 1: Logic and Correctness

| # | Location | Defect | Severity | Evidence Level | Action |
|---|----------|--------|----------|----------------|--------|
| D1.6 | `tests/test_*_gui.py:1-40` (all five) | The headless-skip guard sits **after** the import it must guard, so the suites error instead of skipping. **Closes `arch-CF6`.** | medium | observed fact | fix before porting |
| D1.1 | `app.py:505-524` `on_progress` | Progress reports a page as done when it has only *started*; a single-image job sits at 100% for its whole duration. | medium | observed fact | port differently |
| D1.2 | `app.py:526-548` `on_page_text` | The non-streaming fallback append is unreachable in production — dead defensive code. | low | strong inference | leave behind |
| D1.3 | `app.py:421-425` `_apply_refresh_busy_state` | Three controls stay visually enabled during a model refresh but silently no-op. | low | observed fact | port differently |
| D1.4 | `app.py:329`, `341` | Image clearing relies on `CTkLabel.configure(image=None)`, whose clearing behavior is version-dependent. | low | open question | port differently |
| D1.5 | `app.py:349-359` `_register_review_page` | `bisect.insort` can shift the page the user is viewing; safe only because pages arrive in ascending order. | low | strong inference | port differently |

### D1.6 — A guard placed after the thing it guards (medium) — closes `arch-CF6`

All five GUI test files carry a docstring promising graceful degradation: *"They are skipped automatically when no display is available (CI, headless containers)."* Each backs it with a real guard in `setUp`:

```python
try:
    self.app = app_module.LocalOCRApp()
    self.app.withdraw()
except Exception:
    self.skipTest("No display available — skipping GUI smoke tests.")
```

But every file also does `import app as app_module` at **module scope** (`tests/test_review_gui.py:14` and equivalents), and `app.py:18` does `from tkinter import filedialog, messagebox`. On a host where `tkinter` is absent — the exact "headless container" case the docstring names — the import raises `ModuleNotFoundError` during collection, before any test class is instantiated and before `setUp` can skip.

**Measured** (`observed fact`, this session): `python3 -m unittest discover -s tests -t .` → `Ran 77 tests … FAILED (errors=5, skipped=1)`, while `python3 -m unittest tests.test_ocr_service` → `OK (skipped=1)` with 72 passing. The service suite is healthy; the GUI suites take the whole run red.

Two consequences worth separating. First, the documented behavior is wrong in the case it was written for, so anyone wiring up CI sees a red build and may conclude the project is broken. Second, and more corrosive: because the failure is at collection, **these ~470 lines of UI contracts have very likely never executed anywhere without a display** — which is precisely why C02 now requires distinguishing "a test exists" from "a test runs." The guard catches a *missing display* (where `tkinter` imports fine but `Tk()` fails); it cannot catch a *missing toolkit*.

**Action:** `fix before porting`. Guard the import itself — `importlib.util.find_spec("tkinter")` plus a module-level `skipUnless`, or move the `import app` inside `setUp` behind the existing `try`. A port must place the guard before the dependency it protects.

### D1.1 — Progress fraction is off by one page (medium)

`progress_callback("ocr", number, total)` fires at the **top** of each iteration, before the page is sent (`ocr_service.py:179-180`; same for `"render"` at `123-124`), but the handler treats `current` as work completed: `fraction = 0.2 + 0.8 * current / total` (`app.py:515`). With `current == total` the bar sits at 100% for the whole final page. For a **single image** it is starker — `_render_phase_seen` is `False`, so `fraction = 1/1 = 1.0` the instant recognition begins (`app.py:517`), and the bar is full for the entire job while the `Page 1 / 1` label looks correct. The app appears hung. Note this formula is *asserted by a passing test*, so it is a deliberate choice with a poor consequence, not an accident — the contracts phase will confirm from the test side. **Action:** compute from completed work (`(current - 1) / total`), or emit progress after each page.

### D1.2 — Unreachable fallback branch (low)

`on_page_text` guards `if page == self._result_page: return` and otherwise appends (`app.py:543-548`). That append is unreachable: `recognize_images` raises when the assembled text is empty (`ocr_service.py:224-229`), so any page reaching a `page_text` event emitted at least one non-empty delta, which already set `_result_page` in `on_stream_chunk`. The docstring acknowledges this. Recorded because it is genuinely dead on every production path and a port must not treat it as required. **Action:** `leave behind` unless the port adds a non-streaming mode, in which case it becomes live and needs its own test.

### D1.3 — Controls enabled but inert during refresh (low)

`_apply_refresh_busy_state` disables only `url_entry`, `refresh_button`, and `start_button` (`app.py:421-425`), while `_apply_ocr_busy_state` also disables `select_button`, the model box, and the DPI box. During a refresh, `Select File` renders enabled but `select_file` returns immediately on its state guard (`app.py:461-462`) with no dialog and no log line. `_restore_idle` then re-enables `select_button` unconditionally, confirming the asymmetry was unintentional. **Action:** derive enablement from one state→control mapping.

### D1.4 — Toolkit-specific null-image clearing (low, open question)

`_clear_preview` (`app.py:329`) and `_clear_review` (`app.py:341`) clear a displayed image by passing `image=None` to `CTkLabel.configure`. Whether customtkinter removes the underlying Tk image or retains the previous one varies by version, and the repo pins a range (`>=6.0.0,<7`) rather than an exact version. If it retains, the previous run's last page stays visible behind a cleared panel. Unverifiable here — this host has no `tkinter`, so C01 does not apply and the label stands. Tracked as `q-ctk-image-clear`. **Action:** clear by assigning an explicit empty image or hiding the widget.

### D1.5 — Review index is positional and shiftable (low)

`_review_index` indexes into `_review_order`, and `_register_review_page` inserts with `bisect.insort` (`app.py:355`). A page arriving with a number lower than one already registered would land *before* the user's position, silently changing the displayed page without navigation. Impossible today — `recognize_images` iterates strictly in order (`ocr_service.py:178`) — but it becomes live the moment a port recognizes pages concurrently, which is a plausible optimization given each page is an independent request. **Action:** track the current page *number* and recompute the index after each insert.

---

## Pass 2: Error Handling and Resilience

| # | Location | Defect | Severity | Evidence Level | Action |
|---|----------|--------|----------|----------------|--------|
| D2.1 | `ocr_service.py:158-236` | One failed page aborts the document and discards every page already recognized; no retry, no partial save. | high | observed fact | fix before porting |
| D2.2 | `ocr_service.py:224-229` | A legitimately blank page is treated as fatal, aborting the document. | high | observed fact | fix before porting |
| D2.3 | `app.py:743-752` + `ocr_service.py:302-366` | Shutdown neither joins nor cancels the worker, and the cleanup `finally` **provably** does not run. **Closes `arch-CF1`.** | high | observed fact | fix before porting |
| D2.4 | `ocr_service.py:239-263` | No `fsync` before rename, and the `delete=False` temp file lives in the user's document folder. | medium | observed fact | fix before porting |
| D2.5 | `app.py:59`, `308-315` | Unbounded event queue and never-evicted per-page PNG bytes. **Closes `arch-CF2`.** | medium | strong inference | fix before porting |
| D2.6 | `app.py:247-261` | A raising handler discards events already dequeued; losing a terminal event strands the UI. | medium | strong inference | fix before porting |
| D2.7 | `ocr_service.py:366` | `shutil.rmtree(..., ignore_errors=True)` silently swallows cleanup failures. | low | observed fact | port differently |
| D2.8 | `ocr_service.py:295` | Windows reveal omits `check=True`, so failure is indistinguishable from success. | low | observed fact | port differently |
| D2.9 | `app.py:287-291` | Diagnostics live only in an in-memory textbox; a crash loses them. | low | observed fact | port differently |
| D2.10 | `ocr_service.py:196-223` | A **local** missing-file error is reported as an Ollama request failure. | low | observed fact | port differently |

### D2.1 — A single page failure destroys the whole job's work (high)

Each page's `client.chat` is wrapped and **re-raised immediately** as `OCRServiceError` (`ocr_service.py:219-223`), which propagates out of `process_ocr` before `save_markdown_atomic` is ever reached (`352-362`). Nothing is written. There is no retry, no backoff, and no partial-result path anywhere.

The cost scales with document size: on a 200-page PDF, one transient blip, one Ollama restart, or one idle timeout on page 199 discards 198 successfully recognized pages, and the user must re-run from page 1 — paying the full inference cost again. Since per-page inference dominates wall-clock (architecture §Concurrency Model), this is the highest-cost failure mode in the system. **Action:** `fix before porting`. Retry each page with bounded attempts and backoff; on unrecoverable failure still persist the pages already recognized, with an explicit marker for the failed one.

### D2.2 — A blank page is a fatal error (high)

```python
content = "".join(chunks).strip()
if not content:
    raise OCRServiceError(f"Ollama returned no text for page {number}/{total} ...")
```
(`ocr_service.py:224-229`)

The check cannot distinguish "the model failed" from "this page is genuinely blank." Blank pages are ordinary in real documents — chapter versos, scanned separator sheets, empty back pages — and any one of them aborts the whole document through the D2.1 path. The README frames the empty-output case purely as a symptom of a non-vision model, which shows the blank-page case was not considered. **Action:** `fix before porting`. Treat an empty page as an empty section with a warning; reserve hard failure for transport-level errors.

### D2.3 — Shutdown abandons the worker, and the leak is proven (high) — closes `arch-CF1`

`on_close` sets `closing` and calls `destroy()` without joining or signalling the worker (`app.py:743-752`); workers are `daemon=True` (`app.py:654-656`, `493-495`). After `destroy()` ends the mainloop, `main()` returns and the interpreter shuts down. The cleanup that removes the render directory lives exactly there:

```python
finally:
    if temp_dir is not None:
        shutil.rmtree(temp_dir, ignore_errors=True)   # ocr_service.py:364-366
```

The v0.14.1 pass rated this `strong inference` and left `q-shutdown-tempdir` open. **Per C01 the mechanism was isolated and executed this run** — a probe reproducing exactly this shape (daemon thread: `mkdtemp` → long sleep → `finally: rmtree`; main thread returns after 0.4 s) on Python 3.11:

```
worker temp dir: /tmp/local_ocr_probe_w75e_g09
RESULT: dir SURVIVED -> finally did NOT run -> temp dir LEAKS
```

So the `finally` **does not run** and the directory **does** survive (`observed fact`). Quitting mid-job therefore leaves a `local_ocr_*` directory holding a PNG for every page rendered so far — at 300 DPI, potentially gigabytes — with nothing but the OS temp reaper to collect it. `drain_ui_events` also returns early once `closing` is set (`app.py:248-249`), discarding any diagnostic the worker emits on its way out.

Compounding this, **there is no cancellation mechanism at all**: no stop button, no `threading.Event`, no cancel check inside `render_pdf` or `recognize_images`. A user who starts a 500-page job at the wrong DPI can only quit — leaking — or wait it out. The confirmation dialog ("An operation is still running. Close anyway?") asks the user to accept a consequence the app then handles badly.

The residual unknown is only leak *size*, not leak *existence*: how much has been rendered when the user quits. `q-shutdown-tempdir` is closed. **Action:** `fix before porting`. Cooperative cancellation checked between pages, with cleanup owned by the shutdown path rather than by a `finally` in an abandoned thread.

### D2.4 — "Atomic" save is not durable, and its temp file lands in the user's folder (medium)

Two gaps in `save_markdown_atomic` (`ocr_service.py:239-263`). **No `fsync`:** the code calls `flush()` (`255`) then `os.replace` (`256`); `flush()` reaches the OS page cache, not the disk, so the rename is atomic against *process* death but a power loss can leave a zero-length or truncated file at the destination. The README's "written atomically, so a failed run never leaves a partial result" (`README.md:82-84`) is true for the guarded failure class and overstated for the one users mean by "crash." **Temp file in the user's directory:** `NamedTemporaryFile(..., delete=False, dir=output_path.parent, prefix=f".{output_path.stem}_", suffix=".tmp")` correctly keeps the rename same-filesystem, but with `delete=False` and cleanup only in the `except` branch, a hard kill between creation and `os.replace` leaves a hidden `.report_*.tmp` permanently beside the user's document. Combined with D2.3 — now proven — this is reachable in normal use. **Action:** `fsync` before close; sweep stale `.{stem}_*.tmp` siblings.

### D2.5 — Unbounded queue and unbounded retained bytes (medium) — closes `arch-CF2`

Two growth paths, both driven by document length. **The queue:** `queue.Queue()` with no `maxsize` (`app.py:59`), so `put` never blocks; the producer emits one `stream_chunk` per token delta (`ocr_service.py:214-218`) while the consumer drains every 50 ms, and nothing applies backpressure. **Retained bytes:** `on_page_image` stores the full PNG for every page in `review_pages` (`app.py:313-315`) and nothing evicts; the 5-entry LRU (`376-377`) bounds only *decoded* `CTkImage` objects. At `THUMBNAIL_MAX_SIDE = 900` a text page is roughly 100–300 KB, so a 500-page document retains on the order of 50–150 MB for the run, atop the queue. Neither is a leak in the strict sense — both are freed by `_clear_review` on the next run — but both are unbounded in exactly the dimension users scale. **Action:** give the queue a `maxsize` so a saturated UI applies real backpressure; evict page bytes on an LRU or spool them to the temp directory that already exists.

### D2.6 — A raising handler drops events already dequeued (medium)

`drain_ui_events` wraps only the *reschedule* in `finally`, not the dispatch (`app.py:250-261`). The comment explains the `finally` correctly — a broken `after()` chain would stop all event processing — but the guard is incomplete: if `handle_event` raises, the `while` loop is abandoned, losing the event being handled and any others this pass would have drained. The pump survives, but the exception surfaces only as a Tk traceback on stderr, invisible to a user launching from a desktop icon. Worst case is narrow but severe: if the lost event is the terminal `ocr_success`/`ocr_error`, `_restore_idle` never runs and every control stays disabled with `Start OCR` reading "Processing, please wait..." forever — and since exactly one terminal event is ever enqueued, there is no retry. **Action:** wrap each `handle_event` call individually, log into the Log tab, continue the drain.

### D2.7 — Cleanup failures silently swallowed (low)

`shutil.rmtree(temp_dir, ignore_errors=True)` (`ocr_service.py:366`) discards every error. On Windows a still-open handle makes removal fail routinely and the user is never told a directory of page images was left behind. The choice is defensible — cleanup must not mask the real exception in a `finally` — but the information should not vanish, especially now that D2.3 proves this path is also skipped entirely on quit. **Action:** use `onerror`/`onexc` to log what could not be removed.

### D2.8 — Windows reveal cannot report failure (low)

`subprocess.run(["explorer", f"/select,{path}"])` (`ocr_service.py:295`) omits `check=True` because `explorer` returns 1 even on success — correctly reasoned, but the consequence is that a genuine failure is indistinguishable from success and the button silently does nothing. The macOS and Linux branches do use `check=True` (`292`, `297`), so error reporting is inconsistent across platforms. **Action:** verify the path exists before invoking and report that failure directly.

### D2.9 — No persistent diagnostics (low)

`append_log` writes only into the Tk textbox (`app.py:287-291`); nothing reaches a file. Every message describing what happened — including the `[Error]` line for a failed run — dies with the window. A user reporting a bug has nothing to attach unless they think to copy the tab before quitting. **Action:** mirror log lines to a rotating file in a per-user data directory.

### D2.10 — A local filesystem error is reported as a server failure (low)

`recognize_images` wraps everything inside its `try` as `Ollama request failed on page {n}/{N} (model {model!r}): {cause}` (`ocr_service.py:219-223`). But the page image is passed to the client as a **path string** (`204`), and the client resolves it locally: inspecting the installed `ollama` client shows `Image.serialize_model` reads the file and base64-encodes it, raising `ValueError(f'File {value} does not exist')` when a path with an image extension is absent (`observed fact`, this session — see the protocols phase for the full encoding finding).

So if the rendered PNG is missing at request time — the temp directory reaped mid-run, a filesystem error, an antivirus quarantine — the user is told their **Ollama request failed and shown their model name**, for what is entirely a local file problem. The error names the wrong subsystem and points debugging at the server. Low severity because the trigger is uncommon, but it is a real misattribution of blame, and it is the same weakness the semantic pass will find in a more consequential form. **Action:** stat the image before the request, or classify client-side `ValueError` separately from transport failure.

---

## Pass 6: Configuration and Environment Hazards

| # | Location | Defect | Severity | Evidence Level | Action |
|---|----------|--------|----------|----------------|--------|
| D6.1 | `config.py` (whole module), `app.py:109,122,136` | Every setting is a source constant; no config file, no env override, and user choices are discarded on exit. | medium | observed fact | port differently |
| D6.2 | `requirements.txt` | Inconsistent pinning and no lockfile, against APIs the code already treats as unstable. | medium | observed fact | fix before porting |
| D6.3 | `ocr_service.py:326`, `121-137` | All pages rendered up front into system temp with no free-space check. | medium | strong inference | fix before porting |
| D6.4 | `README.md:9-14`, repo root | The Python 3.10+ requirement is documented but enforced nowhere. | low | observed fact | port differently |
| D6.5 | `config.py:7` | Combobox seeded with model tags not guaranteed to exist. | low | observed fact | leave behind |
| D6.6 | `app.py:571-574` | Tk owns the clipboard; on X11 copied text vanishes when the app exits. | low | observed fact | port differently |
| D6.7 | `ocr_service.py:278,297` | Non-macOS/Windows branches assume `xdg-open` is on PATH. | low | observed fact | leave behind |

### D6.1 — No configuration surface, and no persistence (medium)

`config.py` is 44 lines of module constants with **zero** `os.environ`/`os.getenv` reads anywhere in first-party code, and no config file is read at startup. So changing `DEFAULT_OLLAMA_URL`, either timeout, the DPI set, or — most importantly — either prompt string requires editing source in an installed tree. The two prompts (`config.py:34-44`) drive output quality more than any other setting and are the least accessible thing in the app. And **user choices are not persisted**: every launch resets the URL to `http://localhost:11434` (`app.py:109`), the model box to empty (`122`), and DPI to 150 (`136`), so a user with Ollama on a LAN address and a preferred model retypes both every single launch. The constants themselves are *well handled* — centralized, named, each carrying a rationale comment — so this is about the absence of an override path, not the values. **Action:** load defaults from a user config file; persist last-used URL/model/DPI.

### D6.2 — Inconsistent pinning and no lockfile (medium)

```
customtkinter>=6.0.0,<7      # capped
Pillow>=10,<12               # capped
PyMuPDF>=1.24.0              # open-ended
ollama>=0.4.0                # open-ended
```

The two uncapped dependencies are the two whose APIs the code is most exposed to. `ocr_service` already reads Ollama responses through defensive `getattr` chains (`91`, `210-211`) — direct evidence that response-shape drift is a known concern — yet nothing prevents `pip` from resolving a future major. PyMuPDF is used through `open`, `needs_pass`, `get_pixmap(dpi=, colorspace=, alpha=)`, and `csRGB` (`109-132`), a wide surface for a library with no ceiling. There is no lockfile and no CI to catch a breaking release.

**Measured, and it tempers the rating** (`observed fact`, this session): the service suite passes on **PyMuPDF 1.28.2 and Pillow 11.3.0** — four minor versions above the declared floor — and the installed `ollama` client still exposes `ChatResponse.message.content` and `ListResponse.models[].model`. So the exposure is real but **currently unrealized**; drift has not yet broken anything. That is a reason to pin deliberately, not evidence that the risk is theoretical: the same resolution that produced a working 1.28.2 today produces an unreviewed version tomorrow. **Action:** cap both and commit a lockfile recording the tested versions.

### D6.3 — Eager rasterization into unbounded temp space (medium)

`render_pdf` writes a PNG for **every** page before `recognize_images` is called at all (`ocr_service.py:121-137`, sequenced at `331-352`), and no free-space check precedes `mkdtemp` (`326`). At 300 DPI an A4 RGB page is roughly 2–8 MB as PNG, so a 500-page document needs 1–4 GB of temp space up front. Where `/tmp` is a tmpfs sized to a fraction of RAM — the default on many Linux distributions — this fails partway through rendering, after the user has already waited through most of it, surfacing as a generic `Failed to render page N/M`. The eager design is also what makes D2.3's now-proven leak expensive. **Action:** render lazily one page ahead of recognition, deleting each PNG once consumed. Bounds temp usage to a constant and shrinks the abandoned-cleanup blast radius.

### D6.4 — Documented runtime floor unenforced (low)

`README.md:9-11` requires "Python 3.10 or newer with Tkinter support," but there is no `pyproject.toml`, no `setup.py`, therefore no `python_requires`, and no runtime version check in `main.py`. On an older interpreter the failure is an obscure error deep in a module rather than a clear message. The true floor is also untested in either direction — every PEP 604 annotation is protected by `from __future__ import annotations` (`app.py:8`, `ocr_service.py:7`), and the suite ran fine on 3.11 here. **Action:** assert the interpreter version at entry, or declare it in packaging metadata.

### D6.5 — Non-existent model tags seeded into the picker (low)

`EXAMPLE_MODELS = ["gemma4:12b", "qwen3.6:27b"]` (`config.py:7`) populates the dropdown (`app.py:119-121`). Neither tag is guaranteed to exist, and picking one yields a `model not found` error. Well mitigated and honestly documented: the constant carries a comment saying the tags are never assumed to exist, `app.py:122` calls `set("")` so nothing is preselected, and the README repeats the caveat. Recorded for completeness. **Action:** `leave behind`; populate from `Refresh Models` alone.

### D6.6 — Clipboard does not outlive the process (low)

`copy_result` uses Tk's `clipboard_clear`/`clipboard_append` (`app.py:573-574`). Under X11 the clipboard is owned by the source application, so on most Linux desktops without a clipboard manager the copied Markdown disappears when Local OCR exits. macOS and Windows use a system-owned clipboard and are unaffected — so `Copy` silently behaves differently per platform. **Action:** use a platform clipboard API with ownership transfer.

### D6.7 — `xdg-open` assumed present (low)

Both OS helpers fall through to `xdg-open` (`ocr_service.py:278`, `297`), not guaranteed on minimal or headless Linux. The failure is contained — `FileNotFoundError` is caught, wrapped, and surfaced through `_run_file_action`'s dialog — so it degrades to an error message rather than a crash. **Action:** `leave behind`; the fallback chain is a target-platform concern.

---

## Summary

### Findings by Severity

| Severity | Count |
|----------|-------|
| Critical | 0 |
| High | 3 |
| Medium | 8 |
| Low | 12 |
| **Total** | **23** |

### Findings by Pass

| Pass | Critical | High | Medium | Low | Total |
|------|----------|------|--------|-----|-------|
| 1. Logic and correctness | 0 | 0 | 2 | 4 | 6 |
| 2. Error handling | 0 | 3 | 3 | 4 | 10 |
| 6. Config and environment | 0 | 0 | 3 | 4 | 7 |
| **Total** | **0** | **3** | **8** | **12** | **23** |

No critical findings: nothing in the mechanical scope produces silently wrong OCR output or destroys user data in normal operation. All three high findings concern **losing completed work**.

### Top Findings

1. **D2.1** — one failed page discards every page already recognized; no retry, no partial save. `high` → `fix before porting`.
2. **D2.2** — a legitimately blank page is fatal, aborting the document via the D2.1 path. `high` → `fix before porting`.
3. **D2.3** — no join, no cancellation, and the cleanup `finally` **provably** does not run: quitting mid-job leaks a full page-render directory. `high` → `fix before porting`.
4. **D6.3** — eager whole-document rasterization with no free-space check; gigabytes of temp on large PDFs, and the reason D2.3's leak is expensive. `medium` → `fix before porting`.
5. **D1.6** — the headless-skip guard sits after the import it guards, so five suites error instead of skipping and ~470 lines of UI contracts have very likely never run. `medium` → `fix before porting`.

The first three share a root cause worth carrying into `porting`: **the pipeline has no notion of partial success.** It is written as one all-or-nothing transaction over N independent network calls, which is the wrong shape for the work it does.

### Routed To Semantic Phase

| ID | Description | Why Routed |
|----|-------------|-----------|
| `mech-CF1` | TOCTOU windows: `validate_input_path` runs at `app.py:606` and the overwrite check at `app.py:633`, both well before the worker reads the input (`ocr_service.py:331`) or writes the output (`362`). The file can be replaced, deleted, or created in between. | Time-of-check/time-of-use is a trust-boundary and race concern — pass 3/4 — not a local logic defect. |
| `mech-CF2` | Transport and trust: `normalize_ollama_url` accepts any `http://` host (`55`), the default is plaintext, Ollama has no authentication, and full page images are transmitted to whatever host is entered. The README warns; the code enforces nothing. | Auth gaps, trust boundaries, and transport security are pass 4's rubric and need the contracts phase's security model for grounding. |
| `mech-CF3` | The `OperationState` mutual exclusion (`app.py:461,483,596`) uses no lock; correctness rests entirely on the unverified invariant that all guards run on the Tk main thread. `self.closing` is likewise read and written without synchronization. | Lock-free invariant verification and cross-thread visibility are pass 3's rubric and need the protocols phase's state machine to check exhaustively. |
| `mech-CF4` | Documentation-vs-implementation drift: the README's atomicity claim (`82-84`) overstates the guarantee (D2.4), and its troubleshooting table attributes "returned no text" solely to a non-vision model, omitting the blank-page case (D2.2). | Spec-vs-implementation drift is pass 5's rubric and requires the contracts phase's recovered contracts to compare against systematically. |
| `mech-CF5` | The defensive `getattr` reads (`91`, `210-211`) currently match the installed client (measured), but if a field is renamed every delta silently reads empty and the failure surfaces as `returned no text` — which the README attributes to a non-vision model. D2.10 is the mild form of this; the severe form is total and misdiagnosed. | Provider-contract drift with a misleading diagnostic is pass 5's rubric, judged against the recovered contracts. |

---

## Coverage and limits

- **Inspected scope:** all four first-party modules read in full (1,178 lines) against the three pass checklists (`01-logic-and-correctness.md`, `02-error-handling.md`, `06-config-and-environment.md`) in order. `requirements.txt`, `.gitignore`, and `README.md` re-read for pass 6. All three routed items (`arch-CF1`, `arch-CF2`, `arch-CF6`) addressed and closed. **Two probes executed** per C01: the full test suite via both `unittest tests.test_ocr_service` and `unittest discover`, and an isolated daemon-thread `finally` probe.
- **Skipped scope:** passes 3 (concurrency), 4 (security), and 5 (API contract violations) are out of scope by pipeline design; five items routed to `defect-scan-semantic` rather than judged here. Test *bodies* were read only where needed to settle D1.1, D1.6, and D6.2 — systematic mining is the contracts phase's rubric (`arch-CF4`). Third-party internals were not audited beyond the `ollama` type introspection that D2.10 required.
- **Secondary outputs:** this phase declares none, so there is nothing to account for.
- **Evidence basis:** source inspection; **runtime verification** (test-suite execution, daemon-thread probe, dependency and client-type introspection); upstream findings from the architecture map; project documentation as the pass-6 baseline.
- **Known blind spots:** (1) **the GUI layer cannot be executed here** — no `tkinter` — so D1.4 stays an `open question` and every UI-behavior claim rests on reading; (2) the daemon-thread probe proves the *mechanism* of D2.3, but the precise leak *size* depends on how far a real run had rendered when the user quit; (3) memory figures in D2.5 and disk figures in D6.3 are order-of-magnitude estimates from typical PNG sizes, not measurements; (4) D6.2's exposure is measured as currently-unrealized, which is a snapshot of today's resolvable versions and not a guarantee about future ones; (5) gitignored files (`build-mac.sh`, `docs/`) are absent from the clone and could carry configuration this pass could not see.
- **Coverage disposition:** COMPLETE for the mechanical scope. All three assigned passes ran over the entire first-party source; gaps are deliberate deferrals (5 routed items) or the blind spots above.

## Validation

| # | Criterion | Result | Evidence |
|---|-----------|--------|----------|
| 1 | At least two of the three mechanical passes (1, 2, 6) produced findings or documented "no defects found." | PASS | All three produced findings: pass 1 → 6, pass 2 → 10, pass 6 → 7. |
| 2 | Each finding has location, severity, evidence level, and recommended action. | PASS | All 23 rows carry `file:line` location, severity, evidence level, and a pre-porting action, each with a prose subsection expanding the evidence. |
| 3 | Findings are organized by pass and sorted by severity. | PASS | One section per pass; rows descend high → medium → low within each (passes 1 and 6 have no critical or high). |
| 4 | Summary tables are complete and counts match the detailed findings. | PASS | §Findings by Severity totals 23 (0/3/8/12); §Findings by Pass rows total 6 + 10 + 7 = 23 with column sums 0/3/8/12. Both reconcile with D1.1–D1.6, D2.1–D2.10, D6.1–D6.7. |
| 5 | Items spotted that are actually semantic in nature are routed onward via a carry_forward entry in the phase handoff targeting defect-scan-semantic. | PASS | `mech-CF1`–`mech-CF5` documented in §Routed To Semantic Phase and recorded as `carry_forward` entries with `target_phase: defect-scan-semantic` in `scratch/handoffs/defect-scan-mechanical.yaml`. |
| 6 | Findings are marked with evidence levels. | PASS | 19 `observed fact`, 3 `strong inference`, 1 `open question` (D1.4, tracked as `q-ctk-image-clear`). Three findings are `observed fact` *because* they were executed this run (D1.6, D2.3, D6.2's tempering). |
| 7 | Coverage and limits name inspected scope, skipped scope, evidence basis, and blind spots. | PASS | §Coverage and limits names all four plus a COMPLETE disposition, records the two probes executed under C01, and notes the GUI layer as an inexecutable blind spot. Orchestrator duties (re-triage, contradiction sweep) are discharged in §Scan Context. |

**Validated by:** 2026-08-18 (defect-scan-mechanical phase, MCP-driven session 2, framework v0.16.0)
**Overall:** PASS
