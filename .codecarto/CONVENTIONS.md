# Conventions

<!--
  Project-level skeleton. Copy this to `.codecarto/CONVENTIONS.md` (one level up from templates/)
  the first time the orchestrator promotes a convention. Then add entries as they accumulate.

  This file holds cross-cutting patterns that have been promoted to project-wide invariants.
  Every new session reads this file and either honors these conventions or documents why it
  diverges.

  This file is **orchestrator-maintained**. Phase executors propose additions in their session
  closeout; the orchestrator promotes them at the phase boundary. In an inline run the same chat
  does both — the rule is about *when* (between phases, deliberately), not about which thread.
-->

Cross-cutting patterns promoted to project-wide invariants. Every session reads this file at start
and either honors these conventions or documents why it diverges.

This file is **orchestrator-maintained**. Phase executors propose additions in their closeout's
"Proposed Conventions" section; the orchestrator promotes them here at the phase boundary — in an
inline run, the same chat changing hats between phases.

## How conventions get added

A new entry lands here when ONE of the following holds:

1. **Three independent sessions** reach for the same pattern (the "lift if it generalizes" rule
   applied to conventions themselves), OR
2. **One session** explicitly promotes a pattern in its closeout report and the orchestrator
   confirms it generalizes, OR
3. **The spec or framework feedback corpus** identifies a project-wide invariant that future
   implementing sessions need to know about.

The orchestrator owns this file. Implementing sessions propose; orchestrator promotes.

## Entry shape

Each convention is a numbered section (`## C<NN>. <Title>`) with three required parts:

- **Body** — the rule itself, in prose. May include a code block for shape contracts.
- **Why:** — the reason the rule exists. Often a past incident or a defect class the rule
  prevents. Future maintainers judging edge cases need to know *why* to judge whether the rule
  applies.
- **How to apply:** — when and where the rule kicks in. Should answer "is this case in scope?"

Optional:
- **Current implementers:** — files/modules that already follow the rule. Useful as worked examples.
- **Source:** — the closeout entry where the orchestrator promoted this convention.

---

## C01. Execute the headless layer instead of inferring it

Where a layer can run without the GUI toolkit, **run it and cite the result** rather than reasoning
about it. Prefer executed evidence over `strong inference` whenever the cost is a package install.
Record the exact command in the phase's Coverage and limits section, and mark the resulting claims
`observed fact` rather than `strong inference`.

**Why:** the architecture phase asserted, from a module docstring, that `ocr_service.py` is Tk-free
and therefore headlessly testable. Installing three packages and running the suite on a host with no
`tkinter` proved it (72 passed / 1 skipped, 0.52 s) — and in the same pass revealed `arch-CF6`, a
real defect that pure reading had concluded the *opposite* about. Reading produced a confident wrong
answer where execution produced a cheap right one.

**How to apply:** applies whenever a phase is about to record a `strong inference` that a test run,
an import, or a small script could settle. Does **not** apply to layers that genuinely need the
absent dependency — this host has no `tkinter`, so GUI behavior stays `needs-runtime-test`
(`q-shutdown-tempdir`). Establish which half of the system is reachable before concluding a claim is
untestable.

**Current implementers:** the architecture phase (service suite execution; `ollama` client type
introspection).

**Source:** `closeouts/2026-08-18-architecture.md`.

---

## C02. A test's existence is not evidence that it runs

Never treat the presence of a test — or its own docstring about when it skips — as evidence that the
behavior is verified. Establish separately whether the test actually executes in the environment it
claims to handle. Contracts backed only by tests of unproven execution must be marked
**asserted-not-verified**, not `observed fact`.

**Why:** the five GUI suites each document that they "are skipped automatically when no display is
available (CI, headless containers)" and each guards `LocalOCRApp()` in `setUp` with a `skipTest`
fallback. But `import app as app_module` sits at module scope, so on a host without `tkinter` the
import raises before any skip logic runs — `unittest discover` reports **5 errors, not 5 skips**. The
~470 lines of UI contracts they pin have very likely never executed, while the service suite passes
cleanly. A guard placed after the thing it guards is not a guard.

**How to apply:** applies to every phase that cites a test as evidence — most heavily `contracts`,
whose rubric treats tests as executable specifications. When citing a test, note whether it is known
to pass, known to skip, known to error, or unverified. Applies equally to any skip/guard construct:
check that the guard precedes what it protects.

