
---

## From: architecture phase (2026-08-18, v0.16.0)

Local-OCR exposes **one interactive surface only**. Recorded here so later phases can append without re-deriving the inventory.

**Absent by verification** (`observed fact`): no CLI arguments (`main.py` reads no `sys.argv`, uses no `argparse`), no network listener, no exported library API (no packaging, no `__all__`), no IPC, no bot or daemon surface.

**Present:**

| Surface | Detail | Owner |
|---|---|---|
| Desktop GUI | 11 controls: file select, URL entry, Refresh Models, model combobox, DPI combobox, Start OCR, Log/Result/Review tabs, Copy, `◀`/`▶`, completion dialog, window close | `app.LocalOCRApp` |
| Outbound HTTP (Ollama) | `client.list()` timeout 10 s; `client.chat(..., stream=True)` once per page, timeout 120 s applied to **inter-chunk gaps** not total response | `ocr_service.list_models`, `recognize_images` |
| Input file formats | `.pdf`, `.png`, `.jpg`, `.jpeg`, `.webp` — matched case-insensitively on suffix | `ocr_service.validate_input_path` |
| Output artifact | `<stem>_extracted.md` beside the input; UTF-8, LF-only, pages joined `"\n\n"`, no header/footer/page markers | `ocr_service.save_markdown_atomic` |
| OS integration | macOS `open` / `open -R`; Windows `os.startfile` / `explorer /select,`; else `xdg-open` (file, then parent dir). All argv vectors, never `shell=True` | `ocr_service.open_in_default_app`, `reveal_in_file_manager` |

**Verified this run** (`observed fact`): the installed `ollama` client still exposes the exact shapes the code reads defensively — `ChatResponse.message` → `Message` (with `content`), and `ListResponse.models` → `Sequence[ListResponse.Model]` (with `model`). The defensive `getattr` chains are therefore currently satisfied, not papering over drift.

## From: contracts phase (2026-08-18, v0.16.0)

Surface split with **verification status per surface** (convention C02):

| Surface | Contracts recovered | Evidence weight |
|---|---|---|
| Desktop GUI | F1–F10 (select, refresh, run, overwrite guard, streaming+copy, progress, preview/review, validation messages, completion dialog, shutdown) | **Mixed.** Validation messages, model-list normalization, and all six OS-integration branches are **verified** by the passing service suite. Progress formulas, streaming de-duplication, review navigation, tab switching, and the completion dialog rest on the five GUI suites, which **error at collection** without `tkinter` (D1.6) — therefore `asserted-not-verified`. |
| Storage / export | F11 — `<stem>_extracted.md` | **Verified.** UTF-8, LF-only normalization, `"\n\n"` joining, dotted-stem naming, atomic-replace failure behavior all pinned by passing service tests. |
| Outbound Ollama | F12 (per-page streaming chat), F13 (PDF rasterization) | **Verified.** Message shapes asserted field-for-field; all four rejection paths and the mid-stream exception wrap pass. Client response fields independently confirmed present by introspection. |

**Absent by verification** (`observed fact`): no CLI arguments, no network listener, no exported library API, no IPC, no bot or daemon surface.

**Surface-level caveat for a port:** the GUI is the *only* interactive surface and simultaneously the *least verified* half of the specification. 14 of the 42 acceptance scenarios are asserted-not-verified, all of them UI-side.

## From: protocols phase (2026-08-18, v0.16.0)

**Wire encoding of the outbound surface — resolved** (`observed fact`). The app passes page images to the Ollama client as **filesystem path strings**; the client's `Image.serialize_model` reads the file and **base64-encodes it into the JSON body**. Exercised directly: `Image(value=str(path)).model_dump()` returns a `str` beginning `iVBORw0KGgo` (PNG magic), with the path absent.

Consequences for the surface contract:
- The wire format is base64 file bytes. There is no path-passing shortcut for a port.
- **A remote server never receives the path** — the client reads locally and uploads bytes.
- A missing file raises a client-side `ValueError('File ... does not exist')` *inside* the app's remote-error wrapper, so a local filesystem problem is reported as `Ollama request failed … (model 'x')` (hazard H14, defect D2.10, convention C03).

**Six protocol boundaries** own the surfaces above: worker→UI event queue (B1), service callbacks (B2), Ollama HTTP (B3), output persistence (B4), the render directory (B5), and OS integration (B6). Configuration is explicitly **not** a boundary — `config.py` is imported directly with no serialization, precedence, or reload path.
