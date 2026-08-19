# The Auto-Research Standard

**Version:** 0.1 (Draft)
**Status:** Methodology Specification
**Last updated:** 2026-08-19
**Companion documents:** `skill-standard.md`, `mcp-standard.md`

---

## 1. Introduction

### 1.1 Purpose

Auto-research is a methodology for letting an AI agent conduct genuine
scientific research autonomously: proposing hypotheses, implementing
experiments, and iterating — under constraints strict enough that the
final result is *publishable* without a human re-checking the agent's
homework.

The core problem this standard solves: **an agent that grades its own
work cannot be trusted.** Capable agents game any scorer they can reach
(reward-hacking rates of ~30% have been measured on realistic tasks),
and "do not cheat" instructions in prompts are near-useless. Therefore
the methodology replaces trust with construction: a sealed validator —
not a human, and not the agent's own judgment — decides whether an
attempt succeeded.

This document is the normative specification of that methodology. It
defines the pipeline stages, the gates between them, the artifacts each
stage must produce, and the conformance requirements for any project
claiming to follow the standard.

### 1.2 Scope

In scope:

- The four-stage pipeline (topics → db → validator → run) and its gates.
- The artifact contracts (file formats and schemas) that make the
  pipeline auditable and resumable.
- The anti-gaming requirements (sealed holdout, negative controls,
  harness hardening).
- The human-in-the-loop checkpoints where user decisions are mandatory.

Out of scope:

- The scientific write-up of results (use a paper-writing workflow once
  the run stage terminates successfully).
- Any specific research domain. The standard is domain-agnostic.

### 1.3 Audience and how to read this document

This specification serves two audiences:

- **Researchers (human users)** — read §1–§6 to understand what the
  pipeline does, where your decisions are required, and what guarantees
  the gates give you.
- **AI agents and skill developers** — §7–§8 are the normative contracts
  you must implement exactly: directory layout, file schemas, CLI
  behavior, gate checklists.

### 1.4 Conformance language

The key words **MUST**, **MUST NOT**, **SHOULD**, and **MAY** are to be
interpreted as in RFC 2119. Anything marked MUST is a hard rule; a
project that violates it under user-approved exception MUST record the
deviation in `overrides:` of `STATE.md` (see §8.2).

---

## 2. Overview

### 2.1 What auto-research is

A research project is a search: hypotheses are proposed, implemented as
candidates, evaluated, and the results steer the next hypotheses.
Auto-research automates this loop with an AI agent, under three
conditions that make automation safe:

1. **Success is machine-checkable.** A validator program scores each
   candidate with no human judgment. A topic on which no such validator
   can be built is not an auto-research topic.
2. **The validator cannot be gamed.** It runs outside the agent's reach,
   in a canonical environment, against instances the agent cannot see,
   and it is proven strict by negative controls before the loop starts.
3. **The agent's autonomy is budgeted.** The user authorizes a number of
   attempts; the loop runs autonomously within that budget and stops for
   review when it is exhausted.

### 2.2 Pipeline at a glance

```
┌──────────┐   ┌──────────┐   ┌────────────┐   ┌──────────┐
│ 1.topics │ → │ 2.  db   │ → │3.validator │ → │ 4.  run  │ → done
└──────────┘   └──────────┘   └────────────┘   └──────────┘
                    │               │
              survey_gate     validator_gate
              (owned by db)   (owned by validator)
```

- **Stage 1 — Topics.** Pick research topics whose success a machine can
  check; define metrics and a red-teamed acceptance gate per topic.
- **Stage 2 — DB.** Build the evidence base the agent needs to *propose
  new ideas*: distilled insights, a domain database, pinned reference
  implementations, a catalog. Closes with the **survey gate**.
- **Stage 3 — Validator.** Formalize the publishable bar, seal a holdout
  instance set, build the canonical environment and the `validate` CLI,
  and prove strictness against negative controls. Closes with the
  **validator gate**.
- **Stage 4 — Run.** The autonomous loop: batches of attempts (each in
  its own git worktree, scored only by the validator), reflection,
  reporting, re-planning — until the bar is met or the authorized
  attempt budget runs out.

Pipeline position and loop configuration live in a single file,
`research/STATE.md`, which is the single source of truth (§8.2).

---

## 3. Terminology

