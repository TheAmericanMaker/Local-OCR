# Closeout — protocols

## Summary

- Formalized six protocol boundaries, each with the nine required fields:
  the worker→UI event queue, the service callback protocol, Ollama HTTP,
  output persistence, the render directory, and OS integration.
- Gave all nine event kinds explicit payload schemas with producer and
  handler, closing `arch-CF3` and `contracts-CF3`.
- Tabulated four state machines (SM1 `OperationState` with 21 transitions,
  plus progress mode, result-panel streaming, and review navigation) with
  guards and side effects.
- Recorded 13 compatibility hazards, three of them `high`.

## The finding worth carrying forward

B1's events split three ways, and the split decides what a port may drop:

- **Observational** (`log`, `progress`, `page_image`, `stream_chunk`,
  `page_text`) — UI only. The saved Markdown is built from
  `recognize_images`' *return value*, not from the event stream, so losing
  every one of these still produces a byte-identical file.
- **Synchronous barriers** — three, all inside the worker and invisible to
  the queue: render-all before recognize-any, recognize-all before save,
  cleanup before the terminal event.
- **State-bearing** (`models_loaded`, `refresh_error`, `ocr_success`,
  `ocr_error`) — the only four that drive a state transition. Losing one
  strands the machine permanently.

That makes D2.6 precise: event loss is cosmetic except for four specific
kinds, which need guaranteed delivery.

## State machine properties

SM1 has no cancellation transition, no error state, and exactly four exits
from a busy state. The absence of an error state is why the app can never
offer "retry" — it keeps no memory of the last outcome.

## Top hazards

- **H1 (high)** — pages are passed to Ollama as filesystem *path strings*,
  not bytes. What the client does with them is `q-ollama-image-encoding`,
  the most consequential unknown for a cross-language port.
- **H2, H3 (high)** — Tk thread affinity with 50 ms `after()` polling, and
  daemon-thread abandonment semantics. Neither transliterates.
- **H8 (medium)** — the defensive `getattr` reads turn a provider field
  rename into a silent, total, *misdiagnosed* failure. Routed to the
  semantic scan.

## Routing

- Closed `arch-CF3` and `contracts-CF3`.
- `protocols-CF1` → `porting` (event schema and state machine must be
  extended together for cancellation/concurrency/resume).
- `protocols-CF2` → `defect-scan-semantic` (provider-contract drift).
- Opened `q-ollama-image-encoding` and `q-queue-depth-under-load`.
