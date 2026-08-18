# Closeout — reimplementation-spec

## Summary

- Produced a **language-agnostic** reimplementation spec (front matter
  `selection: language-agnostic`): nine concept modules, 44 numbered MUST
  rules, 35 black-box acceptance scenarios, a kernel-first implementation
  sequence in nine phases, three scope tiers, and eight spikes.
- Closed `porting-CF1`, `porting-CF2`, and `porting-CF3`.
- **All 41 defects from both scans are dispositioned** — the 4 high and 16
  medium each became a numbered MUST rule citing its defect ID; the 21 low
  were absorbed into rules or named explicitly as left-behind non-goals.

## The one deliberate contract break

R1 replaces the original's all-or-nothing discard with first-class partial
success. Because two **passing** tests assert the old behavior, the spec names
the split at the rule itself:

- **Preserved (R1.4):** never leave the canonical artifact corrupt or partial,
  and never destroy it except by a successful complete write.
- **Dropped:** discard all completed work when any page fails.

Cost: exactly one test needs rewording. `test_late_page_failure_preserves_
existing_output` is satisfied by R1.4 unchanged.

## What execution bought this spec

Nine scenarios encode behavior the original **demonstrably fails**, and none
of them would exist from reading alone: the proven temp-directory leak on quit
(A11, A12), the local-error-as-remote misattribution (A13), the silent
coercion of an unrecognized response shape (A14), the overwrite guard that
checks minutes too early (A21), the platform action that cannot report failure
(A28), and the headless test run that errors instead of skipping (A29).

R10 is the unusual one — it constrains the port's *own test harness*. Kept
deliberately: the original's misplaced skip guard is what hid ~470 lines of
never-executed contracts, and it is why 14 acceptance rows carry the `A`
(asserted-not-verified) origin tag rather than `V`.

## Compression boundary held

Only **three** targeted deep reads were needed beyond the bundle, each against
a trigger the bundle named explicitly: the acceptance list, the event catalog
plus SM1, and two defect rationales. The architecture map, contracts,
protocols, and both scan reports were never reloaded wholesale.

## What remains open

Six Known Unknowns, and the two that matter most are the same decision: **no
target stack and no scope tier are committed**, so hazard mitigations are
stated as requirements rather than techniques. The right follow-up is an
opinionated re-run once that decision exists (`spec-CF2`,
`pp-opinionated-spec-rerun`) — recorded as an amendment, not a gap.

Two spikes carry real engineering risk: `spec-CF1` sets R2's numeric
thresholds by measurement rather than guesswork, and `spec-CF3` confirms the
provider envelope so R4's error taxonomy maps onto real failures.
