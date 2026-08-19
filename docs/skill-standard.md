# The Agent Skill Standard

**Version:** 0.1 (Draft)
**Status:** Authoring Specification
**Last updated:** 2026-08-19
**Companion documents:** `auto-research-standard.md`, `mcp-standard.md`

---

## 1. Introduction

### 1.1 Purpose

A **skill** is a self-contained directory of instructions, examples, and
reference material that extends an AI agent's capabilities — specialized
knowledge, workflow patterns, tool integrations, or templates. Skills
are how a methodology (such as the Auto-Research Standard) becomes
operational: the pipeline stages, gates, and protocols are packaged as
skills the agent loads and follows.

Skills are cheap to write and easy to write badly. A bad skill is worse
than none: it triggers at the wrong time, burns context with prose the
agent already knows, or — worst — describes a workflow in its metadata
so the agent follows the summary instead of the skill. This standard
exists so that every skill in the ecosystem is discoverable, token-lean,
tested, and honest about what it does.

### 1.2 Scope

In scope:

- The anatomy and file contract of a conforming skill (§4–§5).
- Authoring rules: descriptions, naming, token budgets, style (§6).
- The skill lifecycle: drafting, testing, evaluation, iteration (§7).
- Anti-patterns and the conformance checklist (§8–§9).

Out of scope:

- **MCP servers**, which provide *tools* (live capability), not
  *knowledge* (when and how to act). See `mcp-standard.md`. A skill may
  document how to use an MCP server's tools; the two compose.
- Host-specific configuration of any particular agent runtime, beyond
  the common contract defined here.

### 1.3 Audience

- **Skill authors** (human or agent) — §1–§8 are normative for you.
- **Reviewers and auditors** — §9 is your checklist.

### 1.4 Conformance language

**MUST**, **MUST NOT**, **SHOULD**, and **MAY** are interpreted as in
RFC 2119.

---

## 2. Terminology

| Term | Definition |
|---|---|
| **skill** | A directory containing a `SKILL.md` plus optional bundled resources. |
| **SKILL.md** | The skill's entry file: YAML frontmatter + markdown body. |
| **frontmatter** | The YAML block at the top of SKILL.md carrying `name` and `description`. The only part of the skill that is *always* in the agent's context. |
| **description** | The frontmatter field that is the skill's *only* triggering mechanism. It answers one question: "Should I read this skill right now?" |
| **bundled resources** | Optional `scripts/` (executable code), `references/` (docs loaded on demand), `assets/` (files used in output). |
| **progressive disclosure** | The three-level loading system (metadata → body → resources) that keeps irrelevant content out of context (§3, D1). |
| **triggering** | The agent's decision, made from the description alone, to load the skill body. |
| **scope** | Where a skill is installed: Project, User, Extra, or Built-in (§4.3). |
| **CSO** | Claude/agent Search Optimization — writing names, descriptions, and keywords so future agents *find* the skill (§6.3). |
| **eval** | A test prompt with expected-output criteria, used to verify the skill changes agent behavior for the better. |
| **baseline run** | The same eval prompt executed *without* the skill; the control group. |

---

## 3. Design Principles

- **D1 — Progressive disclosure.** Skills load in three levels:
  1. *Metadata* (name + description) — always in context, ~100 words.
  2. *SKILL.md body* — loaded when the skill triggers; keep under ~500
     lines, and under ~500 words for frequently-loaded skills.
  3. *Bundled resources* — loaded on demand; unlimited in size, and
     scripts can be *executed* without being read into context.

  Content MUST live at the lowest level that can afford it. Detail that
  sits in the body but is only occasionally needed belongs in
  `references/`; deterministic, repetitive logic belongs in `scripts/`.

- **D2 — The description is a trigger, not a summary.** The description
  MUST describe *when to use* the skill — symptoms, situations, user
  phrasings — and MUST NOT summarize the skill's process or workflow.
  Measured failure mode: when a description summarizes the workflow, the
  agent follows the *description* and skips the skill body. The
  description is a signpost; the body is the map.

- **D3 — Token efficiency.** Every token in a frequently-loaded skill is
  paid in every conversation. Prefer references over repetition, one
  excellent example over five mediocre ones, and `--help` output over
  inlined flag documentation.