**Current implementers:** the architecture phase (`arch-CF6`).

**Source:** `closeouts/2026-08-18-architecture.md`.

---

## C03. Name the subsystem the error actually came from

An error message must name the subsystem that **actually** failed, not the one the call was aimed at.
When a `try` block spans both local work and a remote call, classify the cause before attributing
blame in the text a user will read.

**Why:** `recognize_images` wraps everything in its `try` as `Ollama request failed on page {n}/{N}
(model {model!r})`. But the page image is passed to the client as a *path string*, and the client
resolves it locally — raising `ValueError('File ... does not exist')` when the render is missing. So a
purely local filesystem problem is reported as a server failure with the user's model name attached,
pointing debugging at the wrong machine (D2.10). The severe form of the same weakness — a renamed
provider field surfacing as "returned no text", which the README attributes to a non-vision model — is
routed as `mech-CF5`.

**How to apply:** applies whenever reviewing or specifying error handling around a call that mixes
local and remote failure modes. Ask what the *narrowest* true statement about the failure is, and
whether the message would send a reader to the right subsystem. A wrapper that widens blame is worse
than one that says less.

**Current implementers:** the mechanical defect scan (D2.10); `mech-CF5` extends it.

**Source:** `closeouts/2026-08-18-defect-scan-mechanical.md`.

---

## C04. Record verification status alongside every acceptance scenario

An acceptance list must say, **per scenario**, whether an equivalent assertion currently passes,
exists but is unverified, or does not exist at all. A flat list silently averages strong and weak
evidence and invites a reader to assume uniform coverage.

**Why:** Local-OCR's 42 scenarios split **28 verified / 14 asserted-not-verified / 2 known-failing**.
The 14 rest on the five GUI suites, which error at collection and have very likely never run
(D1.6); without the column they read exactly like the 28 backed by the passing 1,069-line service
suite. The two known-failing rows (41, 42) would also have been invisible as *failures* rather than
aspirations.

**How to apply:** applies to `contracts` when building the acceptance list and to
`reimplementation-spec` when turning it into a parity harness. Carry the status forward — a spec that
drops it hands the implementer a harness whose weak rows are indistinguishable from its strong ones.
This is C02 applied at list granularity rather than per claim.

**Current implementers:** the contracts phase (§Black-Box Acceptance List, **V** column).

**Source:** `closeouts/2026-08-18-contracts.md`.

---

## C05. Read the dependency when its behavior is the contract

When a third-party call site **is** the protocol — the app hands over a value and the library decides
what crosses the wire — read or exercise that library's own code rather than deferring it as
out-of-scope. "Third-party internals are out of scope" applies to *auditing their quality*, not to
*establishing the contract the port must reproduce*.

**Why:** the wire encoding of page images was the previous run's self-declared largest unknown for a
cross-language port, deferred precisely because it required third-party inspection. Settling it took
one read of `Image.serialize_model` and one `model_dump()` call: base64 file bytes in the JSON body,
path never sent. The same read produced hazard **H14** — a local `ValueError` raised from inside the
app's remote-error wrapper. A question can be both out of scope to audit and trivial to answer.

**How to apply:** applies when a phase is about to record an `open_question` about behavior that lives
in a dependency the app hands data to — serializers, encoders, transport clients, format writers. Ask
whether one read or one call would settle it. Does **not** license auditing the dependency for
defects; the finding is the *contract*, not the library's quality. Complements C01: C01 says execute
your own layer, C05 says read the layer you delegate to.

**Current implementers:** the protocols phase (§B3 wire encoding; hazards H1, H14).

**Source:** `closeouts/2026-08-18-protocols.md`.

---

## C06. <Title>


<!-- Lead with the rule itself. -->

**Why:**

**How to apply:**

**Current implementers:**

**Source:**

---

<!-- Repeat the C<NN> block for each convention. Number sequentially. -->

## Pending proposals

Staged by completion from each phase handoff's `proposed_conventions`. The orchestrator promotes an entry into a numbered convention above (or removes it with a note) at the phase boundary — see GUIDE.md §Roles.

*(none pending — C01/C02 at architecture; C03 at defect-scan-mechanical; C04 at contracts; C05 at protocols.)*
