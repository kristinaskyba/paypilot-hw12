# Eval Framework + ADR — PayPilot evaluation pipeline

**System:** PayPilot · **Date:** 2026-11-20 · **Mandate:** Ship it · **Author:** kristinaskyba · **HW repo:** https://github.com/kristinaskyba/paypilot-hw12 · **Kit commit:** `5db4dd0` · **Stand:** `74f1995`, profile `clean`, agent `claude-haiku-5-5` (stand default, `LLM_MODEL` empty) · **`JUDGE_MODEL`:** `ollama:qwen3:8b` (L10 recommendation; not used in the daily layer)

---

## Prediction before step 1

| What I predict | Now | After the lab |
| --- | --- | --- |
| How many commands I run by hand to get the full picture for sets L03–L11 | 6 | <…> |
| How many of the six version fields in my last report tell the truth | 4 | <…> |

**What changed and what it means (one sentence):** <…>

---

## 1. Inventory

| Script / command | Lesson | Component | Input | Output | Note |
| --- | --- | --- | --- | --- | --- |
| `loader.py`, `load(path)` | L03 | dataset manager | path to `.jsonl` | list of validated cases | format example |
| `validate.py` (`SET=…`) | L03 | dataset manager | set name | `set_hash` of the file bytes + summary | hashes file bytes; replaced in step 2 |
| `generate_from_engines.py` | L03 | dataset manager | engine rules | generated cases | writes cases, does not run them |
| `runner.py`, `main()` | L03–L09 | eval runner | `SET`, `GATE`, `CASE` | verdicts, report file, printed summary | does several things at once: loads and filters, runs, asserts, writes the report, prints |
| `cli.py --set … --dry-run` | L05–L07 | eval runner | CLI flags | calls `runner.py` | wrapper; replaced by environment variables |
| `stand.py` | L03 | eval runner | HTTP to the stand | answers, traces, state | stand client |
| `assertions.py` / `runner.evaluate()` | L03–L09 | assertion engine | case and agent runs | verdict |  |
| `judge.py` | L09 | assertion engine | question, answer, rubric | judge verdict | release layer only |
| `console.py` | L03 | reporting | results | printed table |  |
| `run_dialog.py` | L08 | eval runner | dialog cases | transcripts, verdicts | not in the L12 kit |
| `tools/calibrate_run.py` | L09–L10 | separate input | labelled set, `JUDGE_MODEL` | calibration report | judge calibration |
| `tools/compare_judges.py` | L10 | separate input | two calibration reports | comparison |  |
| `pytest tests/manual tests/generated` | L11 | separate input | test files | pass / fail | unit tests |
| `tools/run_mutants.py` | L11 | separate input | `mutants.toml` | mutation score |  |

**Scripts that do several things at once, and where those parts are in `run_eval.py`:** `runner.main()` loads and filters cases, runs them, asserts, writes the report and prints the summary. In `run_eval.py` these are split into sections 1 (dataset manager), 2 (eval runner), 3 (assertion engine), 4 (result storage) and 5 (reporting).

### Pipeline components

| Component | File / function in my pipeline | Takes | Returns | What it does not know |
| --- | --- | --- | --- | --- |
| dataset manager | <…> | set names, `gate` / `CASE` filters | list of cases and `set_hash` | <…> |
| eval runner | <…> | cases, `PROFILE`, `DEFECTS`, `JUDGE_MODEL` | answers, traces, oracle results dictionary | <…> |
| assertion engine | <…> | case, oracle results dictionary | verdict per case and level | <…> |
| result storage | <…> | verdicts and run conditions | one run record | <…> |
| reporting | <…> | two run records | difference in conditions, cases that changed verdict, exit code | <…> |

---

## 2. Dataset manager and `set_hash`

| Field | Value |
| --- | --- |
| Entry point to cases | `load_cases(names, gate, only)` in section 1 of `run_eval.py` |
| What is hashed | the content of the cases that went into the run (after the `GATE` and `CASE` filters), each case as canonical JSON (`sort_keys`, no spaces), sorted by id; not the file bytes, names, paths or time |
| Algorithm and hash length in the record | SHA-256 of the lines joined with `\n`, first 12 hex characters |
| `step2` tests | 6 of 6 green |

