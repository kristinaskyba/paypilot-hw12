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
| <…> | <…> | <…> | <…> | <…> | <…> |

**Scripts that do several things at once, and where those parts are in `run_eval.py`:** <…>

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
| Entry point to cases | <…> |
| What is hashed | <…> |
| Algorithm and hash length in the record | <…> |
| `step2` tests | <… of 6 green> |

&nbsp;

| Check | `set_hash` before | `set_hash` after | Changed? | Expected |
| --- | --- | --- | --- | --- |
| Test case appended to `sets/l03.jsonl` | <…> | <…> | yes / no | yes |
| File saved with CRLF line endings | <…> | <…> | yes / no | no |
| Test case removed | <…> | <…> | yes / no | hash back to the first one |

**If the result did not match the expectation — what I fixed:** <…>

---

## 3. Runner and assertion engine

| Check | How I checked | Result |
| --- | --- | --- |
| `run_eval.py` is one entry point for all sets | <…> | yes / no |
| The assertion engine takes the oracle results dictionary | <…> | yes / no |
| The runner collects the dictionary, the assertion only reads it (no HTTP in the check) | <…> | yes / no |
| The judge does not get into the daily layer (`gate` field) | <…> | yes / no |
| `tests/manual/` and `tests/generated/` are a separate input, not a case set | <…> | yes / no |

**Oracle dictionary for `JNT-026`** (keys collected by the runner; the `SKIP` line from the `SET=l12` run): <…>

**`--dry-run` output:**

| Layer | Cases | Agent calls | Judge calls |
| --- | --- | --- | --- |
| daily (`GATE=daily`) | <…> | <…> | 0 |
| all (`GATE=all`) | <…> | <…> | <…> |
| L11 unit tests | manual <…>, generated <…> | — | — |

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
