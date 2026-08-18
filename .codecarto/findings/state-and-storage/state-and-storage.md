
---

## From: architecture phase (2026-08-18, v0.16.0)

The system is close to stateless. One durable artifact; everything else is transient or in-memory.

| Kind | Present? | Detail |
|---|---|---|
| Config files | **No** | All configuration is hardcoded in `config.py`; nothing is read from disk at startup |
| Environment variables | **No** | No `os.environ` / `os.getenv` anywhere in first-party code |
| Auth material | **No** | Ollama has no built-in auth; the app sends no credentials (README flags the exposure risk) |
| Session / preferences | **No** | URL, model, and DPI are **not persisted** — every launch resets to `http://localhost:11434`, an empty model box, and DPI 150 |
| Logs | **In-memory only** | The Log tab is a Tk textbox; nothing is written to a file, so a crash loses all diagnostics |
| Caches | **In-memory only** | 5-entry LRU of decoded review images; raw page PNG bytes retained in `review_pages` for the whole run with **no eviction** |
| Databases | **No** | None |
| Generated artifacts | **Yes** | `<stem>_extracted.md` beside the input — the single durable output. Mutable whole-file replace, no history, no versioning |
| Temporary files | **Yes** | `local_ocr_*` mkdtemp holding `page_NNNN.png` renders, removed in `process_ocr`'s `finally` on every normal path; plus a hidden `.{stem}_*.tmp` staging file consumed by `os.replace` |

**Publication mechanism** (`observed fact` — `ocr_service.py:239-263`): `NamedTemporaryFile(delete=False, dir=output.parent, prefix=".{stem}_", suffix=".tmp")` → `flush()` → `os.replace`. Staging in the output directory keeps the rename same-filesystem, which is what makes it atomic. There is **no `fsync`**, so the guarantee holds against process death but not against OS crash or power loss.

**No locking, no deduplication, no resume** (`observed fact`): two instances targeting the same input race to last-writer-wins; a failed run leaves no journal or partial file; page boundaries are unrecoverable from the output, since a model-emitted blank line is indistinguishable from the `"\n\n"` page separator.

## From: contracts phase (2026-08-18, v0.16.0)

**F11 — the `_extracted.md` contract, fully verified** by the passing service suite (`observed fact`):

- **Naming:** `input.with_name(f"{stem}_extracted.md")` — always the input's directory. Dotted stems preserved (`report.v2.pdf` → `report.v2_extracted.md`).
- **Encoding:** UTF-8, no BOM. **LF only** — both `\r\n` and bare `\r` normalized, on every platform. A Windows port must not "fix" this.
- **Body:** pages joined with exactly `"\n\n"`, each `.strip()`ed. **No header, footer, page markers, or front-matter** — raw concatenated model output. Consequence: **page boundaries are unrecoverable from the file**, since a model-emitted blank line is indistinguishable from the separator.
- **Publication:** temp file staged in the *output directory* (keeping the rename same-filesystem) then `os.replace`.
- **Failure behavior:** on replace failure the original is preserved and the temp removed; on temp-creation failure no files appear at all; on success no leftovers remain.

**What is deliberately absent:** no history or versioning (the overwrite prompt is the only guard), no journal, no resume marker, no locking, no deduplication. Two instances targeting one input race to last-writer-wins.

**Durability gap:** atomic against process death, **not** against OS crash — there is no `fsync` before the rename, so the README's atomicity claim overstates the guarantee (D2.4, routed as `mech-CF4`).
