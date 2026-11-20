# Eval Framework + ADR — PayPilot evaluation pipeline

**System:** PayPilot · **Date:** 2026-11-20 · **Mandate:** Ship it · **Author:** kristinaskyba · **HW repo:** https://github.com/kristinaskyba/paypilot-hw12 · **Kit commit:** `5db4dd0` · **Stand:** `74f1995`, profile `clean`, agent `claude-haiku-5-5` (stand default, `LLM_MODEL` empty) · **`JUDGE_MODEL`:** `ollama:qwen3:8b` (L10 recommendation; not used in the daily layer)

---

## Prediction before step 1

| What I predict | Now | After the lab |
| --- | --- | --- |
| How many commands I run by hand to get the full picture for sets L03–L11 | 6 | 1 (`docker compose run --rm eval`) |
| How many of the six version fields in my last report tell the truth | 4 | 6 |

**What changed and what it means (one sentence):** One command now runs all daily sets plus the L11 unit tests and writes a record whose six fields are all true (the prompt version comes from the prompt text, not the stand label), so any two runs can be put side by side.

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
| dataset manager | `load_cases()`, `set_hash()`, `_raw_cases()` (section 1 of `run_eval.py`) | set names, `gate` / `CASE` filters | list of cases and `set_hash` | whether a stand exists, or whether a judge will be called on the set |
| eval runner | `run_case()`, `collect_oracles()`, `runner._one_run()`, `stand.py` (section 2) | cases, `PROFILE`, `DEFECTS`, `JUDGE_MODEL` | answers, traces, oracle results dictionary | how a verdict is decided, or which check passes |
| assertion engine | `assert_case()`, `joint_checks()`, `assertions.py` (section 3) | case, oracle results dictionary | verdict per case and level | how to reach the stand: it only reads the oracle dictionary |
| result storage | `write_record()`, `prompt_version()` (section 4) | verdicts and run conditions | one run record | which cases are red for which reason, or what an earlier run contained |
| reporting | `compare()` (section 5), `console.py` | two run records | difference in conditions, cases that changed verdict, exit code | how the cases were run: it only reads two record files |

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

**How I switched the prompt:** to `base.v4` (in `paypilot-stand`): `docker compose -f docker-compose.yml -f ../l11/agent-prompt/compose.v4.yml up -d` / back to `base.v1`: `docker compose up -d`

**What the stand said after switching** (`curl -s http://localhost:8000/api/_test/prompt`): field `version` — `base.v1`, first line of `text` — `# PayPilot — system prompt (base.v4)`

| Field | Run 1 | Run 2 | Same? |
| --- | --- | --- | --- |
| Record file in `reports/` | `run-20261120T144625Z.json` | `run-20261120T145022Z.json` | — |
| `set` / `gate` | `l03` / `daily` | `l03` / `daily` | yes |
| `set_hash` | `2eeadf91c793` | `2eeadf91c793` | yes |
| `prompt_version` | `base.v1` | `base.v4` | differs, as it must |
| `stand_prompt_label` | `base.v1` | `base.v1` | yes (the label lies for v4) |
| `profile` | `clean` | `clean` | yes |
| `judge_model` | `not-used` | `not-used` | yes |
| `rubric_version` | `rubric.v1` | `rubric.v1` | yes |
| `elapsed`, s | 135.6 | 142.6 | — |
| `passed` / `failed` | 33 / 4 | 32 / 5 | — |
| `machine.cost_usd` | 0.0149 | 0.0157 | — |

**Cases that changed verdict** (ID and assertion type): `DIS-004-N` (pass → FAIL; forbidden-pattern assertion, no `customer_id` in the case). Red in both runs: `DIS-002-N`, `DIS-006`, `DIS-007` (no `customer_id` in the case — a set defect) and `ORD-001` (extra `get_account` call — agent behaviour).

**Noise** (how many cases change between two runs under the same conditions: my own measurement or the reference one): reference noise from the lesson — 1 case of 71 changes verdict between two runs under the same conditions; `DIS-004-N` is one of the cases that flipped between two reference `base.v4` runs.

