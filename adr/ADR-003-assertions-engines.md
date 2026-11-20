# ADR-003: Assertions compare with the engine answer recorded in the case, not with a live engine call

Status: accepted

## Context

PayPilot has rule engines (disputes, fx, limits, policy) in the stand code. The eval needs
their answer as the expected result. The engines are part of the system under test, and
the stand runs with a fixed clock (`CLOCK_OVERRIDE` 2026-09-15).

## Decision

We record the engine answer in the case when the case is made (`generate_from_engines.py`,
`additional_metadata.engine`). At run time the assertion reads that recorded answer from
the oracle dictionary. The eval does not import or call the engines while it runs.

## Consequences

- \+ The expected result is fixed and visible in the set; if it changes, `set_hash` changes too.
- \+ A defect in the stand's engine code cannot silently change the expected result.
- − If the engine rules change on purpose, the recorded answers become stale until the cases
  are regenerated; nothing warns us automatically.
- − Recorded answers depend on the fixed clock; with another date they may no longer be true.
- Revisit if: the engine rules start changing often, or the stand stops using a fixed clock.

## Alternatives

- Call the engine live in the assertion: the code under test becomes its own oracle, and the
  eval depends on the stand's code version.
- Write the expected results by hand: slower and easier to get wrong than generating them
  from the engines once.