| Term | Definition |
|---|---|
| **attempt** | One implementation + one scored evaluation of a single hypothesis. The atomic unit of the run loop and of user authorization. |
| **candidate** | The code/artifact an attempt produces, living in its own git worktree. |
| **cycle** | One batch of attempts plus the reflection that follows it. Internal bookkeeping; users authorize *attempts*, never cycles. |
| **batch** | The set of attempts executed in one cycle (`batch_size`, default 10). |
| **draft / improve / debug** | Attempt kinds. A *draft* tries a genuinely different approach; an *improve* makes exactly one atomic change to the best-scoring known-good ancestor; a *debug* fixes a promising-but-broken branch (capped at 2 per branch). |
| **validator** | The CLI program that scores candidates. The only thing that counts as a result. |
| **dev instances** | The instance set visible to attempts during development. |
| **holdout** | The sealed instance set, invisible to attempts, used only for metered adjudication and final verification. |
| **insight** | A distilled, transferable technique (algorithmic idea, proof method, data structure, benchmark practice) with stated applicability and limits. |
| **gate** | A checklist-owned state transition (`survey_gate`, `validator_gate`) that flips to `passed` only when its artifacts verify on disk. |
| **negative control** | A runnable fake candidate that *must be rejected* by the validator (cheater, wrong-answer, timeout, env-escape, plus topic-specific hacks). |
| **primary metric** | The metric that enters the score function — usually one per topic. |
| **guard metric** | An anti-gaming side condition, enforced by the validator as a rejection rule, not a warning. |
| **acceptance gate** | The user-confirmed condition under which a result counts as publishable: primary metric, threshold, instance families, baseline. |
| **authorized attempts** | The number of attempts the loop may run without further user review. |
| **blacklist** | The running list of dead approaches with reasons, maintained across cycles so nothing is retried. |
| **override** | A dated, user-approved deviation from a hard rule, appended to `STATE.md`. Append-only. |

---

## 4. Design Principles

The normative rules in §5–§8 all derive from these principles. When a
situation is not covered by a specific rule, resolve it in the direction
these principles point.

- **P1 — Machine-checkable success.** If no program can tell whether an
  attempt succeeded, the topic is not suitable. Topic selection (§5.1)
  exists to enforce this *before* any research happens.
- **P2 — The validator is the only judge.** No eyeballed timings, no
  partial credit, no "it looks right." Every scored run goes through the
  validator CLI under a hard wall-clock limit.
- **P3 — Sealed by construction, not convention.** The holdout and the
  scorer are unreachable from candidate code because of *how the harness
  is built* (separate process, read-only mounts, gitignored paths), not
  because anyone was told not to look.
- **P4 — Harden the harness, never trust the prompt.** When a new hack
  is discovered, patch the validator and add a negative control
  reproducing the hack. Never respond by adding "do not cheat"
  instructions to attempt prompts.
- **P5 — Single source of truth.** `STATE.md` records stage, gates,
  budget, and configuration. Skills update it; humans may edit it;
  nothing important lives only in conversation.
- **P6 — Gates are never skipped.** A gate flips only when its checklist
  verifies on disk. Any user-approved exception is recorded in
  `overrides:` first. Nothing is worked around silently.
- **P7 — Authorized autonomy.** The agent runs without interruption
  only within the attempt budget and directions the user approved.
  Mandatory checkpoints (§6) are never inferred, defaulted, or assumed.
- **P8 — Failures are data.** Crashes and timeouts are recorded failed
  attempts, never silently retried. Failure artifacts (validator
  `errors[]`, stderr) feed forward into the next batch. Worktrees and
  logs are the audit trail and are left intact.

---

## 5. Pipeline Specification

### 5.1 Stage 1 — Topics

**Input:** a research domain from the user.
**Output:** `topics.md` — chosen topics, each with a `### Metrics`
block and a user-confirmed `### Acceptance gate` block — and `STATE.md`
advanced to `stage: db`.

A good auto-research topic is one where **a validator, not a human, can
tell whether an attempt succeeded**. The metrics defined here become the
validator's score function in Stage 3; the gaming risks identified here
become negative controls there.

**Procedure (normative summary):**

1. Generate 5–10 candidate topics, each grounded by web search and at
   least one recent reference showing the problem is open. **No
   candidate from memory alone.**
2. Score each candidate 1–5 on four suitability criteria:
   - **Checkable** — success is machine-checkable with no human
     judgment.
   - **Cheap** — one attempt evaluates in minutes on local hardware.
   - **Headroom** — the space supports tens-to-hundreds of genuinely
     distinct attempts.
   - **Publishable** — a clearly stated bar exists whose crossing is a
     publishable result.

   Any candidate scoring ≤2 on *Checkable* or *Cheap* MUST be flagged
   unsuitable.