&nbsp;

| Check | `set_hash` before | `set_hash` after | Changed? | Expected |
| --- | --- | --- | --- | --- |
| Test case appended to `sets/l03.jsonl` (copy of `FX-001` as `FX-001-COPY`) | `2eeadf91c793` | `78dacc031336` | yes | yes |
| File saved with CRLF line endings | `78dacc031336` | `78dacc031336` | no | no |
| Test case removed | `78dacc031336` | `2eeadf91c793` | yes | hash back to the first one |

**If the result did not match the expectation — what I fixed:** nothing, all three checks matched. `2eeadf91c793` is the same as the reference `SET=l03` hash. The file was restored with `git checkout -- sets/l03.jsonl`.

---

## 3. Runner and assertion engine

| Check | How I checked | Result |
| --- | --- | --- |
| `run_eval.py` is one entry point for all sets | `docker compose run --rm eval --dry-run`: one command lists all 7 sets (l03–l08, l12), `set_hash 58cf5b498718` | yes |
| The assertion engine takes the oracle results dictionary | `assert_case(case, answer, oracles: dict)` | yes |
| The runner collects the dictionary, the assertion only reads it (no HTTP in the check) | `collect_oracles()` in section 2 calls the stand (`tool_calls`, `state`); `assert_case()` / `joint_checks()` only get the dictionary | yes |
| The judge does not get into the daily layer (`gate` field) | `--dry-run`: 0 judge calls; `GATE=all --dry-run`: 40 judge calls, all in release cases | yes |
| `tests/manual/` and `tests/generated/` are a separate input, not a case set | `--dry-run` prints a separate line `unit-тести L11: manual — 6, generated — 27`, not counted in the 71 cases | yes |

**Oracle dictionary for `JNT-026`** (keys collected by the runner; the `SKIP` line from the `SET=l12` run): `engine:disputes`, `tool:check_dispute_eligibility`, `state:customers`, `state:disputes` (SKIP: `joint_checks()` not written yet; 1 agent call, $0.0009, exit 1; the SKIP record was deleted)

**`--dry-run` output:**

| Layer | Cases | Agent calls | Judge calls |
| --- | --- | --- | --- |
| daily (`GATE=daily`) | 71 | 72 | 0 |
| all (`GATE=all`) | 90 | 264 | 40 |
| L11 unit tests | manual 6, generated 27 | — | — |

---

## 4. Run record and two runs side by side

**How I switched the prompt:** <…>

**What the stand said after switching** (`curl -s http://localhost:8000/api/_test/prompt`): field `version` — <…>, first line of `text` — <…>

| Field | Run 1 | Run 2 | Same? |
| --- | --- | --- | --- |
| Record file in `reports/` | <…>.json | <…>.json | — |
| `set` / `gate` | <…> | <…> |  |
| `set_hash` | <…> | <…> | must match |
| `prompt_version` | `base.v1` | `base.v4` | must differ |
| `stand_prompt_label` | <…> | <…> |  |
| `profile` | <…> | <…> |  |
| `judge_model` | <…> | <…> |  |
| `rubric_version` | <…> | <…> |  |
| `elapsed`, s | <…> | <…> | — |
| `passed` / `failed` | <…> | <…> | — |
| `machine.cost_usd` | <…> | <…> | — |

**Cases that changed verdict** (ID and assertion type): <…>

**Noise** (how many cases change between two runs under the same conditions: my own measurement or the reference one): <…>

**What changed between the runs (one sentence):** <…>

**Minutes from "run finished" to "I know which cases changed verdict":** by hand, reading JSON (step 4) — <…> min · with `--compare` (step 6) — <…> min

---

## 5. Joint check

| Field | Value |
| --- | --- |
| Case | `JNT-026`: `CUS-0006`, `TX-0601`, goods not received |
| Commands | `-e SET=l12` and `-e SET=l12 -e DEFECTS=D26` |
| Where the assertion gets the state from | <…> |
| Who fills the oracle dictionary | <…> |