- **D4 — Explain the why.** Instructions that state *reasons* outperform
  bare imperatives; capable agents generalize from rationale and route
  around edge cases the author didn't foresee. Heavy-handed ALWAYS/NEVER
  blocks without justification are a yellow flag — reserve them for
  genuinely non-negotiable rules (safety, audit trails).

- **D5 — No surprise.** A skill MUST NOT contain anything its
  description wouldn't lead the user to expect: no malware, no exploit
  code, no hidden network calls, no behavior beyond its stated intent.

- **D6 — Skills are proven, not plausible.** A skill encodes a technique
  that *worked*, verified by testing with the agent (§7). "If you didn't
  watch an agent fail without the skill, you don't know if the skill
  teaches the right thing."

---

## 4. Structure Contract

### 4.1 Anatomy

```
skill-name/
├── SKILL.md              # required — frontmatter + instructions
├── scripts/              # optional — executable code for deterministic tasks
├── references/           # optional — docs loaded into context as needed
└── assets/               # optional — files used in output (templates, fonts, icons)
```

Only `SKILL.md` is required. Add a resource directory only when content
genuinely needs it (D1); a single-file skill is a feature, not a gap.

### 4.2 Frontmatter (normative)

Two fields are required; the whole block SHOULD stay under 1024
characters.

```yaml
---
name: verb-first-name
description: Use when <triggering conditions, symptoms, contexts>...
---
```

- **`name`** — letters, numbers, and hyphens only; identical to the
  directory name. Prefer active, verb-first or gerund forms that name
  what the agent *does* (`condition-based-waiting`,
  `creating-skills`), not noun phrases (`skill-creation`,
  `async-test-helpers`).
- **`description`** — third person; starts with "Use when..." (or an
  equivalent trigger statement); states concrete triggering conditions
  (symptoms, error messages, user phrasings, adjacent situations);
  SHOULD stay under 500 characters; MUST NOT summarize the workflow
  (D2). Include enough of *what* the skill covers to disambiguate it
  from neighbors, but the process belongs in the body.

Optional fields (e.g. `license`, `compatibility`, `metadata`) MAY be
added per the host's spec; this standard neither requires nor forbids
them.

**Description examples:**

```yaml
# BAD — summarizes the workflow; agents may follow this instead of the body
description: Use when executing plans - dispatches subagent per task with code review between tasks

# BAD — vague, no triggering conditions
description: For async testing

# GOOD — triggering conditions only
description: Use when executing implementation plans with independent tasks in the current session

# GOOD — symptoms and situations, technology-agnostic
description: Use when tests have race conditions, timing dependencies, or pass/fail inconsistently
```

### 4.3 Scopes and precedence

Skills are installed at one of four scopes. When scopes define the same
skill name, the more specific scope wins:

**Project > User > Extra > Built-in**

- **Project** — skills shipped with a repository, for everyone working
  on it.
- **User** — personal skills (`~/.agents/skills/`, `~/.claude/skills/`,
  or the host equivalent).
- **Extra / Built-in** — provided by toolchains and the host runtime.

A skill MUST NOT silently depend on a more specific scope's copy of
another skill; cross-references name the skill, not a scope path (§6.4).

### 4.4 SKILL.md body (recommended skeleton)

No fixed section order is mandated, but a conforming body SHOULD cover,
in roughly this order:

```markdown
# <Skill Name>

## Overview            # what this is; the core principle in 1–2 sentences
## When to use         # symptoms and use cases; when NOT to use
## <Core content>      # procedure, pattern, or reference material
## Quick reference     # scannable table/bullets (for larger skills)
## Common mistakes     # what goes wrong + fixes (where applicable)
```

Rules:

- Keep the body under ~500 lines; approaching the limit means a layer of
  `references/` is missing, with clear pointers from the body on *when*
  to read each reference.
- Reference files over ~300 lines SHOULD carry their own table of
  contents.
- When a skill spans multiple domains/frameworks, split by variant
  (`references/aws.md`, `references/gcp.md`, …) so the agent loads only
  the relevant one.

---

