# Behavioral Contracts — Local-OCR

Source: `../` (repository root), commit `8e7388c` · Framework v0.16.0 · Date 2026-08-18
Upstream: `findings/architecture/architecture-map.md`, `findings/defect-scan-mechanical/mechanical-defects.md`
**Closes `arch-CF4`.**

Evidence priority followed the skill's order: `README.md`, then `examples/`, then the ~1,540 lines of tests as executable contracts, then source for gaps. Tests were decisive — `tests/test_ocr_service.py` (1,069 lines) pins exact error strings, event ordering, and cleanup guarantees the README only gestures at.

### Verification status of the evidence — convention C02 applied

This is the single most important framing in this document. The test suite splits in two, and the halves have **different evidentiary weight**:

| Suite | Lines | Status this run | Weight of contracts it pins |
|---|---|---|---|
| `tests/test_ocr_service.py` | 1,069 | **Executed: 72 passed, 1 skipped, 0 failures** (`python3 -m unittest tests.test_ocr_service`) | `observed fact` — verified |
| 5 × `tests/test_*_gui.py` | ~470 | **Errors at collection** on a host without `tkinter` (D1.6); could not be executed here | **asserted-not-verified** |

Per **C02**, every contract below whose only evidence is a GUI test is marked `asserted-not-verified` rather than `observed fact`. That is not a claim the behavior is wrong — it is a claim that nothing has demonstrated it is right, and D1.6 makes it likely these suites have never run in any headless environment. Service-layer contracts carry full weight.

### Orchestrator duties discharged

**Open-question re-triage.** `q-test-invocation` (`needs-maintainer-decision`) — label re-tested, **confirmed**; a working command exists, only the project's preference is unstated. `q-ctk-image-clear` (`needs-runtime-test`) — label re-tested, **confirmed**; it needs a real customtkinter build, which C01 cannot reach on this host.

**Contradiction sweep.** One material item, and it changes this phase rather than being smoothed over: the mechanical scan measured that the GUI suites error rather than skip. A contracts phase that cited those tests as verified evidence would be contradicting a measured fact, so the verification table above was added and every affected contract downgraded. `q-gui-tests-ever-run` is opened to carry the residue.

## Surfaces Covered

| Surface | Present | Notes |
|---|---|---|
| Desktop GUI | **yes** | The only interactive surface. `python main.py`; all behavior reached through widgets. |
| Storage / export format | **yes** | `<stem>_extracted.md`, UTF-8, LF-normalized. The one durable artifact. |
| Outbound integration client | **yes** | Ollama HTTP (`list`, streaming `chat`). Not user-facing, but a hard external contract with test-pinned message shapes. |
| CLI | **no** | `main.py` reads no arguments (`observed fact`). |
| TUI / web UI | **no** | — |
| API or SDK | **no** | Nothing exported for programmatic use; no packaging, no `__all__` (`strong inference`). |
| Bot / background worker | **no** | — |

---

## Feature Contracts

### Surface: Desktop GUI

#### F1 — Select input file

| Field | Value |
|---|---|
| **Feature** | Choose the single PDF or image to convert. |
| **Trigger or input** | `Select File` → native open dialog. Ignored unless state is IDLE (`app.py:461-462`) (`observed fact`). |
| **Defaults** | Label reads `No file selected`; no path preselected. Filters: *Supported documents* (`*.pdf *.png *.jpg *.jpeg *.webp`), *PDF files*, *Images*, *All files* (`app.py:34-39`). |
| **Observable output** | Success → label shows the **basename only** (`path.name`, `app.py:478`). Rejection → `showwarning` titled `Unsupported file`. |
| **Side effects** | Sets `selected_path`. **A rejected file leaves the previous valid selection intact** (`app.py:471-476`, comment states this) (`observed fact`). Cancelling is a silent no-op. |
| **Persisted state** | None — selection does not survive exit. |
| **Error behavior** | `validate_input_path`: four ordered checks, distinct messages (F8). The *All files* filter makes reaching them easy. |
| **Retry or recovery** | Re-open the dialog. No retry logic. |
| **Owner** | `app.select_file` → `ocr_service.validate_input_path` |

#### F2 — Refresh model list

| Field | Value |
|---|---|
| **Feature** | Query the Ollama server for installed model tags. |
| **Trigger or input** | `Refresh Models`; reads the URL entry. IDLE-gated (`app.py:483`). |
| **Defaults** | URL `http://localhost:11434` (`config.py:3`). Timeout **10 s** (`config.py:22`) — "model listing should fail fast." |
| **Observable output** | Log `Refreshing model list from {url}...`, then `Found N model(s).` or `No models found on the server; enter a model tag manually.` Tags are **deduplicated, whitespace-stripped, empty/None discarded, case-insensitively sorted** (`ocr_service.py:90-97`; **verified** by `test_extraction_dedup_and_case_insensitive_sort` → `["Alpha:12b","beta:2b","zeta:7b"]`) (`observed fact`). |
| **Side effects** | Repopulates the combobox. **A tag the user already typed is preserved**; only an empty box is overwritten with `models[0]` (`app.py:583-585`) (`observed fact`). During refresh, `url_entry`/`refresh_button`/`start_button` are disabled — but `select_button`, model box, and DPI box are not (D1.3). |
| **Persisted state** | None. |
| **Error behavior** | Any exception → `OCRServiceError` carrying **the URL and the original message** (`ocr_service.py:94-95`); surfaced as `[Error] ...` **and** a `showerror` titled `Model refresh failed`. An unexpected response shape is wrapped identically — **verified** by `test_unexpected_response_shape_wrapped_with_context` (`observed fact`). |
| **Retry or recovery** | None automatic; state returns to IDLE. |
| **Owner** | `app.refresh_models` / `_refresh_worker` → `ocr_service.list_models` |

