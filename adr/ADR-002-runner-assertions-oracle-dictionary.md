# ADR-002: The runner collects all oracle results into one dictionary; the assertion only reads it

Status: accepted

## Context

The L03–L09 ladder gives each case one oracle. Case `JNT-026` needs four sources at once:
the disputes engine (recorded in the case), the tool result (trace), the customer row and
the dispute rows (stand state). With `D26` each source alone looks right; only the pair is wrong.

## Decision

We collect the sources in the runner: `collect_oracles()` returns a dictionary with
`engine:disputes`, `tool:check_dispute_eligibility`, `state:customers` and `state:disputes`.
The assertion engine (`assert_case`, `joint_checks`) takes this dictionary and makes no HTTP calls.

## Consequences

- \+ The joint check reads several sources together and catches "each is right, together wrong":
  with `D26` two separate checks pass and `cross_dispute_under_hold` fails; on clean all pass.
- \+ Assertions can be tested without a stand (the step tests use a fake stand).
- − The runner must know which sources each case needs; a new kind of oracle means changing
  `collect_oracles()`, not only the assertion.
- − The state is read once, after the agent run. A problem that exists only in the middle of
  the run (and is fixed by the end) is not visible.
- Revisit if: a check needs the state at a moment during the run (for example between dialog
  turns), or `collect_oracles()` grows into a long list of special cases per case type.

## Alternatives

- Let the assertion call the stand itself: mixes checking with fetching, cannot be tested
  without a stand.
- Keep one oracle per case: cannot see a conflict between two sources.