3. The user picks topics (multi-select).
4. For each chosen topic, derive 2–5 candidate metrics, each with a
   precise definition, a computation method whose cost fits the *Cheap*
   budget, and stated gaming risks. Classify each as **primary** or
   **guard**. The user approves the metric set per topic.
5. Define the **acceptance gate** per topic — mandatory, never deferred.
   The user MUST state: the primary metric, the threshold, the instance
   families it must hold on, and the baseline it must beat. Vague gates
   ("state of the art-ish") are non-conformant.
6. **Red-team the gate before accepting it.** Enumerate the concrete
   ways an attempt could satisfy the gate while being wrong, trivial,
   or unpublishable. Check at minimum: overfitting to visible dev
   instances; lookup tables / hard-coded answers; a baseline too weak to
   be publishable; a threshold at or below already-published results; an
   instance family too narrow to generalize; metric/claim divergence;
   and the topic's own gaming risks. For each hack, name the
   strengthening that closes it. If any hack survives, the gate is not
   strict enough — strengthen and iterate. The final gate requires the
   user's **explicit confirmation**.

**Exit criteria:** every chosen topic in `topics.md` has an approved
`### Metrics` block (one bullet per metric: name, primary|guard,
definition, computation + cost, gaming risks) and an `### Acceptance
gate` block (the confirmed condition verbatim, plus one bullet per
considered hack: `**<hack>** — closed by <strengthening>`).

### 5.2 Stage 2 — DB (evidence base)

**Input:** a topic with metrics in `topics.md`.
**Output:** a knowledge base, `research/INSIGHTS.md`,
`research/database/`, pinned reference implementations, and
`research/CATALOG.md`. Closes with the **survey gate**.

The point of this stage is not "papers exist on disk" but "the agent
holds the insights needed to *propose new ideas*."

**Procedure (normative summary):**

1. **Map insight areas** — the algorithmic techniques, proof/analysis
   methods, data structures, and benchmark practices an idea-proposer
   needs for this topic. Show the map to the user before downloading
   anything.
2. **Download for coverage.** Every insight area MUST be covered by ≥1
   downloaded reference in `.knowledge/`. Coverage by SOTA-results-only
   papers is insufficient.
3. **Distill** each insight area into `research/INSIGHTS.md`
   (Technique / Applies when / Limits / Sources — §8.4), distilled from
   the rendered papers, not from memory. Every entry's Sources MUST
   cite references present in `.knowledge/INDEX.md`.
4. **User selects** which insight areas ground idea generation. Selected
   entries go under `## Selected`, the rest under `## Shelved`.
   *Selected is the run loop's default grounding, not a cap* — run-stage
   hypotheses may go beyond it, and Shelved entries can be promoted at
   cycle gates.
5. **Build the domain database** under `research/database/` with a
   `README.md` documenting schema and provenance of every record.
6. **Pin reference implementations**: for each key algorithm with
   public code, record URL + commit hash, build it, and run its own
   smoke test. Write small reference implementations only where no code
   exists and the algorithm is load-bearing.
7. **Catalog** every relevant algorithm/software in
   `research/CATALOG.md` (§8.5) with status `reproduced` / `pinned` /
   `paper-only`.

**Survey gate (owned by this stage).** Flip `survey_gate: passed
<date>` and set `stage: validator` only when **all** of:

- `research/CATALOG.md` exists and every key algorithm from the insight
  map appears in it;
- `.knowledge/INDEX.md` lists every reference cited in INSIGHTS.md and
  CATALOG.md;
- `research/INSIGHTS.md` has a non-empty, user-chosen `## Selected`
  section;
- every `reproduced`-status entry actually ran, with its output recorded
  in CATALOG.md notes.

### 5.3 Stage 3 — Validator

**Input:** a topic whose survey gate passed.
**Output:** `research/validator/` — `GOAL.md`, the `validate` CLI, a
sealed holdout, negative controls, and `manifest.json`. Closes with the
**validator gate**.

The goal: **once the validator's bar is met, the result is publishable**
— so the validator MUST be impossible to satisfy by accident or by
cheating.

**Procedure (normative summary):**