## 5. Content Rules

### 5.1 What belongs in a skill

Create a skill for:

- Techniques that weren't intuitively obvious and will recur.
- Patterns that apply broadly across projects.
- Reference material the agent would otherwise have to rediscover.

Do **not** create a skill for:

- One-off solutions and narratives of a single past success.
- Standard practices well-documented elsewhere.
- Project-specific conventions (those belong in the project's
  `AGENTS.md` / `CLAUDE.md`).
- Mechanical constraints that a regex, linter, or CI check can enforce —
  automate those; save documentation for judgment calls.

### 5.2 Writing style

- Prefer the imperative form ("Run X", "Reject candidates that…").
- Explain *why* a step matters (D4); one sentence of rationale beats a
  paragraph of prohibition.
- Describe the *problem* in technology-agnostic terms unless the skill
  is genuinely technology-specific — then say so explicitly in the
  description.
- One excellent, complete, runnable, commented example beats many
  partial ones. Do not port examples across 5 languages; do not ship
  fill-in-the-blank templates posing as examples.
- Flowcharts only for non-obvious decision points and loops where the
  agent might stop too early — never for reference material (use tables)
  or linear instructions (use numbered lists).

### 5.3 Discoverability (CSO)

Agents find skills by matching the description (and searchable body
keywords) against the situation at hand. To be found:

- Cover the vocabulary a struggling agent would use: error messages
  ("Hook timed out", "ENOTEMPTY"), symptoms ("flaky", "hanging"),
  synonyms ("timeout/hang/freeze"), tool and file names.
- Put searchable terms early in the description.
- Name the skill after what the agent *does* or the core insight
  (verb-first/gerund), not the category it belongs to.

### 5.4 Cross-referencing other skills

Reference another skill by name with an explicit requirement marker:

```markdown
✅ **REQUIRED BACKGROUND:** Understand <other-skill> before this procedure.
✅ Use <other-skill> for the sub-workflow.
❌ @path/to/other-skill/SKILL.md        # force-loads; burns context unconditionally
❌ "see skills/testing/..."             # unclear whether required
```

`@`-style force-loading MUST NOT be used for cross-skill references —
it defeats progressive disclosure (D1).

### 5.5 Token budgets (SHOULD)

| Skill type | Budget |
|---|---|
| Getting-started / always-loaded workflows | < 150 words each |
| Frequently-loaded skills | < 200 words total |
| Typical skills | < 500 words in the body; details to `references/` |

Verify with `wc -w path/to/SKILL.md`. Budgets are targets, not gates —
exceeding one requires a reason, not permission.

---

## 6. Bundled Resources

### 6.1 `scripts/`

Executable code for deterministic or repetitive tasks. A script is
justified when test runs show agents independently rewriting the same
helper — write it once, bundle it, and point to it from SKILL.md.
Scripts SHOULD support `--help` rather than having every flag documented
in the body.

### 6.2 `references/`

Documentation loaded into context only when the body directs the agent
to read it. Organize by domain/variant when the skill is broad. Each
reference SHOULD state, in the body's pointer, the conditions under
which it is needed.

### 6.3 `assets/`

Files used *in output* (templates, icons, fonts) — copied or rendered,
not read as instructions.

---

## 7. Lifecycle: Draft → Test → Iterate

Writing skills is test-driven development applied to process
documentation. A skill that has never been watched changing an agent's
behavior is an unverified hypothesis.

### 7.1 Capture intent

Before drafting, establish: what the skill enables, when it should
trigger (user phrasings, contexts), the expected output format, and
whether the output is objectively verifiable (file transforms, code
generation, fixed workflows — yes; writing style, art — evaluate
qualitatively instead). Extract answers from the current conversation
first when the skill captures a workflow that just happened.

### 7.2 Draft

Write SKILL.md per §4–§6, addressing the *specific* failures the skill
exists to prevent — not hypothetical ones.

### 7.3 Test with the agent (RED → GREEN)

1. **Baseline (RED).** For 2–3 realistic test prompts — the kind a real
   user would type, with concrete detail — run the agent *without* the
   skill. Record what it does and, for discipline-enforcing skills, the
   rationalizations it produces. Save test prompts (e.g.
   `evals/evals.json`) with expected-output descriptions.
