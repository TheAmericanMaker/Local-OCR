# Closeout — protocols

## Summary

- Formalized six protocol boundaries with the nine required fields each, gave
  all nine event kinds explicit payload schemas, and tabulated four state
  machines — closing `arch-CF3` and `contracts-CF3`.
- Recorded 14 compatibility hazards, three `high`.

## Resolved: how page images actually reach the server

The previous pass called this the single largest unknown for a cross-language
port and deferred it as third-party inspection. Convention **C01** made
reaching for it the default, and it took one read plus one call:

The app passes a **path string**; the client's `Image.serialize_model` reads
the file and **base64-encodes it into the JSON body**. Exercised directly,
`Image(value=str(path)).model_dump()` returns a `str` beginning
`iVBORw0KGgo` — PNG magic — with the path **absent**.

So: the wire format is base64 file bytes; a remote server never sees the
path; and a port must base64 the bytes with the file readable and unmoved for
the whole request.

The same read produced **hazard H14**: branch 3 of that serializer raises a
*local* `ValueError('File ... does not exist')` from inside the app's
*remote*-error wrapper, so a missing render is reported as `Ollama request
failed … (model 'x')`. That is the already-live form of the misattribution
convention **C03** was promoted to prevent.

## The property a port should build around

B1's events split three ways, and the split decides what may be dropped:

- **Observational** (`log`, `progress`, `page_image`, `stream_chunk`,
  `page_text`) — UI only. The saved Markdown is built from
  `recognize_images`' *return value*, so losing every one still produces a
  byte-identical file.
- **Synchronous barriers** — three, inside the worker, invisible to the queue.
- **State-bearing** (`models_loaded`, `refresh_error`, `ocr_success`,
  `ocr_error`) — the only four that drive a transition. Losing one strands
  the machine permanently.

That makes D2.6 precise: event loss is cosmetic except for four kinds.

## State machine properties, and their evidence

SM1 has **no cancellation transition**, **no error state**, and exactly four
exits from a busy state. The missing error state is why the app can never
offer "retry."

SM2, SM3, and SM4 are real machines with real guards — but they are pinned
**only** by the GUI suites, which error at collection here. Per C02/C04 they
are labelled `asserted-not-verified`, making them the weakest-evidenced part
of this document rather than silently equal to SM1.

## Routing

- Closed `arch-CF3` and `contracts-CF3`.
- `protocols-CF1` → `porting` (event schema and state machine must be
  extended together for cancellation/retry/concurrency/resume).
- `protocols-CF2` → `defect-scan-semantic` (provider drift; note the fields
  currently match, which is evidence to weigh rather than a reason to drop).
- Opened `q-queue-depth-under-load`; **did not** carry
  `q-ollama-image-encoding`, which is now resolved.
- Proposed one convention: *read the dependency when its behavior is the
  contract*.