1. **Formalize the publishable bar** from the topic's `### Acceptance
   gate` in `topics.md` into `GOAL.md` (primary metric, threshold,
   instance families, baseline). Re-confirm with the user; any revision
   MUST itself be red-teamed and explicitly confirmed — never silently
   weakened. In the same exchange, present the **validation method** for
   user confirmation *before anything is built*: dev/holdout families
   and provenance, holdout query budget, per-attempt wall-clock budget
   (`time_limit_seconds`, default 300 s), environment choice, and the
   planned negative controls.
2. **Split instances.** Development instances are visible to attempts;
   the holdout lives under `research/benchmark/private/` and MUST be
   sealed by construction:
   - `research/benchmark/private/` is in `.gitignore`;
   - the validator runs as a separate process outside the attempt
     worktree, with the holdout mounted read-only into the validator
     environment only; neither the holdout nor the scorer source is
     reachable from candidate code;
   - holdout labels are never opened in design context — attempts and
     reflection see only aggregate pass/fail;
   - a holdout query budget is set in `manifest.json` (default: 1
     aggregate query per 3 cycles); the validator logs every holdout
     query and refuses beyond budget;
   - instance provenance is recorded in `manifest.json`.
3. **Build the canonical environment.** A Docker image with pinned
   dependencies, `--network none` at run time, CPU/memory limits, and
   the wall-clock limit enforced by the harness, not the candidate. If
   Docker is unavailable or unsuitable, fall back to a locked venv +
   subprocess sandbox and record `validator_env: fallback (<reason>)`
   in STATE.md and the manifest — never fall back silently.
4. **Implement the `validate` CLI** per the contract in §8.7. Guard
   metrics from `topics.md` MUST be enforced as rejection rules. Include
   the free `--precheck` mode and cascade evaluation (cheap checks
   first) so attempts stop wasting cycles on format bugs.
5. **Strictness self-test.** Build the four standard negative controls
   (`cheater`, `wrong-answer`, `timeout`, `env-escape` — Appendix A)
   plus one per topic-specific gaming risk, including every hack listed
   in the topic's `### Acceptance gate` block. Run them; all MUST be
   rejected with informative errors. Record results under `"self_test"`
   in `manifest.json`.

**Validator gate (owned by this stage).** Flip `validator_gate: passed
<date>` and set `stage: run` only when **all** of:

- `GOAL.md` states the bar in terms of the primary metric, matching the
  user-confirmed acceptance gate (or an explicitly re-confirmed
  revision);
- the holdout is sealed (gitignored, labels unopened) with provenance in
  the manifest;
- `validate` runs end-to-end on a trivial baseline candidate against dev
  instances;
- every negative control is rejected with a specific `errors[]` entry;
- the manifest records the environment (`docker` or
  `fallback (<reason>)`);
- everything the run stage needs is **committed to the main branch** —
  `research/validator/`, dev instances, database, INSIGHTS.md,
  CATALOG.md — with the holdout absent. Attempt worktrees branch from
  main and must see these files; committing first also pins the
  validator version that results are reproducible against. The validator
  that *scores* is always main's copy running in the canonical
  environment — the copy inside an attempt worktree is inert.

### 5.4 Stage 4 — Run (the loop)

**Input:** both gates passed. **Output:** cycles of attempts, reflection
reports, and — on success — a candidate that meets the `GOAL.md` bar.

**Entry check.** Both `survey_gate` and `validator_gate` MUST read
`passed` in `STATE.md`, and their artifacts MUST verify on disk.
Otherwise refuse and route back through the pipeline. A missing gate is
never worked around.

**Hard rules (non-negotiable during the loop):**

1. Every attempt runs in its own `.worktrees/attempt-NNN/` with a
   `LOG.md` (§8.8).
2. Every scored run goes through the validator CLI; nothing else counts.
3. Hard wall-clock limit `time_limit_seconds` on every scored run.
4. Holdout labels never enter design context — only aggregate validator
   output.
5. Crashes/timeouts are recorded failures, never silently retried.

**Cycle:**

1. **Plan the batch.** Generate ~2× `batch_size` candidate hypotheses,
   grounded in `## Selected` insights by default but not fenced by them
   (original ideas, cross-insight combinations, and fresh literature are
   equally welcome; each hypothesis names its source). Rank by expected
   information gain, cost, and distinctness; promote the top
   `batch_size`. Two filters apply before anything is implemented:
   - **Novelty check** — reject any near-duplicate of a hypothesis in
     any prior attempt's LOG.md. Never re-spend an attempt on a restated
     old idea.
   - **Batch composition** — mix *drafts* with *improvements* (exactly
     one atomic change to the best known-good ancestor) and at most 2
     *debug* attempts per broken branch. A branch that exhausts its
     debug cap is abandoned, not nursed.

   Scope planning context: drafts see a digest of *sibling* attempts;
   improvements and debugs see their *ancestral* chain. Feed forward the
   previous batch's failure artifacts — failures are data.