**What changed between the runs (one sentence):** One condition changed — the prompt `base.v1` → `base.v4` on the same `set_hash` — and one case (`DIS-004-N`) flipped, which is within the reference noise of 1 case, so the L03 set shows no real difference between the two prompts.

**Minutes from "run finished" to "I know which cases changed verdict":** by hand, reading JSON (step 4) — a few minutes (not timed) · with `--compare` (step 6) — about 2 seconds

---

## 5. Joint check

| Field | Value |
| --- | --- |
| Case | `JNT-026`: `CUS-0006`, `TX-0601`, goods not received |
| Commands | `-e SET=l12` and `-e SET=l12 -e DEFECTS=D26` |
| Where the assertion gets the state from | only from the oracle dictionary: `state:customers` and `state:disputes` (stand tables), `tool:check_dispute_eligibility` (trace), `engine:disputes` (the case) |
| Who fills the oracle dictionary | the eval runner, `collect_oracles()` in section 2 of `run_eval.py`; the assertion makes no HTTP calls |

**Three report lines:**

| Line | Level | `clean` | `DEFECTS=D26` |
| --- | --- | --- | --- |
| `disputes_window_ok` | 1 | pass | pass |
| `compliance_hold_flag` | 1 | pass | pass |
| `cross_dispute_under_hold` | 4 | pass (`engine: eligible=False · tool: eligible=False · disputes rows: 0`) | FAIL (`engine: eligible=False · tool: eligible=True · disputes rows: 1`) |

**What the joint assertion reads (one sentence):** It reads the engine verdict, the tool result and the dispute rows from the oracle dictionary, and fails when the engine says "not eligible" while the tool says "eligible" or a dispute row exists for the transaction.

**Findings in the set** (cases that are red not because of the agent, and why): `DIS-002-N`, `DIS-004-N`, `DIS-006`, `DIS-007` (L03) and `TOOL-402` (L07) have no `customer_id`, so the stand answers "no customer is signed in" — a set defect. `ORD-001` is agent behaviour: an extra `get_account` call. I did not edit these cases, so that `set_hash` stays comparable.

**Agent answer with `D26`** (first 200 characters from the record): "I've opened a dispute for transaction TX-0601 under the reason "goods not received." - **Dispute ID:** 1 - **Status:** Open - **Eligibility:** Passed all checks. The transaction date is 16 August 202" — the text is truthful about what the agent did, so only the state shows the defect.

---

## 6. One run on a clean clone

| Run contract item | How I checked | Holds? |
| --- | --- | --- |
| Asks nothing from the keyboard | full run on the clean clone finished without input | yes |
| Takes everything that varies from environment variables: `SET`, `GATE`, `CASE`, `PROFILE`, `DEFECTS`, `JUDGE_MODEL` | steps 3–5 used `-e SET=…`, `-e GATE=all`, `-e DEFECTS=D26`; no flags or code edits | yes |
| Writes a record to `reports/` with all six version fields | `reports/run-20261120T150022Z.json` has `prompt_version`, `set_hash`, `profile`, `judge_model`, `rubric_version`, `elapsed` | yes |
| Returns a non-zero code if any check fails | `echo $?` → `1` (5 red cases) | yes |
| `--dry-run` counts cases and calls without spending | prints 71 cases, 72 agent calls, 0 judge calls, `нічого не викликано` | yes |

**Commands on the clean clone:**

```text
git clone git@github-personal:kristinaskyba/paypilot-hw12.git l12-clean
cd l12-clean
cp ../hw12/.env .env    # environment setting: L11_DIR=../hw11
docker compose run --rm eval --dry-run
docker compose run --rm eval
echo $?   # 1
docker compose run --rm eval --compare reports/<v1>.json reports/<v4>.json
docker compose run --rm --entrypoint pytest eval tests -q
git add reports, git commit, git push
```

