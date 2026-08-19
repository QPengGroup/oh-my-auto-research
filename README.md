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

## Repository layout

```
├── README.md
└── docs/
    ├── auto-research-standard.md   # methodology: pipeline, gates, contracts
    ├── skill-standard.md           # packaging: authoring & testing skills
    └── mcp-standard.md             # capability: building MCP servers
```