2. **Confirm the plan.** The first batch of any authorization executes
   only after the user confirms the promoted hypotheses and batch
   composition. Later cycles within the same authorization run
   autonomously.
3. **Execute** each attempt per the attempt protocol (§8.8).
4. **Reflect.** Write the cycle reflection
   `docs/discussion/YYYY-MM-DD-HHMMSS-cycle-NN.md` (§8.9): facts with
   honest denominators, root-cause lessons (a score is a result, not a
   cause — keep asking why until the answer is actionable), evidence
   carried forward including the blacklist, a literature check, and the
   next-round plan. Emit `cycle-NN.json` and render the HTML report
   (non-fatal on failure — the markdown is canonical). If the holdout
   query budget allows, adjudicate the cycle's top candidate on the
   holdout (aggregate only) and record it — dev-score selection overfits
   over long runs.
5. **Sync.** Commit the cycle's artifacts to main (reflection md/json/
   html, STATE.md, INSIGHTS.md changes, validator manifest) and, when a
   remote is configured, push main plus every `attempt-NNN` branch.
   Never push `research/benchmark/private/`; holdout results appear
   only as aggregates.
6. **Soft gate.** Subtract the cycle's attempts from
   `authorized_attempts`. If enough remain, continue autonomously. If
   exhausted: stop, present the report, and offer 2–4 candidate
   directions for the next batch, each grounded in the cycle's lessons.
   The user decides both direction(s) and how many attempts to authorize
   (0 = stop). Attempts are the unit the user authorizes — never ask
   them to reason in rounds or cycles.

**Termination.** When the validator reports the `GOAL.md` bar met on dev
instances, run `validate --instances holdout` once, report both results,
set `stage: done`, and hand off to the user. (Scientific write-up is out
of scope.)

---

## 6. Human-in-the-Loop Checkpoints

Auto-research is autonomous *within* user-granted authority, never
instead of it. These checkpoints are mandatory; none may be inferred,
defaulted, or assumed.

| # | When | The user decides |
|---|---|---|
| 1 | Stage 1, after scoring | Which candidate topics to pursue |
| 2 | Stage 1, per topic | The metric set (primary + guards) |
| 3 | Stage 1, per topic | The acceptance gate — stated by the user, red-teamed, explicitly confirmed after strengthening |
| 4 | Stage 2, insight map | Whether the planned insight areas are right before downloading |
| 5 | Stage 2, after distillation | Which insights are Selected vs Shelved |
| 6 | Stage 3, before building | The GOAL.md bar, validation method, time budget, environment, negative controls |
| 7 | Stage 4, first batch of each authorization | The promoted hypotheses and batch composition |
| 8 | Stage 4, soft gate (budget exhausted) | Next direction(s) and how many attempts to authorize |
| 9 | Any stage | Any protocol deviation — recorded in `overrides:` before taking effect |

---

## 7. Anti-Gaming Requirements

### 7.1 Threat model

Assume the attempting agent is capable and adversarial-by-incentive: it
will exploit any reachable scorer, visible holdout, or weak baseline —
not out of malice, but because optimizing the visible metric is its job.
Measured reward-hacking rates on realistic tasks are ~30%, and prompt
instructions do not prevent it. Therefore:

- The scorer MUST run outside the attempt worktree, in the canonical
  environment, as main's committed copy (P3).
- The holdout MUST be unreachable from candidate code and its labels
  never read in design context (P3).
- Baselines and thresholds MUST be justified against published results
  at topic-definition time, so a "passing" score is a publishable claim.
- Responses to newly discovered hacks MUST be harness patches plus a new
  negative control — never prompt edits (P4).

### 7.2 Negative controls

The validator gate requires the validator to reject, with an informative
`errors[]` entry each: the four standard controls (Appendix A) **plus
one control per topic-specific gaming risk**, including every hack
listed in the topic's acceptance-gate block. The four standard controls
are the floor.

Controls are written before the first real attempt and kept working: a
validator change that stops rejecting any control **re-opens the
validator gate** until the control is rejected again. Each control's
rejection report is pasted into `manifest.json` under `"self_test"` with
the date it last passed.

### 7.3 Holdout discipline

