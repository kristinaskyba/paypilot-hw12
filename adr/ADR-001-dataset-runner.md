# ADR-001: The dataset manager gives the runner cases and a content hash; the runner never reads set files

Status: accepted

## Context

Before L12 the cases were loaded in several places (`loader.py`, `validate.py`, `runner.main()`),
and `validate.py` hashed the bytes of one file. The same cases could get a different hash after
a re-save with CRLF, and a hash of one file said nothing about a run over several sets or
over a `GATE`/`CASE` filter.

## Decision

We load and filter cases only in `load_cases()` (section 1 of `run_eval.py`). It returns the
list of cases and `set_hash`: SHA-256 of the cases that went into the run, each as canonical
JSON (sorted keys, no spaces), sorted by id, first 12 characters. The runner only gets the list.

## Consequences

- \+ Two runs can be compared: same `set_hash` means the same cases (checked: an appended case
  changes the hash, CRLF does not, removing the case brings it back).
- − A different `GATE` or `CASE` filter gives a different hash, so a daily run and a full run
  are never "the same set", even over the same files.
- − The old `loader.set_hash()` and `validate.py` stay in the kit but are no longer used.
- Revisit if: cases move out of `.jsonl` files (a database, a shared store), or a case starts
  pointing to external files (fixtures) whose content the hash does not see.

## Alternatives

- Hash the file bytes: breaks on CRLF and key order.
- Use the git commit as the set version: does not show uncommitted edits or which filter ran.
- Hash all set files whatever the filter: hides which cases actually ran.