| Field | Value |
| --- | --- |
| Manual steps between cloning and the record | 0 (only the environment `.env` with `L11_DIR=../hw11`) |
| Run record | `reports/run-20261120T150022Z.json` |
| `passed` / `failed`, exit code | 66 / 5, exit code 1 (red: `DIS-002-N`, `DIS-006`, `DIS-007`, `TOOL-402` — no `customer_id`; `ORD-001` — agent behaviour) |
| `unit_tests` block in the record | manual 6 passed (exit 0), generated 27 passed (exit 0) |
| `--compare` output for the two step 4 records (conclusion line) | `comparable: one condition changed — prompt_version`; changed verdict: `DIS-004-N: pass -> FAIL` |
| What I had to fix to make the clone work | nothing in the repo; my L11 tests live in `hw11`, not in `l11`, so `.env` sets `L11_DIR=../hw11` (an environment setting, not in git) |

### Run cost in two currencies

| Currency | Value | Source |
| --- | --- | --- |
| Machine: minutes | 4.4 (262.0 s) | `elapsed` in the record |
| Machine: money | $0.0284 | `machine.cost_usd` in the record |
| Human: minutes on "what changed" | by hand a few minutes (not timed) · with reporting about 2 seconds | section 4 |

---

## 7. ADR

| # | Boundary | Decision (one sentence) | File | Status |
| --- | --- | --- | --- | --- |
| ADR-001 | dataset ↔ runner | `load_cases()` gives the runner the cases and a content `set_hash`; the runner never reads set files | `adr/ADR-001-dataset-runner.md` | accepted |
| ADR-002 | runner ↔ assertions | the runner collects all oracles into one dictionary; the assertion only reads it | `adr/ADR-002-runner-assertions-oracle-dictionary.md` | accepted |
| ADR-003 | assertions ↔ engines | assertions use the engine answer recorded in the case, not a live engine call | `adr/ADR-003-assertions-engines.md` | accepted |

**Check of the three decision lines:**

| ADR | What we traded (minus in Consequences) | What we do not do (Alternatives) | When we will revisit |
| --- | --- | --- | --- |
| ADR-001 | yes | yes | yes |
| ADR-002 | yes | yes | yes |
| ADR-003 | yes | yes | yes |

### Assistant perimeter (a separate line, not an ADR)

| The assistant may change | The assistant does not change |
| --- | --- |
| `run_eval.py` (the five step functions), `eval-framework.md`, `.env` settings, and `paypilot-stand/.env` (provider and model), always after my explicit OK | `sets/`, `base.v*.md` / `compose.v4.yml`, `reports/` content, and git history (I run all commands and commits myself) |

---

## 8. AI use declaration and cascade readiness

**AI declaration:** An AI assistant (Claude) helped me filled the template from my run outputs, made the temporary hash-check edits in `sets/l03.jsonl` (restored with `git checkout`), and drafted the three ADRs, which I reviewed and accepted with changes. All Docker runs, commits and pushes were done by me. 

**Cascade readiness (chain-ready):**

| Check on a clean clone | Passes |
| --- | --- |
| `docker compose run --rm eval` finishes without manual steps | yes (clean clone `l12-clean`, record `run-20261120T150022Z.json`) |
| A record with all six version fields appears in `reports/` | yes |
| Non-zero exit code when a check is red | yes (exit 1, 5 red cases) |
| L11 unit tests run with the same command (`unit_tests` block: `manual.exit_code` and `generated.exit_code` are `0`) | yes (manual 6 passed, generated 27 passed, both exit 0) |
| All 20 step tests are green | yes (20 passed) |

**Attachments:** `run_eval.py` · `reports/run-20261120T144625Z.json` (v1) · `reports/run-20261120T145022Z.json` (v4) · `reports/run-20261120T145305Z.json` (clean, `SET=l12`) · `reports/run-20261120T145310Z.json` (D26) · `reports/run-20261120T150022Z.json` (clean clone) · `adr/`