#### F3 — Run OCR

| Field | Value |
|---|---|
| **Feature** | Convert the selected document to Markdown, page by page. |
| **Trigger or input** | `Start OCR`. IDLE-gated. All four inputs validated and snapshotted into a frozen `OCRRequest` **on the main thread** before the worker starts (`app.py:598-656`) (`observed fact`). |
| **Defaults** | DPI **150** (`config.py:10`); 100/150/200/300, widget `readonly`. Model box starts **empty** — the two `EXAMPLE_MODELS` are dropdown entries only (`app.py:122`). DPI affects **PDFs only**. |
| **Observable output** | Two `[Start]` log lines, then staged `[1/3]`/`[2/3]`/`[3/3]` lines, a determinate bar, a `Page N / M (Render\|OCR)` label, a live thumbnail, Result text streaming token-by-token, Review pairs accumulating. Success → **Result** tab + completion dialog; failure → **Log** tab + `showerror`. |
| **Side effects** | Writes `<stem>_extracted.md` beside the input. Creates and removes a `local_ocr_*` temp dir (PDF only). Disables all controls. Clears Result/preview/Review and forces the **Log** tab (`app.py:436-439`) (`asserted-not-verified` — GUI test only). |
| **Persisted state** | The output `.md` only. |
| **Error behavior** | Any stage failure aborts the run. **No partial output is written, and an existing output from a previous run is left untouched** — **verified** by `test_late_page_failure_preserves_existing_output` (`observed fact`). |
| **Retry or recovery** | **None.** No retry, no resume, no cancellation. Completed work is discarded (D2.1). |
| **Owner** | `app.start_ocr` / `_ocr_worker` → `ocr_service.process_ocr` |

#### F4 — Overwrite confirmation

| Field | Value |
|---|---|
| **Feature** | Guard against silently replacing an existing extraction. |
| **Trigger or input** | Automatic during `start_ocr` when `build_output_path(input).exists()` (`app.py:632-641`). |
| **Defaults** | `askyesno`, titled `Overwrite existing file?`, naming the file. |
| **Observable output** | Modal yes/no. |
| **Side effects** | **No** on decline: the run never starts, state stays IDLE, nothing is logged, the file is untouched (`observed fact`). |
| **Persisted state** | None. |
| **Error behavior** | A plain `exists()` check well before the write (`mech-CF1`, TOCTOU). |
| **Retry or recovery** | User re-triggers `Start OCR`. |
| **Owner** | `app.start_ocr` → `ocr_service.build_output_path` |

#### F5 — Live result streaming and Copy

| Field | Value |
|---|---|
| **Feature** | Show recognized Markdown as the model generates it. |
| **Trigger or input** | `stream_chunk` events; `Copy` button. |
| **Defaults** | Flush throttled to **100 ms** (`config.py:37`). Textbox read-only, toggled around each insert. |
| **Observable output** | Text appended in flush batches, autoscrolled. Pages separated by exactly `"\n\n"`, inserted at the page boundary **inside the buffer** so a streamed run reads byte-identically to the saved file (`app.py:557-558`) (`asserted-not-verified` — pinned by `test_multi_page_streaming_inserts_single_separator`, a GUI test; the *saved-file* side of the equality is independently `observed fact` from the service suite). |
| **Side effects** | `Copy` calls `clipboard_clear` then `clipboard_append` of the whole panel (`app.py:571-574`). |
| **Persisted state** | None (and on X11 the clipboard dies with the process — D6.6). |
| **Error behavior** | A page already written live is **not** re-appended when its `page_text` arrives (`asserted-not-verified` — `test_page_text_after_streaming_does_not_duplicate`). |
| **Retry or recovery** | `_flush_stream_buffer` is forced on both terminal events so nothing is stranded (`app.py:673`, `735`). |
| **Owner** | `app.on_stream_chunk` / `on_page_text` / `_flush_stream_buffer` / `copy_result` |

#### F6 — Progress reporting

| Field | Value |
|---|---|
| **Feature** | Determinate progress across render and recognition. |
| **Trigger or input** | `progress` events, emitted **before** each unit of work (`ocr_service.py:123-124`, `179-180`) (`observed fact` — service suite verifies the emission order). |
| **Defaults** | Bar starts `indeterminate` and animating; the first `progress` event switches it to `determinate` (`app.py:519-521`). |
| **Observable output** | Render `0.2 × c/t`; OCR after render `0.2 + 0.8 × c/t`; OCR with no render `c/t`. Label `Page {c} / {t} ({Render\|OCR})`. All three formulas are asserted by `test_render_phase_fraction`, `test_ocr_phase_after_render`, `test_ocr_phase_without_render` — **all GUI tests, so `asserted-not-verified`**. Note `test_ocr_phase_without_render` asserts `1.0` for a single image, locking in D1.1's behavior. |
| **Side effects** | `_render_phase_seen` latches on the first render event and resets in both `_apply_ocr_busy_state` and `_restore_idle`. |
| **Persisted state** | None. |
| **Error behavior** | `_restore_idle` returns the bar to `indeterminate`, value 0, label empty. |
| **Owner** | `app.on_progress` |