**Three report lines:**

| Line | Level | `clean` | `DEFECTS=D26` |
| --- | --- | --- | --- |
| `disputes_window_ok` | <…> | <…> | <…> |
| `compliance_hold_flag` | <…> | <…> | <…> |
| `cross_dispute_under_hold` | <…> | <…> | <…> |

**What the joint assertion reads (one sentence):** <…>

**Findings in the set** (cases that are red not because of the agent, and why): <…>

**Agent answer with `D26`** (first 200 characters from the record): <…>

**Another PayPilot source pair where the "each is right, together wrong" class is possible** (optional): <…>

---

## 6. One run on a clean clone

| Run contract item | How I checked | Holds? |
| --- | --- | --- |
| Asks nothing from the keyboard | <…> | yes / no |
| Takes everything that varies from environment variables: `SET`, `GATE`, `CASE`, `PROFILE`, `DEFECTS`, `JUDGE_MODEL` | <…> | yes / no |
| Writes a record to `reports/` with all six version fields | <…> | yes / no |
| Returns a non-zero code if any check fails | <…> | yes / no |
| `--dry-run` counts cases and calls without spending | <…> | yes / no |

**Commands on the clean clone:**

```text
git clone <HW repo> l12-clean
cd l12-clean
docker compose run --rm eval --dry-run
docker compose run --rm eval
<exit code>
docker compose run --rm eval --compare reports/<v1>.json reports/<v4>.json
docker compose run --rm --entrypoint pytest eval tests -q
git add reports, git commit, git push
```

| Field | Value |
| --- | --- |
| Manual steps between cloning and the record | <…> |
| Run record | `reports/<…>.json` |
| `passed` / `failed`, exit code | <…> |
| `unit_tests` block in the record | manual <…>, generated <…> |
| `--compare` output for the two step 4 records (conclusion line) | <…> |
| What I had to fix to make the clone work | <…> |

### Run cost in two currencies

| Currency | Value | Source |
| --- | --- | --- |
| Machine: minutes | <…> | `elapsed` in the record |
| Machine: money | <…> | `machine.cost_usd` in the record |
| Human: minutes on "what changed" | by hand <…> · with reporting <…> | section 4 |

---

## 7. ADR

| # | Boundary | Decision (one sentence) | File | Status |
| --- | --- | --- | --- | --- |
| ADR-001 | dataset ↔ runner | <…> | `adr/ADR-001-….md` | <…> |
| ADR-002 | runner ↔ assertions | <…> | `adr/ADR-002-….md` | <…> |
| ADR-003 | assertions ↔ engines | <…> | `adr/ADR-003-….md` | <…> |

**Check of the three decision lines:**

| ADR | What we traded (minus in Consequences) | What we do not do (Alternatives) | When we will revisit |
| --- | --- | --- | --- |
| ADR-001 | yes / no | yes / no | yes / no |
| ADR-002 | yes / no | yes / no | yes / no |
| ADR-003 | yes / no | yes / no | yes / no |

### Assistant perimeter (a separate line, not an ADR)

| The assistant may change | The assistant does not change |
| --- | --- |
| <…> | <…> |

---

## 8. AI use declaration and cascade readiness

**AI declaration:** <…>

**Cascade readiness (chain-ready):**

| Check on a clean clone | Passes |
| --- | --- |
| `docker compose run --rm eval` finishes without manual steps | yes / no |
| A record with all six version fields appears in `reports/` | yes / no |
| Non-zero exit code when a check is red | yes / no |
| L11 unit tests run with the same command (`unit_tests` block: `manual.exit_code` and `generated.exit_code` are `0`) | yes / no |
| All 20 step tests are green | yes / no |

**Attachments:** `run_eval.py` · `reports/<v1>.json` · `reports/<v4>.json` · `reports/<clean, SET=l12>.json` · `reports/<D26>.json` · `reports/<clean clone>.json` · `adr/`