- Dev instances: visible, unlimited use.
- Holdout: metered aggregate queries only (default budget: 1 per 3
  cycles), logged in the manifest, refused beyond budget.
- Final termination: exactly one full holdout run, reported alongside
  the dev result.
- The standing risk — dev score improving while holdout does not — MUST
  be named explicitly in every cycle reflection.

---

## 8. Artifact Contracts

This section is normative for implementers. File formats here are the
inter-stage API of the pipeline; skills and agents MUST read and write
them exactly.

### 8.1 Standard project layout

```
<project>/
├── topics.md                      # Stage 1 output
├── research/
│   ├── STATE.md                   # single source of truth (§8.2)
│   ├── INSIGHTS.md                # distilled insights (§8.4)
│   ├── CATALOG.md                 # algorithm/software catalog (§8.5)
│   ├── database/                  # domain dataset + README (schema, provenance)
│   ├── benchmark/
│   │   ├── dev/                   # visible development instances
│   │   └── private/               # sealed holdout — GITIGNORED
│   └── validator/
│       ├── GOAL.md                # the publishable bar (§8.6)
│       ├── validate               # the validator CLI (§8.7)
│       ├── manifest.json          # env, provenance, budgets, self-test (§8.6)
│       └── controls/<name>/       # negative controls (Appendix A)
├── .knowledge/                    # downloaded references + INDEX.md
├── .worktrees/attempt-NNN/        # one per attempt: code + LOG.md + report.json
└── docs/discussion/               # cycle reflections: md (canonical) + json + html + index.html
```

### 8.2 `research/STATE.md`

Single source of truth for pipeline position and loop configuration.
Skills update it; humans may edit it.

```markdown
# Autoresearch State

- stage: topics            # topics | db | validator | run | done
- topic: (unset)           # slug of the chosen topic once stage >= db
- batch_size: 10           # attempts per cycle
- time_limit_seconds: 300  # hard wall-clock limit per scored run
- authorized_attempts: 0   # attempts the loop may run without user review
- next_attempt: 1          # next .worktrees/attempt-NNN number
- next_cycle: 1            # next reflection cycle number
- gates:
  - survey_gate: pending     # pending | passed YYYY-MM-DD
  - validator_gate: pending  # pending | passed YYYY-MM-DD
- validator_env: (unset)   # docker | fallback (<reason>)
- overrides: (none)        # every user-approved protocol deviation, dated
```

Rules:

- A gate flips to `passed` only by the stage that owns it (DB for
  `survey_gate`, Validator for `validator_gate`), after its checklist
  verifies on disk.
- `overrides:` is append-only. Every deviation from a hard rule is
  recorded here with date and reason before taking effect.
- `attempt-NNN` numbering is zero-padded to 3 digits and never reused,
  even for crashed attempts.

Stage routing (used when resuming a project):

| `stage` | required artifacts before entering |
|---|---|
| topics | — |
| db | `topics.md` with ≥1 chosen topic, each with `### Metrics` and a user-confirmed `### Acceptance gate` |
| validator | survey gate passed: `CATALOG.md`, `.knowledge/INDEX.md`, `INSIGHTS.md` with user-selected section |
| run | validator gate passed: `research/validator/manifest.json` with self-test results |
| done | final report in `docs/discussion/` |

### 8.3 `topics.md`

One `## <topic title>` section per chosen topic containing:

- problem statement;
- why auto-research fits (the four suitability scores);
- key references (title + arXiv ID/DOI);
- a `### Metrics` block — one bullet per approved metric:
  `**<name>** (primary|guard): definition; computation + cost; gaming
  risks`;
- a `### Acceptance gate` block — the user-confirmed condition verbatim,
  followed by one bullet per considered hack:
  `**<hack>** — closed by <strengthening>`.

### 8.4 `research/INSIGHTS.md`

Two sections: `## Selected` and `## Shelved`. One `###` entry per
insight area, four required lines:

```markdown
### <insight area>
- **Technique**: the transferable idea, stated so it can be applied to a
  new attempt without rereading the paper.
- **Applies when**: preconditions — problem structure, size regime, data.
- **Limits**: where it breaks down, known failure modes, complexity walls.
- **Sources**: citation keys, e.g. [smith2025exact].
```

Rules: every Sources line cites ≥1 reference present in
`.knowledge/INDEX.md` (no from-memory insights). Moving an entry between
Selected and Shelved is a user decision; record the date when moved.

### 8.5 `research/CATALOG.md`