#### F7 — Preview and Review tabs

| Field | Value |
|---|---|
| **Feature** | Thumbnail of the page being read; side-by-side spot-checking of finished pages. |
| **Trigger or input** | `page_image` (before the send) and `page_text` (after assembly); `◀`/`▶`. |
| **Defaults** | Thumbnail longest side **900 px** (`config.py:39`), never upscaled (`observed fact` — `test_small_image_not_upscaled`, service suite). Preview column 240 px; Review image 400 px wide, capped 560 px tall; decoded-image LRU holds **5** (`app.py:46`). |
| **Observable output** | Preview caption `Page {n} / {total}`; Review label `Page {page} / {document_total}`, falling back to the ready count (`app.py:404`). Nav buttons enable only at valid edges; empty state `No pages yet`. (`asserted-not-verified` — GUI tests.) |
| **Side effects** | Pages become navigable **only once their text is ready** (`_register_review_page` is called from `on_page_text` alone, `app.py:541`). The first ready page displays automatically; thereafter **the user's position is preserved** (`app.py:356-359`) (`asserted-not-verified` — `test_navigation_across_pages`). |
| **Persisted state** | None; `_apply_ocr_busy_state` clears everything. |
| **Error behavior** | **A thumbnail failure is non-fatal**: logged as `Could not build preview for page N/M: ...`, no `page_image` emitted, OCR continues — **verified** by `test_thumbnail_failure_does_not_abort_ocr` (service suite) (`observed fact`). Such a page is still navigable with a blank image (`asserted-not-verified` — GUI test). |
| **Owner** | `app.on_page_image`, `_register_review_page`, `show_review_page`, `_review_image_for` → `ocr_service.make_thumbnail_png` |

#### F8 — Input validation (exact user-visible messages)

Ordered checks in `validate_input_path` (`ocr_service.py:65-77`); the **first** failure wins. All `ValueError`, all surface verbatim in a dialog. **All four verified** by the service suite (`observed fact`):

| Order | Condition | Message |
|---|---|---|
| 1 | path missing | `File does not exist: {path}` |
| 2 | not a regular file (e.g. a directory named `folder.pdf`) | `Not a regular file: {path}` |
| 3 | suffix unsupported | `Unsupported file type '{suffix}'. Supported: .jpeg, .jpg, .pdf, .png, .webp` (sorted) |
| 4 | not `os.R_OK` | `File is not readable: {path}` |

