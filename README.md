# oh-my-auto-research

Standards for letting AI agents conduct real research — autonomously,
verifiably, and without trusting them to grade their own homework.

This repository hosts three companion specification documents:

| Document | Layer | What it standardizes |
|---|---|---|
| [The Auto-Research Standard](docs/auto-research-standard.md) | Methodology | The four-stage pipeline (topics → db → validator → run), its gates, artifact contracts, and anti-gaming requirements for agent-driven research |
| [The Agent Skill Standard](docs/skill-standard.md) | Packaging | How skills (`SKILL.md` + bundled resources) are authored, discovered, tested, and shipped |
| [The MCP Server Standard](docs/mcp-standard.md) | Capability | How MCP servers expose tools to agents: tool contracts, errors, security, and agent-facing evaluation |

## The one-paragraph version

An agent that grades its own work cannot be trusted. Auto-research
therefore replaces trust with construction: a sealed validator — built
and proven strict *before* the research loop starts — decides whether
each attempt succeeded, against instances the agent can never see. The
methodology is shipped as conforming **skills** (the knowledge of when
and how to act), and long-tail capabilities are exposed as conforming
**MCP servers** (the means to act). The three documents specify each
layer so the whole stack is auditable on disk.

## Status

All documents are **v0.1 drafts**. They are normative within this
project (RFC 2119 MUST/SHOULD/MAY) and evolve through review.

## Repository conventions

The ecosystem is organized into four repo types, named by prefix:

| Prefix | Repo type | Naming rule | Example |
|---|---|---|---|
| *(none)* | Standards & meta | descriptive name | `oh-my-auto-research` (this repo) |
| `auto-` | Research project | `auto-<topic-slug>` — a short abbreviation of the research content, 2–4 hyphenated words | `auto-qec-code-distance`, `auto-moe-routing` |
| `skill-` | Skill infra | `skill-<skill-or-family-name>` — matches the skill name(s) inside | `skill-autoresearch` |
| `mcp-` | MCP infra | `mcp-<service-name>` — matches the server's tool-name prefix | `mcp-paper-search` |

General rules: all lowercase; hyphens, no underscores; the slug must be
short but recognizable — abbreviate the *research question*, not the
methodology (the `auto-` prefix already says that).

### Research project repo (`auto-*`)

One repo per auto-research project, conforming to the Auto-Research
Standard (§8.1). Basic skeleton:

```
auto-<topic-slug>/
├── README.md                    # topic, current stage, link to reports
├── .gitignore                   # MUST contain: research/benchmark/private/
│                                #                .worktrees/
├── topics.md                    # chosen topics + metrics + acceptance gates
├── research/
│   ├── STATE.md                 # single source of truth (stage, gates, budget)
│   ├── INSIGHTS.md              # Selected / Shelved insights
│   ├── CATALOG.md               # algorithms & software, reproduced|pinned|paper-only
│   ├── database/                # domain dataset + README (schema, provenance)
│   ├── benchmark/
│   │   ├── dev/                 # visible development instances
│   │   └── private/             # sealed holdout — gitignored, never pushed
│   └── validator/
│       ├── GOAL.md              # the publishable bar
│       ├── validate             # the validator CLI
│       ├── manifest.json        # env, provenance, budgets, self-test results
│       └── controls/            # negative controls (cheater, wrong-answer, ...)
├── .knowledge/                  # downloaded references + INDEX.md
├── .worktrees/                  # attempt-NNN worktrees (local; branches pushed)
└── docs/discussion/             # cycle reflections: md + json + html + index.html
```

### Skill infra repo (`skill-*`)

Ships one skill or a related family of skills, each conforming to the
Skill Standard. Install by symlinking or copying into the agent's skills
directory (`~/.agents/skills/`, `~/.claude/skills/`, …).

```
skill-<name>/
├── README.md                    # what the skill(s) do, install instructions
├── skills/
│   ├── <skill-name>/
│   │   ├── SKILL.md             # frontmatter (name, description) + body
│   │   ├── references/          # optional, loaded on demand
│   │   ├── scripts/             # optional, executable helpers
│   │   └── assets/              # optional, files used in output
│   └── <another-skill>/         # families live together (e.g. autoresearch-*)
└── evals/                       # trigger-eval sets + test prompts (Skill Std §7)
```

### MCP infra repo (`mcp-*`)

Ships one MCP server conforming to the MCP Standard. The repo slug
SHOULD match the server's tool-name prefix (`mcp-paper-search` serves
`paper_search_*` tools).

```
mcp-<service>/
├── README.md                    # tools overview, install & client config
├── src/                         # TypeScript (default) — or mcp_<service>/ in Python
├── tests/                       # unit + MCP Inspector-driven checks
├── evals/
│   └── evaluation.xml           # ~10 agent-facing QA pairs (MCP Std §9)
├── package.json                 # or pyproject.toml
└── ...                          # language-standard project files
```

### How they compose

An `auto-*` project repo is driven by skills installed from `skill-*`
repos and calls tools served by `mcp-*` repos; all three answer to the
standards in this repo. Example: `auto-qec-code-distance` runs the
pipeline from `skill-autoresearch` and scores attempts through a
validator sandboxed behind `mcp-validator`.

## This repository's layout

```
├── README.md
└── docs/
    ├── auto-research-standard.md   # methodology: pipeline, gates, contracts
    ├── skill-standard.md           # packaging: authoring & testing skills
    └── mcp-standard.md             # capability: building MCP servers
```