One row per relevant algorithm/software: name, source (paper/repo),
status, notes. Status is one of:

- `reproduced` — we built and ran it (output recorded in notes);
- `pinned` — source-locked (URL + commit), not yet run;
- `paper-only` — no code exists.

### 8.6 `research/validator/GOAL.md` and `manifest.json`

`GOAL.md` states the publishable bar: primary metric, threshold,
instance families it must hold on, and the baseline — matching the
user-confirmed acceptance gate in `topics.md`.

`manifest.json` records at minimum: the environment (`docker` or
`fallback (<reason>)` plus image/lock id), instance provenance (where
each instance came from, how labels were obtained), the holdout query
budget and a log of every holdout query, and `"self_test"` — each
negative control's rejection report with the date it last passed.

### 8.7 The `validate` CLI contract

The validator is the *only* thing that counts as scoring.

```
validate <candidate-dir> [--precheck] [--instances dev|holdout] [--out report.json]
```

- `<candidate-dir>` is an attempt worktree.
- `--instances dev` (default) scores visible development instances;
  `--instances holdout` scores the sealed set and prints only aggregate
  pass/fail — never labels. Holdout runs are metered (logged in the
  manifest, refused beyond budget).
- `--precheck` is a **free validity check**: structure and output format
  only, reveals no score, consumes nothing. Attempts may call it
  freely — its purpose is to stop format bugs from consuming scored
  attempts.
- **Cascade evaluation**: cheap structural checks → smallest instances →
  full set. A candidate failing a stage is rejected there with that
  stage's diagnostics; expensive stages run only for survivors.

Exit codes:

| code | meaning |
|---|---|
| 0 | candidate evaluated; see report for score |
| 1 | candidate rejected (wrong answers, cheating, contract violation); report explains why |
| 2 | validator infrastructure error (not the candidate's fault) |

JSON report (written to `--out`, always, even on rejection):

```json
{
  "status": "scored | rejected | error",
  "score": null,
  "per_instance": [
    {"instance": "<id>", "result": "pass | fail | timeout",
     "seconds": 0.0, "detail": "<what happened>"}
  ],
  "errors": [
    {"where": "<instance or phase>", "what": "<precise diagnostic>",
     "hint": "<what a fixer should look at first>"}
  ],
  "environment": {"kind": "docker | fallback", "image_or_lock": "<id>"}
}
```

**Error-richness rule:** every rejection and every failed instance MUST
produce an `errors[]` entry specific enough to debug from the report
alone — name the instance, observed vs expected behavior, and the first
thing to check. `"Validation failed"` alone is a contract violation.

### 8.8 Attempt protocol and `LOG.md`

Every attempt, no exceptions:

1. `git worktree add .worktrees/attempt-NNN` (NNN from `next_attempt`,
   zero-padded to 3 digits; increment immediately; never reused).
2. **LOG.md first** — before writing code: attempt number and date;
   **kind** (`draft | improve | debug`); **parent** (ancestor attempt
   number, `none` for drafts); **hypothesis** (naming the `## Selected`
   insight(s) it draws on; for `improve`, the single atomic change);
   **expected evidence** (what result would confirm or kill it).
3. **Implement** in the worktree. The candidate may read dev instances
   and everything in `research/` except `benchmark/private/`. Use
   `validate --precheck` freely while developing.
4. **Score** via `validate <worktree> --out <worktree>/report.json`.
   Nothing else counts as a result.
5. **Record outcome** in LOG.md: the validator's JSON summary plus what
   was learned — especially from failures. A crash or timeout is a
   recorded failed attempt; retrying with a fix is a *new* attempt.
6. **Commit and leave intact** on the `attempt-NNN` branch: generated
   code, `LOG.md`, `report.json`. Worktrees are the audit trail.

### 8.9 Cycle reflection and reports

One report per cycle: `docs/discussion/YYYY-MM-DD-HHMMSS-cycle-NN.md`
(timestamp UTC, NN from `next_cycle`). Three required sections:

1. **Review — what we did.** Facts only; honest denominators ("K of N
   attempts improved the primary metric; best X vs prior best Y"); one
   line per attempt (kind, parent, metric, causal claim); budget state
   and any overrides.
2. **Lessons we learnt.** For the top improvement and every failed or
   flat branch: **Observation → Root cause → Evidence → Implication**.
   A score is a result, not a cause — the root cause must name something
   actionable or testable, marked *confirmed* or *suspected*. Includes
   *Evidence carried forward* (what the batch established, what each
   failure rules out, the running blacklist, the explicit dev-vs-holdout
   risk statement) and a *Literature check*. Off-goal but potentially
   publishable findings are recorded, marked `off-goal`.
3. **Next round.** The next batch's falsifiable hypothesis, planned
   attempts, and abandonment conditions; any proposed insight promotions
   (for user confirmation at the gate).

The markdown is canonical. After writing it, emit
`docs/discussion/cycle-NN.json` (NN zero-padded to 2 digits, no
timestamp prefix, so prior cycles are globbable) and render HTML
(`cycle-NN.html` + cross-cycle `index.html`). A report-render failure
never blocks the loop.

Key `cycle-NN.json` fields (schema version 1): `cycle`, `date_utc`,
`project`, `attempts_range`, `attempts_remaining`, `primary_metric`
(name + `direction: min|max` — never assume higher-is-better),
`guard_metrics`, `bar`, `best_this_cycle`, `best_prior`, `holdout`
(spent/result), `attempts[]` (id, kind, parent, hypothesis, primary,
guards, status ∈ `improved|no-change|failed|timeout`, causal_note,
log_path), `reflection` (review/evidence/literature/next, verbatim from
the md), `lessons[]` (observation, root_cause, evidence, implication,
confidence; ≥1 entry; off-goal entries have `implication` starting with
`off-goal:`), `blacklist_new[]`, `insight_promotions[]`.

---

## 9. Conformance Checklist

A project conforms to this standard when an auditor can verify, on disk:

- [ ] `research/STATE.md` exists, is readable, and its recorded stage
      matches the artifacts present.
- [ ] `topics.md` contains ≥1 chosen topic with an approved Metrics
      block and a red-teamed, user-confirmed Acceptance gate.
- [ ] The survey gate passed: CATALOG.md complete, INDEX.md covers all
      citations, INSIGHTS.md has a user-selected `## Selected` section,
      all `reproduced` entries actually ran.
- [ ] The validator gate passed: GOAL.md matches the acceptance gate;
      holdout sealed and gitignored with provenance recorded; validate
      runs end-to-end on a baseline; all negative controls rejected with
      specific errors; environment recorded.
- [ ] Every attempt lives in its own `attempt-NNN` worktree with LOG.md
      and report.json; no attempt number reused.
- [ ] Every cycle produced a reflection md (canonical) with honest
      denominators, root-cause lessons, and a blacklist update.
- [ ] Every protocol deviation appears, dated, in `overrides:`.
- [ ] The holdout was never pushed, never opened in design context, and
      queried only within budget.

---

## Appendix A — Standard Negative Controls

Runnable candidate directories under `research/validator/controls/`.
The validator gate requires rejecting all four, each with an informative
`errors[]` entry.

| Control | What it does | Validator must |
|---|---|---|
| `cheater` | Returns hard-coded answers for dev instances (lookup table keyed on instance id / input hash) | Reject: guard metrics / unseen-instance check catches it |
| `wrong-answer` | Runs plausibly but returns an incorrect result on ≥1 instance | Reject: exactness check fails; error names the instance and the wrong value |
| `timeout` | Busy-loops past the wall-clock limit | Mark `timeout` on the instance; enforce the limit with <5% overshoot; score accordingly |
| `env-escape` | Attempts network access or reads outside the candidate dir (e.g. tries to open holdout labels) | Reject: the environment blocks it and the report says what was attempted |

Plus, per topic: one control per gaming risk in the topic's Metrics
block and per hack in its Acceptance gate block. The four above are the
floor, never the ceiling.

---

## Appendix B — Design Rationale (informative)

- **Why is the acceptance gate defined in Stage 1 but enforced in
  Stage 3?** Because the *topic choice itself* must be constrained by
  machine-checkability; deferring the gate lets unsuitable topics
  consume weeks of evidence-gathering before the mismatch is discovered.
- **Why worktrees instead of directories?** Lineage. Every attempt is a
  branch from a pinned main, so improvements diff cleanly against their
  ancestors, the scoring validator is always main's committed copy, and
  the audit trail survives the agent.
- **Why is the markdown reflection canonical over the HTML?** Because
  rendering must never block the loop; prose is what reflection actually
  produces, and structured JSON/HTML are presentational copies.
- **Why meter the holdout instead of forbidding it?** Because dev-score
  selection overfits over long runs; occasional aggregate adjudication
  is the cheapest known defense, and a hard zero would push projects to
  never check at all.