2. **With-skill (GREEN).** Run the same prompts *with* the skill, in the
   same round as the baselines. Compare.
3. **Iterate (REFACTOR).** Improve from feedback: generalize instead of
   overfitting to test prompts; cut content that isn't pulling its
   weight; convert repeatedly-rewritten helpers into `scripts/`;
   close loopholes agents found. Rerun the full eval set each
   iteration, keeping the baseline fixed.

Skills with objectively verifiable outputs SHOULD carry programmatic
assertions; subjective outputs (prose, design) are reviewed
qualitatively — do not force assertions onto judgment calls.

### 7.4 Description optimization

Because the description alone drives triggering, verify it empirically
before shipping:

1. Write ~20 trigger-eval queries — realistic, detailed, mixed length,
   including typos and casual phrasing: 8–10 **should-trigger** covering
   varied phrasings and near-neighbor intents; 8–10
   **should-not-trigger** that are *near-misses* (shared keywords, wrong
   task), not obviously irrelevant prompts.
2. Have the user review and amend the query set.
3. Measure trigger rates of the draft description against the set;
   revise the description and re-measure. Prefer held-out queries for
   the final selection to avoid overfitting the wording.

### 7.5 Ship

A skill is shippable when: its evals pass against a fixed baseline, the
description holds up on the trigger-eval set, and §9 checks green.
Deploy one skill at a time — no untested batches.

---

## 8. Anti-Patterns

| Anti-pattern | Why it fails |
|---|---|
| **Workflow in the description** | Agents follow the summary and skip the body (D2). |
| **Narrative examples** ("In session 2025-10-03 we found…") | Too specific to transfer; a skill is a reusable technique, not a war story. |
| **Multi-language dilution** | Five mediocre ports of one example; maintenance burden, no added insight. |
| **Force-loaded cross-references** (`@…`) | Burns context unconditionally; defeats D1. |
| **Inlined `--help` documentation** | Flags drift; the binary's own help is canonical (D3). |
| **Generic flowchart labels** (`step1`, `helper2`) | No semantic content; cannot be scanned. |
| **Unjustified ALWAYS/NEVER walls** | Without rationale, agents route around them at the worst moment (D4). |
| **Skills for mechanical constraints** | What CI can enforce, documentation cannot (§5.1). |

---

## 9. Conformance Checklist

- [ ] Directory contains `SKILL.md`; resource directories exist only if
      used.
- [ ] `name` is hyphenated, verb-first where possible, and equals the
      directory name.
- [ ] `description` is third-person, starts with a trigger ("Use
      when…"), names concrete symptoms/contexts, contains no workflow
      summary, ≤ ~500 chars.
- [ ] Body under ~500 lines with pointers to `references/` for anything
      deeper; large references have tables of contents.
- [ ] Token budget respected for the skill's loading frequency (§5.5).
- [ ] Rationale given for non-obvious instructions; ALWAYS/NEVER
      reserved for non-negotiables.
- [ ] One excellent example rather than many partial ones; no
      narratives.
- [ ] Cross-skill references by name with requirement markers; no
      `@` force-loads.
- [ ] Evals exist: baseline vs with-skill runs recorded; assertions for
      verifiable outputs.
- [ ] Trigger-eval set (~20 queries incl. near-miss negatives) reviewed
      by the user; description verified against it.
- [ ] Nothing in the skill would surprise a user who read only the
      description (D5).

---

## Appendix A — Relationship to the Companion Standards

- **Auto-Research Standard** — the methodology: stages, gates,
  validators, artifact contracts.
- **Skill Standard (this document)** — the packaging: how that
  methodology (or any technique) is written into skills an agent will
  actually find, load, and follow.
- **MCP Standard** — the capability layer: how live tools and external
  services are exposed to the agent. Skills tell the agent *when and
  how*; MCP servers give it the *means*.

A mature setup composes all three: the auto-research pipeline is shipped
as conforming skills; long-tail external capabilities (paper search,
reference download, scoring sandboxes) are exposed as conforming MCP
servers; skills reference those tools by name.