Extension matching is **case-insensitive** — `UPPER.PDF`, `SHOUT.PNG`, `MIXED.JpEg` accepted. *(Check 4's test self-skips as root, so it is verified-by-inspection only — the one service-suite skip.)*

URL normalization (`normalize_ollama_url`, `ocr_service.py:45-62`), all rows **verified**:

| Input | Result |
|---|---|
| `  http://192.168.1.20:11434/  ` | `http://192.168.1.20:11434` (trimmed both ends) |
| `http://ollama.local:11434//` | `http://ollama.local:11434` (**all** trailing slashes) |
| `https://server.lan/ollama/` | `https://server.lan/ollama` (**path prefix preserved** for reverse proxies) |
| `""`, `"   "`, `"///"` | `ValueError` — `Ollama server URL is empty.` |
| `http://`, `http:///path` | `ValueError` — `...has no host: ...` |
| `ftp://host`, `file:///tmp/x`, `localhost:11434` | `ValueError` — `...must start with http:// or https:// (got: ...)` |

`/api` is **never** appended — the client owns API paths (`ocr_service.py:48-50`).

#### F9 — Completion dialog and file actions

| Field | Value |
|---|---|
| **Feature** | Post-run actions on the saved file. |
| **Trigger or input** | Emitted automatically on `ocr_success`. |
| **Defaults** | A **`CTkToplevel`, not a messagebox** — a native box cannot carry custom buttons (`app.py:687-691`) (`asserted-not-verified` — GUI tests). **Non-blocking** (`transient`, no `grab_set`); OK has focus. |
| **Observable output** | `Markdown saved to:\n{path}`, plus `Open`, a platform-named reveal button (`Show in Finder` / `Show in Explorer` / `Open Folder`, `app.py:679-685`), and `OK`. |
| **Side effects** | `Open` → `open` / `os.startfile` / `xdg-open`; reveal → `open -R` / `explorer /select,` / `xdg-open` on the **parent directory**. All argv vectors, never `shell=True`. **All six branches verified** by the service suite (`observed fact`). |
| **Persisted state** | None. |
| **Error behavior** | `_run_file_action` catches **only** `OCRServiceError`, logs `[Error] ...`, shows `Action failed` (`asserted-not-verified` — GUI test). The Windows reveal branch cannot detect failure (D2.8). |
| **Owner** | `app._show_completion_dialog` / `_run_file_action` → `ocr_service.open_in_default_app` / `reveal_in_file_manager` |

#### F10 — Shutdown

| Field | Value |
|---|---|
| **Feature** | Close the window. |
| **Trigger or input** | `WM_DELETE_WINDOW`. |
| **Defaults** | Idle → immediate `destroy()`. Busy → `askyesno` "Quit" / "An operation is still running. Close anyway?" |
| **Side effects** | Sets `closing`, making `drain_ui_events` return without rescheduling and discarding queued UI work (`app.py:248-249`). The daemon worker is **neither joined nor signalled** (`observed fact`). |
| **Persisted state** | Nothing saved. A mid-run quit **provably leaks** the `local_ocr_*` directory — D2.3, confirmed by probe this run (`observed fact`). |
| **Error behavior** | Declining the confirmation cancels the close; the run continues. |
| **Retry or recovery** | None. |
| **Owner** | `app.on_close` |

### Surface: Storage / export format

#### F11 — The `_extracted.md` artifact

| Field | Value |
|---|---|
| **Feature** | The single durable output. |
| **Trigger or input** | Written once at stage `[3/3]` after **all** pages succeed. |
| **Defaults** | `input.with_name(f"{stem}_extracted.md")` — same directory always. Dotted stems preserved: `/docs/report.v2.pdf` → `/docs/report.v2_extracted.md` (**verified**, `test_dotted_stem`) (`observed fact`). |
| **Observable output** | UTF-8, **LF only** (`\r\n` and bare `\r` both normalized, `ocr_service.py:241`; **verified** → `b"a\nb\nc\n"`). Pages joined with exactly `"\n\n"` (**verified** → `"p1\n\np2\n\np3"`). Each page `.strip()`ed. **No header, footer, page markers, or front-matter** — raw concatenated model output (`observed fact`). |
| **Side effects** | Written via `NamedTemporaryFile(delete=False, dir=output.parent, prefix=".{stem}_", suffix=".tmp")` then `os.replace`. |
| **Persisted state** | This file. |
| **Error behavior** | On `os.replace` failure: original preserved, temp removed, `Could not save output file: {cause}` (**verified**). On temp-creation failure: no files at all (**verified**). On success: no leftovers (**verified**). |
| **Retry or recovery** | None; a save failure fails the run. |
| **Owner** | `ocr_service.save_markdown_atomic` / `build_output_path` |

### Surface: Outbound integration (Ollama)

#### F12 — Per-page recognition request

| Field | Value |
|---|---|
| **Feature** | One independent streaming chat request per page. |
| **Trigger or input** | `client.chat(model=..., messages=[system, user], stream=True)` — always `stream=True`. |
| **Defaults** | Client `timeout = 120 s` applied to **gaps between chunks**, not the whole response (`config.py:24-28`). Messages exactly: system = `config.SYSTEM_PROMPT`; user = `{"role":"user","content":"Recognize this document page.","images":[str(path)]}`. **Verified field-for-field**, including that the message lists are distinct objects per call (`observed fact`). |
| **Observable output** | Deltas read as `getattr(getattr(chunk,"message",None),"content",None) or ""`; empty deltas skipped; page result is `"".join(chunks).strip()`. **The installed client still exposes exactly these fields** (`observed fact`, introspection this run). |
| **Side effects** | Emits `stream_chunk` per non-empty delta, then one `page_text`. Requests carry **no page-to-page context** — every page is stateless, so cross-page continuity is impossible by construction (`observed fact`). |
| **Persisted state** | None server-side. |
| **Error behavior** | Any exception → `Ollama request failed on page {n}/{N} (model '{model}'): {cause}`, covering **both** the call and iteration of the stream (**verified**, `test_generator_exception_mid_stream_wrapped_with_context`). Empty assembled text → `Ollama returned no text for page {n}/{N} (model '{model}').` Note this wrapper also captures **local** path-resolution failures, misattributing them to the server (D2.10). |
| **Retry or recovery** | **None.** First failure aborts the document. |
| **Owner** | `ocr_service.recognize_images` |

#### F13 — PDF rasterization

| Field | Value |
|---|---|
| **Feature** | Render every page to PNG before recognition begins. |
| **Trigger or input** | PDF input only; `pymupdf.open(path)`. |
| **Defaults** | `get_pixmap(dpi=dpi, colorspace=pymupdf.csRGB, alpha=False)`; filenames `page_%04d.png`, zero-padded so lexical order matches numeric order past page 9 (**verified**) (`observed fact`). |
| **Observable output** | `Rendering page {n}/{N}...` per page, prefixed `[1/3]` by the caller. |
| **Side effects** | Fills the temp directory. The document is closed via `with` on **every** path, including both early rejections — asserted in four separate service tests, all **verified**. |
| **Error behavior** | Password-protected → `PDF is password-protected; encrypted documents are not supported.` **before any page renders** (`load_page` asserted not called). Zero pages → `PDF contains no pages.` Open failure → `Could not open PDF: {cause}`. Per-page failure → `Failed to render page {n}/{N}: {cause}`. All **verified**. |
| **Retry or recovery** | None. |
| **Owner** | `ocr_service.render_pdf` |

---

## High-Value Behaviors

**Cancellation and abort — absent.** No cancel control, no `threading.Event`, no cancellation check in either loop. The only abort is quitting, which is confirmed but **provably unclean** (F10, D2.3) (`observed fact`).

**Streaming and partial output — present in the UI, absent in the artifact.** Text streams live into the Result panel, but the file is written once, at the end, only if *every* page succeeded. The two are deliberately kept byte-identical: the separator is embedded into the stream buffer at the page boundary specifically so the live view matches the saved file (`app.py:553-555`).

**Partial success — explicitly rejected, by design, and verified.** The system's most consequential contract: `test_late_page_failure_leaves_no_new_output` and `test_late_page_failure_preserves_existing_output` **assert** that a failure on page N discards pages 1..N−1 and preserves any prior output — and both are in the **service suite, so both are verified-passing** (`observed fact`). The mechanical scan rated this `high` (D2.1); the tests show it is an intentional, locked-in all-or-nothing transaction. A port that adds partial saves is **changing a passing, tested contract**, not fixing a bug — that framing belongs in `porting`.

**Queueing and follow-up — none.** One file per run, one operation at a time via `OperationState`. No batch mode, no job list.

**Persistence and resume — none.** No settings, no history, no resume. Every launch is a blank slate; a re-run repeats all inference from page 1.

**Ordering guarantees** (all **verified** by the service suite, `observed fact`). Per page: `page_image` (if the thumbnail succeeded) → `Sending page n/N` log → `stream_chunk`* → `page_text`; `test_page_image_event_before_send_with_valid_png` asserts the image precedes the send log. Across phases: all render events precede all OCR events, asserted by index comparison. Exactly one terminal event per run, enqueued after cleanup.

**Compaction / summarization, tool execution — not applicable.**

---

## Security and Authorization

The app has **no authentication, no authorization, and no session model** — nothing to sign in to, no multi-user concept (`observed fact`). What exists is a trust posture:

- **Credentials:** none anywhere. No API keys, tokens, or secrets in source, config, or on the wire. Ollama itself has no built-in authentication, which the README states (`README.md:48-53`).
- **Trust boundaries.** *Untrusted:* the input document (arbitrary user-supplied PDF/image, parsed by PyMuPDF and Pillow — native parsers) and the model's output text (written verbatim, unsanitized). *Trusted, implicitly:* the Ollama server at whatever URL the user types. Page images — the full content of the user's document — go there with no transport guarantee.
- **Transport:** `normalize_ollama_url` accepts any `http://` or `https://` host; the default is plaintext `http://localhost:11434`. Nothing enforces TLS, warns on a non-loopback plaintext URL, or pins a host. The README's "Nothing leaves your machine or network" holds only as far as the user's URL choice does (`observed fact`).
- **Command execution:** the three OS-integration paths use argv vectors with no `shell=True` (`ocr_service.py:274-297`), so a hostile filename cannot inject a command. The Windows `f"/select,{path}"` interpolation is a single argv element. **All six branches verified** by the service suite.
- **Output path:** derived mechanically from the input path, never user-typed. No traversal surface.
- **CORS / CSP / web policy:** not applicable.

Depth analysis — TOCTOU windows and plaintext-transport exposure — is `defect-scan-semantic`'s rubric, routed as `mech-CF1` and `mech-CF2`.

---

## Configuration Model

Deliberately minimal — the smallest model the template's categories can describe (`observed fact`):

| Question | Answer |
|---|---|
| Config sources and precedence | **One source: Python module constants in `config.py`.** No precedence chain, because there is nothing to precede. |
| Config file format and location | **None.** No file read at startup. |
| Environment variables | **None.** Zero `os.environ`/`os.getenv` reads in first-party code. |
| Feature flags | **None.** |
| Environment-specific overrides | **None.** No dev/staging/prod concept. |
| Config validation | Not applicable to literals. **Runtime input** is validated: URL via `normalize_ollama_url`, file via `validate_input_path`, DPI via membership in `DPI_OPTIONS` with an `int()` guard falling back to `-1` so the membership check rejects it (`app.py:621-630`). |
| User-adjustable at runtime | URL, model tag, DPI — **all three discarded on exit** (D6.1). |

Full constant set: `DEFAULT_OLLAMA_URL`, `EXAMPLE_MODELS`, `DPI_OPTIONS`, `DEFAULT_DPI`, `PDF_EXTENSIONS`, `IMAGE_EXTENSIONS`, `SUPPORTED_EXTENSIONS`, `MODEL_LIST_TIMEOUT` (10 s), `OCR_STREAM_IDLE_TIMEOUT` (120 s), `STREAM_UI_FLUSH_MS` (100), `THUMBNAIL_MAX_SIDE` (900), `UI_POLL_INTERVAL_MS` (50), `SYSTEM_PROMPT`, `USER_PROMPT`.

**`SYSTEM_PROMPT` is configuration in the load-bearing sense** — the single largest determinant of output quality, asserted verbatim by a **verified** service test, and the least reachable setting in the app. Its text instructs: convert to Markdown, high-accuracy OCR, preserve headers/lists/tables, **no greetings or explanations**, output only raw recognized text. A port must treat it as a versioned behavioral asset (`strong inference`).

The model is simple enough that the secondary output `findings/config-model/config-model.md` was not warranted — see Coverage and limits.

---

## Doc/Test Conflicts

| # | Conflict | Sources | Resolution |
|---|---|---|---|
| 1 | **Atomicity overstated.** README: "written atomically, so a failed run never leaves a partial result." Tests verify only process-level failure; there is no `fsync`, so an OS crash can truncate. | `README.md:82-84` vs `ocr_service.py:239-263`, D2.4 | Both right about different failure classes; the wording implies durability it lacks. Routed as `mech-CF4`. |
| 2 | **"Returned no text" has ≥3 causes, one documented.** README blames a non-vision model. Verified tests show *any* empty result triggers it — including a blank page (D2.2) and, per D2.10/`mech-CF5`, a client-side shape mismatch. | README troubleshooting vs `ocr_service.py:224-229` | Docs incomplete, not wrong. Routed under `mech-CF4`. |
| 3 | **A test locks in behavior the scan flagged.** `test_ocr_phase_without_render` asserts `progress.get() == 1.0` for a single image — D1.1's exact complaint. | `tests/test_progress_bar_gui.py` vs D1.1 | Corrects framing from bug to deliberate choice. **But note:** this is a GUI test, so it is `asserted-not-verified` — the behavior is *specified*, not *demonstrated*. Carried to `porting` as `contracts-CF2`. |
| 4 | **The "dead" fallback branch is covered.** D1.2 called `on_page_text`'s non-streaming append unreachable; `test_page_text_without_stream_is_appended` exercises it. | `tests/test_result_panel_gui.py` vs D1.2 | Consistent: dead in production, live under test. Also GUI-only, so `asserted-not-verified`. No action beyond D1.2's `leave behind`. |
| 5 | **NEW — the suites' own skip docstring is false.** All five GUI files state they "are skipped automatically when no display is available (CI, headless containers)." Measured: they **error at collection** without `tkinter`. | `tests/test_*_gui.py:1-10` vs measured run, D1.6 | The docstring is a false claim about the tests themselves, which is why every GUI-backed contract here is `asserted-not-verified`. Residue tracked as `q-gui-tests-ever-run`. |

No conflict was found between the README's described workflow and observed behavior on the main success path; the `examples/` fixtures match the documented supported types.

---

## Black-Box Acceptance List

Runnable against any reimplementation without reading the source. Preconditions assume a reachable Ollama server with a working vision model unless stated. The **V** column records whether an equivalent assertion is currently *verified* in this repo (`Y` = service suite, passing; `A` = GUI-suite only, asserted-not-verified; `—` = no existing test).

| # | Scenario | Precondition | Action | Expected Outcome | V |
|---|---|---|---|---|---|
| 1 | Single image end-to-end | `scan.png` in an empty dir | Select, set model, Start OCR | `scan_extracted.md` appears, UTF-8, LF-only, no added header/footer | Y |
| 2 | Multi-page PDF joins with a blank line | 3-page PDF | Run to completion | `page1 + "\n\n" + page2 + "\n\n" + page3`, each stripped, in document order | Y |
| 3 | Dotted stems preserved | `report.v2.pdf` | Run | Output exactly `report.v2_extracted.md` | Y |
| 4 | Uppercase extensions accepted | `SCAN.PDF` | Select | Accepted, no warning | Y |
| 5 | Unsupported type rejected, prior selection kept | `a.pdf` selected; `notes.txt` exists | Select `notes.txt` via *All files* | Warning listing `.jpeg, .jpg, .pdf, .png, .webp`; label still `a.pdf` | Y |
| 6 | Missing file rejected | Path deleted after listing | Select it | Error containing "exist" | Y |
| 7 | Directory rejected | Directory named `folder.pdf` | Select it | Error: not a regular file | Y |
| 8 | URL trimming | — | Enter `  http://host:11434//  `, Refresh | Contacted at `http://host:11434` | Y |
| 9 | Reverse-proxy path preserved | Proxy at `https://server.lan/ollama` | Enter with trailing slash, Refresh | Contacted at that path; `/api` **not** appended | Y |
| 10 | Bad scheme rejected | — | Enter `ftp://host`, Refresh | Error requiring http/https; no network call | Y |
| 11 | Empty URL rejected | — | Clear field, Refresh | Error "URL is empty"; no network call | Y |
| 12 | Model list deduped, sorted case-insensitively | Server returns `zeta:7b, Alpha:12b, "  ", zeta:7b, null, "beta:2b "` | Refresh | Exactly `Alpha:12b, beta:2b, zeta:7b`; log `Found 3 model(s).` | Y |
| 13 | Empty model list is not an error | Server has no models | Refresh | Log the manual-entry hint; no error dialog; controls return to idle | Y |
| 14 | Typed model survives a refresh | Type `custom:tag`, Refresh with 3 models | — | Selection remains `custom:tag` | Y |
| 15 | Unreachable server | Ollama stopped | Refresh | Error dialog **and** log line, both naming URL and cause; app stays usable | Y |
| 16 | No model selected | File chosen, model box empty | Start OCR | Error "Enter or select an Ollama model tag."; no request sent | — |
| 17 | No file selected | Fresh launch | Start OCR | Error "Select a PDF or image file first." | — |
| 18 | Overwrite declined leaves file untouched | `doc_extracted.md` contains `OLD` | Start OCR, answer No | Still `OLD`; no run starts; controls stay enabled | — |
| 19 | Overwrite accepted replaces content | Same | Answer Yes, finish | File contains only the new extraction | Y |
| 20 | **Mid-document failure preserves prior output, writes nothing new** | `doc_extracted.md` contains `OLD`; 2-page PDF; server fails on page 2 | Run | Still `OLD`; no partial file, no `.tmp` leftovers; error names page 2 and the model | Y |
| 21 | Failure error identifies page and model | 3-page PDF; error on page 2 | Run | Message contains `page 2/3` and the model tag | Y |
| 22 | Blank model response is fatal and identified | Server returns empty text for page 1 of 1 | Run | Error containing `returned no text` and `page 1/1`; no output file | Y |
| 23 | Password-protected PDF rejected before any work | Encrypted PDF | Run | Error mentioning "password"; **no pages rendered**, no temp dir left | Y |
| 24 | Zero-page PDF rejected | 0-page PDF | Run | Error `PDF contains no pages.` | Y |
| 25 | Temp directory always removed **on completed runs** | 3-page PDF | Run to success; then induce a page-2 failure | No `local_ocr_*` remains under system temp in either case | Y |
| 26 | Image input skips rendering | `photo.png` | Run | Exactly one progress event (`ocr 1/1`); no temp dir created; request carries the original path | Y |
| 27 | Progress phases and labels | 3-page PDF | Run | Bar switches from animating to determinate on the first event; labels `Page n / 3 (Render)` then `(OCR)`; render fractions ≤ 0.2 | A |
| 28 | Live text matches the saved file | 2-page PDF | Compare Result tab to the file | Byte-identical, single blank line between pages, no duplicated page text | A |
| 29 | Copy puts the full result on the clipboard | Completed run | Copy, paste elsewhere | Pasted text equals the Result panel | A |
| 30 | Review pairs pages and preserves position | 3-page PDF | Stay on page 1 while 2 and 3 arrive | Still page 1; `▶` enables; `◀` stays disabled; stepping shows each page's own image and text | A |
| 31 | Review nav does not overrun | 1-page document | Press `◀` then `▶` | Stays `Page 1 / 1`; no error | A |
| 32 | Preview failure is non-fatal | Thumbnail made to fail for page 2 of 3 | Run | Log reports the preview failure; all 3 pages recognized; page 2 navigable with a blank image | Y |
| 33 | New run clears prior state | Completed run visible | Start a second run | Result, preview, Review empty; Log tab selected; bar restarts | A |
| 34 | Terminal tab switching | — | Complete a run, then force a failure | Success → **Result**; failure → **Log** | A |
| 35 | Success dialog offers three actions | Completed run | Observe | Non-blocking dialog with saved path, `Open`, platform reveal button, `OK` | A |
| 36 | Reveal failure is reported, not silent | Reveal helper made to fail | Press reveal | Error dialog appears and `[Error]` logged | A |
| 37 | Quit is guarded while busy | Run in progress | Close window, answer No | Stays open, run continues; Yes closes it | — |
| 38 | Only one operation at a time | Refresh or OCR in progress | Press the other | No second operation begins | — |
| 39 | Settings snapshot at start | — | Start with DPI 300, change the widget mid-run if reachable | Running job continues at 300 | Y |
| 40 | Defaults on a fresh launch | Fresh start | Observe | URL `http://localhost:11434`, model box **empty**, DPI `150`, `No file selected`, Log tab active, `No pages yet` | A |
| 41 | **Quitting mid-run must not leak the render directory** | 20-page PDF at 300 DPI; quit at page 5 | Confirm the quit prompt | **No `local_ocr_*` directory remains** under system temp. *The current implementation fails this* (D2.3, proven by probe) | — |
| 42 | **Headless test run is green** | Host without a display, or without the GUI toolkit | Run the project's full test command | The suite reports pass/skip, **never collection errors**. *The current implementation fails this* (D1.6, measured) | — |

Scenarios 20, 22, 23, 25, 28, and 32 are the highest-value parity checks. **41 and 42 are new this run and are the only two the current implementation demonstrably fails** — both added because execution, not reading, revealed them.

---

## Coverage and limits

- **Inspected scope:** `README.md` in full; all six test files read line-by-line (~1,540 lines) and used as the primary contract source; `app.py` and `ocr_service.py` re-read for every contract field; `config.py` in full; `examples/` inventoried. Every user-visible string traced to its emitting line. **Both suites were executed** (per C01): the service suite passes 72/1-skipped, the GUI suites error at collection. `arch-CF4` addressed and closed.
- **Skipped scope:** wire payload schemas, the formal event catalog, and the `OperationState` transition table are `protocols`' rubric (routed as `arch-CF3`) and deliberately not formalized here. Concurrency, transport security, and TOCTOU depth are `defect-scan-semantic`'s (`mech-CF1`–`mech-CF5`). The `examples/` PDF and screenshot were not run through the pipeline (no model available). Third-party rendering fidelity not evaluated.
- **Secondary outputs — explicit accounting** (four declared by this phase):
  - `findings/public-surfaces/public-surfaces.md` — **appended** with the contract-level surface split and the verification-status framing.
  - `findings/runtime-lifecycle/runtime-lifecycle.md` — **appended** with the operation lifecycle from the contracts' point of view.
  - `findings/state-and-storage/state-and-storage.md` — **appended** with the `_extracted.md` contract (F11).
  - `findings/config-model/config-model.md` — **deliberately not written.** §Configuration Model above is complete: one flat set of module constants, no file, no env vars, no precedence chain, nothing to trace across a boundary. Recorded as decision `contracts-D2`.
- **Evidence basis:** tests (primary), **runtime verification** (both suites executed; `ollama` client types introspected), source inspection, project documentation, and upstream findings.
- **Known blind spots:** (1) **no live model was contacted**, so output quality and how well `SYSTEM_PROMPT` preserves tables and lists are unmeasured; (2) **the ~470 lines of GUI contracts could not be executed** — this host has no `tkinter` and D1.6 shows the suites error rather than skip, so 14 of the 42 acceptance rows are `A` (asserted-not-verified) and the UI half of this spec is weaker evidence than the service half; (3) dialog rendering and platform reveal behavior are assumed from the API; (4) whether the GUI suites pass on a host *with* `tkinter` is unknown (`q-gui-tests-ever-run`); (5) gitignored `docs/` may document further contracts.
- **Coverage disposition:** COMPLETE for the contracts scope — all three surfaces split and specified, 13 contracts with all nine fields, 42 executable acceptance scenarios — but note the **evidence is deliberately two-tier**, and the tiering is recorded per contract rather than averaged away.

## Open Questions

| ID | Kind | Description | Deferred Reason |
|---|---|---|---|
| `q-gui-tests-ever-run` | needs-maintainer-decision | **Reframed on measurement.** It is now established that the five GUI suites *error at collection* on a host without `tkinter` (D1.6), so they cannot have passed in such an environment. What remains unknown is whether they pass on a developer machine **with** `tkinter` and a display — and therefore whether the ~470 lines of UI contracts have ever been demonstrated anywhere. | Requires either a maintainer statement about the verification environment, or a host with `tkinter` and a display. Neither is reachable here, and no phase's rubric provides one. |
| `q-prompt-provenance` | needs-maintainer-decision | `SYSTEM_PROMPT` drives output quality more than any other setting and is asserted verbatim by a passing test, but nothing records how it was derived, what alternatives were rejected, or which models it was tuned against. A port cannot tell which clauses are load-bearing. | Project history, not source. Unresolvable by any phase's rubric. |

## Carry-Forward

| ID | Target Phase | Description | Deferred Reason |
|---|---|---|---|
| `contracts-CF1` | porting | The all-or-nothing transaction is a **verified, passing** contract (`test_late_page_failure_leaves_no_new_output`, `test_late_page_failure_preserves_existing_output`, both service-suite), not an oversight. Any partial-save or resume design deliberately breaks two passing tests, and that tradeoff needs an explicit decision with the defect synthesis in view. | Reconciling a passing contract against a `high` defect finding is a synthesis judgment — what `porting`'s Defect Synthesis exists for. Deciding it here would pre-empt that section without the semantic scan's input. |
| `contracts-CF2` | porting | The progress formulas are asserted by `test_ocr_phase_without_render`, so D1.1 is a locked-in design choice — **but that test is GUI-only and unverified**, so the contract is specified rather than demonstrated. A port must decide whether to preserve the tested formula or fix the UX, knowing the test may never have run. | Same class as `contracts-CF1`: a defect-versus-contract tradeoff belonging to the porting synthesis, now with the extra wrinkle that the contract's evidentiary weight is low. |
| `contracts-CF3` | protocols | Contract fields reference event kinds and payload keys informally (`page_image` precedes the send log, exactly one terminal event, per-page ordering). The formal catalog, payload schemas, and the `OperationState` transition table remain unspecified. | Wire shapes and state machines are `protocols`' rubric; this reinforces `arch-CF3` from the contracts side rather than duplicating it. |
| `contracts-CF4` | porting | Acceptance rows 41 and 42 are the only two the current implementation demonstrably **fails** (the mid-run temp-dir leak, proven by probe; and the headless test run erroring rather than skipping). Both must survive into the spec as normative rules, not as observations. | The spec phase converts defects into normative rules plus scenarios; routing through `porting` keeps them attached to their dispositions (D2.3, D1.6) rather than floating free. |

---

## Validation

| # | Criterion | Result | Evidence |
|---|-----------|--------|----------|
| 1 | User-facing surfaces are split by surface type. | PASS | §Surfaces Covered lists all seven template surface types with present/absent; §Feature Contracts is split into three `### Surface:` sections. |
| 2 | Feature contracts record trigger, defaults, outputs, side effects, persisted state, error behavior, and recovery behavior. | PASS | 13 contracts (F1–F13), each with all seven fields plus **Owner**. F8 additionally tabulates every validation message verbatim. |
| 3 | Security and authorization model is documented (if applicable). | PASS | §Security and Authorization documents the absence of auth and specifies what exists: credentials (none), trust boundaries, transport posture, command-execution safety (six branches verified), output-path derivation. Depth routed to the semantic scan. |
| 4 | Contract ownership is mapped back to a layer or package. | PASS | Every contract's **Owner** row names module and function, consistent with the architecture map's layer roles. |
| 5 | A black-box acceptance list is included. | PASS | §Black-Box Acceptance List — 42 scenarios with precondition, action, expected outcome, **and a verification-status column**; runnable without reference to the source. |
| 6 | Findings are marked with evidence levels. | PASS | `observed fact`, `strong inference`, and — new this run under C02 — **`asserted-not-verified`** used throughout, with each `observed fact` traced to a named *passing* test or a `file:line` citation. The two-tier scheme is declared up front in §Verification status of the evidence. |
| 7 | Coverage and limits name inspected scope, skipped scope, evidence basis, and blind spots. | PASS | §Coverage and limits names all four plus a COMPLETE disposition, **explicitly accounts for all four declared secondary outputs** (three appended, one deliberately not), and records the unexecutable GUI half as the primary blind spot. Orchestrator duties discharged in §Orchestrator duties discharged. |

**Validated by:** 2026-08-18 (contracts phase, MCP-driven session 2, framework v0.16.0)
**Overall:** PASS
