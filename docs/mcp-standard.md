# The MCP Server Standard

**Version:** 0.1 (Draft)
**Status:** Development Specification
**Last updated:** 2026-08-19
**Companion documents:** `auto-research-standard.md`, `skill-standard.md`

---

## 1. Introduction

### 1.1 Purpose

The Model Context Protocol (MCP) is how an AI agent reaches live
capability: external APIs, databases, services, and local tools,
exposed through a uniform interface. Where a **skill** carries
*knowledge* (when and how to act — see `skill-standard.md`), an **MCP
server** provides the *means* (the tool calls themselves).

The quality of an MCP server is measured by one thing: **how well it
enables LLMs to accomplish real-world tasks.** A server that wraps an
API endpoint-for-endpoint with cryptic names, dumps unfiltered payloads
into the context window, and fails with "Internal error" is worse than
no server — the agent wastes its budget fighting the interface. This
standard defines the contract a conforming MCP server must meet:
discoverable tools, context-economic responses, actionable errors,
honest annotations, and empirical evaluation.

### 1.2 Scope

In scope:

- Tool design: naming, schemas, descriptions, annotations (§4).
- Response and error contracts (§5).
- Transport, security, and implementation requirements (§6–§7).
- Testing and agent-facing evaluation (§8).

Out of scope:

- The MCP wire protocol itself — defined by the official specification
  at `modelcontextprotocol.io`. This standard *profiles* it: it selects
  recommended options and adds quality requirements, and defers to the
  official spec on everything else.
- Skills and prompts that *consume* MCP tools (see `skill-standard.md`).

### 1.3 Audience

- **Server developers** (human or agent) — §3–§8 are normative.
- **Reviewers** — §9 is the checklist.

### 1.4 Conformance language

**MUST**, **MUST NOT**, **SHOULD**, and **MAY** are interpreted as in
RFC 2119.

---

## 2. Protocol Baseline

This standard assumes the official MCP specification. The load-bearing
facts:

- MCP communication uses **JSON-RPC 2.0**. A session begins with an
  `initialize` handshake in which client and server negotiate protocol
  version and capabilities.
- Servers expose three primitive kinds:
  - **Tools** — model-invoked functions with input schemas (the focus of
    this standard).
  - **Resources** — application/data content exposed for reading.
  - **Prompts** — reusable prompt templates.
- **Transports**: `stdio` for local servers (a subprocess of the client)
  and **Streamable HTTP** for remote servers.
- Tools MAY declare behavioral **annotations** (`readOnlyHint`,
  `destructiveHint`, `idempotentHint`, `openWorldHint`) that clients use
  for display and consent decisions.
- Tool results carry content items (text, structured data); modern SDKs
  support an `outputSchema` plus `structuredContent` for typed returns.

When this standard and the official spec disagree, the official spec
wins; file an issue against this document.

---

## 3. Design Principles

- **M1 — Design for the agent, not the API.** The consumer is a model
  with a finite context window making tool-choice decisions from names
  and descriptions alone. Every design question reduces to: does this
  help the agent pick the right tool, call it correctly, and use the
  result without drowning in tokens?
- **M2 — Coverage before convenience.** Prefer comprehensive API
  coverage over a few hand-tuned workflow tools; agents compose basic
  tools in ways authors don't foresee. Add workflow tools *on top* where
  a task is common and multi-step — never instead of the underlying
  operations. (Some clients execute code against basic tools; others
  need workflows. Coverage serves both.)
- **M3 — Context economy.** Tool descriptions are read at decision time;
  results are read in full. Both are budget. Return focused, relevant
  data; support filtering and pagination; never dump raw upstream
  payloads when a projection will do.
- **M4 — Errors are instructions.** An error message is the agent's
  *next prompt*. "Request failed" wastes a turn; "404: repo
  `owner/name` not found — check `name` spelling or call
  `github_list_repos` to enumerate" recovers it.
- **M5 — Honest metadata.** Names, descriptions, schemas, and
  annotations are the contract the agent plans against. An optimistic
  `readOnlyHint: true` on a mutating tool is a safety bug, not a
  documentation bug.

---

## 4. Tool Contract

### 4.1 Naming

- Tool names MUST be action-oriented and consistently prefixed by
  service or domain: `github_create_issue`, `github_list_repos`,
  `sentry_get_event`.
- Names MUST be stable: renaming a tool breaks every skill and workflow
  that references it. Deprecate, don't rename; when a rename is
  unavoidable, keep the old name as an alias for one major version.

### 4.2 Input schemas

- Every tool MUST declare a typed input schema (Zod in TypeScript,
  Pydantic in Python, or JSON Schema directly).
- Every parameter MUST carry a description; non-obvious parameters
  SHOULD include examples in the description ("ISO-8601 date, e.g.
  `2026-08-19`").
- Schemas SHOULD encode constraints (enums, ranges, formats, defaults)
  so the agent cannot construct invalid calls — validation at the schema
  layer is cheaper than rejection at runtime.
- IDs and handles SHOULD be discoverable through companion tools
  (`*_list_*`, `*_search_*`) so the agent never has to guess them.

### 4.3 Tool descriptions

Each tool description MUST state, concisely:

1. What the tool does (one line).
2. When to use it — and, where ambiguity exists, when to use a sibling
   tool instead.
3. Anything surprising about parameters or returns.

Descriptions are decision-time context (M1, M3): complete but tight.
Implementation trivia does not belong here.

### 4.4 Annotations

Declare truthfully (M5):

| Annotation | Meaning | Set `true` only when |
|---|---|---|
| `readOnlyHint` | The tool does not modify its environment | No side effects whatsoever |
| `destructiveHint` | The tool may irreversibly destroy data | Deletion/overwrite is possible |
| `idempotentHint` | Repeated identical calls have no extra effect | Safe to retry blindly |
| `openWorldHint` | The tool interacts with open external content | Results depend on the live world (web, search) |

Absent annotations default to cautious assumptions; false ones are
conformance violations.

### 4.5 Output schemas

- Tools returning structured data SHOULD declare `outputSchema` and
  return `structuredContent` alongside a human-readable text rendering.
- Return *projections* of upstream data, not raw dumps (M3): the fields
  the agent needs, consistently named, documented in the schema.

---

## 5. Response and Error Contract

### 5.1 Responses

- **Format**: return structured data (JSON) for anything the agent will
  filter, compare, or re-emit; markdown only for content meant for human
  display. Never return both meanings under one ambiguous blob.
- **Pagination**: any tool that can return unbounded collections MUST
  paginate (cursor or offset) and SHOULD support caller-side filtering
  (query, date range, status). Document the pagination mechanism in the
  tool description.
- **Size discipline**: prefer concise summaries with a follow-up
  `*_get_*` tool for full detail over always-full payloads.

### 5.2 Errors (M4)

Every error response MUST:

1. Say *what* failed, in domain terms the agent can act on (not just an
   upstream HTTP status).
2. Say *why* when knowable (invalid parameter value, missing permission,
   rate limit, upstream outage).
3. Suggest the *next step* — the parameter to fix, the companion tool to
   call, the precondition to satisfy.

Transient failures (rate limits, timeouts) SHOULD be distinguishable
from permanent ones so the agent knows whether retrying is sensible.
Authentication failures MUST NOT echo credentials or tokens.

---

## 6. Transport and Deployment

- **Local servers** (single-user, same machine): `stdio`. Simple,
  no network surface.
- **Remote servers** (shared, hosted): **Streamable HTTP**, preferring
  **stateless JSON** request/response over stateful sessions and
  streaming — simpler to scale, restart, and reason about. Reach for
  statefulness only when a feature genuinely requires it.
- A server SHOULD be deployable behind standard HTTP infrastructure
  (TLS termination, auth proxies) without protocol-level changes.

---

## 7. Security Requirements

- **Least privilege**: the server's upstream credentials SHOULD be
  scoped to exactly the operations its tools expose.
- **Secrets**: never in tool descriptions, schemas, responses, or logs;
  never echoed in errors (§5.2). Configuration via environment or secret
  stores, never committed.
- **Input validation**: validate at the schema layer first, then again
  at trust boundaries (path traversal, injection into upstream queries,
  SSRF via caller-supplied URLs).
- **Destructive operations**: MUST carry `destructiveHint: true`; SHOULD
  require explicit identifiers (not wildcards) and SHOULD offer dry-run
  or confirm parameters where the upstream API allows.
- **Remote servers** MUST authenticate callers; an unauthenticated
  mutating MCP endpoint on a network is an incident.
- Servers MUST NOT execute caller-supplied arbitrary code unless that is
  the tool's documented, sandboxed purpose.

---

## 8. Implementation Standards

### 8.1 Recommended stack

- **TypeScript** with the official MCP SDK is the default: strong SDK
  support, static typing, and good linting. **Python** (FastMCP /
  official SDK) is fully acceptable, especially where the domain's
  libraries live in Python.
- Async/await for all I/O; a tool call MUST NOT block the server event
  loop on network or subprocess work.

### 8.2 Project structure (normative expectations)

A conforming server has:

- **Shared infrastructure** used by every tool: one API client (with
  auth), one error-mapping helper (§5.2), one response formatter, one
  pagination helper. No per-tool reimplementation (DRY).
- **Full type coverage** on tool inputs and outputs.
- **Build verification**: TypeScript servers MUST compile clean
  (`npm run build`); Python servers MUST pass `py_compile` and import
  cleanly.

### 8.3 Manual verification

Before release, exercise every tool with the **MCP Inspector**
(`npx @modelcontextprotocol/inspector`): list tools, inspect schemas,
make success and failure calls, and confirm errors are actionable.

---

## 9. Agent-Facing Evaluation

Compiling and passing unit tests does not prove an agent can *use* the
server. A conforming server ships an evaluation set (M1 as an empirical
claim):

1. **Generate ~10 evaluation questions** against a live instance:
   - **Independent** — no question depends on another's side effects.
   - **Read-only** — answerable with non-destructive calls only.
   - **Complex** — each requires multiple tool calls and exploration,
     not a single lookup.
   - **Realistic** — tasks a real user would care about.
   - **Verifiable** — a single clear answer, checkable by string
     comparison.
   - **Stable** — the answer won't drift over time.
2. **Verify every answer yourself** by solving the question with the
   server before shipping it.
3. Store as XML:

```xml
<evaluation>
  <qa_pair>
    <question>Which open issue, opened after 2026-06-01, has the most
      comments, and who is its assignee?</question>
    <answer>octocat</answer>
  </qa_pair>
  <!-- ~9 more -->
</evaluation>
```

4. Run the evaluation with an agent client against the server; a
   conforming server lets a capable agent answer the large majority
   without human hints. Failures are design feedback: fix names,
   descriptions, projections, or errors — not the questions.

---

## 10. Conformance Checklist

- [ ] Tool names action-oriented, consistently prefixed, stable.
- [ ] Every tool: typed input schema with per-parameter descriptions;
      constraints encoded; IDs discoverable via companion tools.
- [ ] Every tool description: what / when / surprises — concise.
- [ ] Annotations declared and *true* (M5).
- [ ] Structured tools return `outputSchema` + `structuredContent`;
      responses are projections, not dumps; unbounded collections
      paginate and filter.
- [ ] Errors say what/why/next-step; transient vs permanent
      distinguishable; no secrets echoed.
- [ ] Transport per §6; remote servers authenticated; least-privilege
      credentials; boundary validation; destructive tools marked and
      guarded.
- [ ] Shared client/error/format/pagination infrastructure; DRY; full
      type coverage; clean build.
- [ ] MCP Inspector pass over every tool (success + failure paths).
- [ ] ~10-question evaluation set, self-verified, shipped, and run
      against an agent client with results recorded.

---

## Appendix A — Minimal Tool Definition (TypeScript sketch)

```ts
server.registerTool(
  "github_list_issues",
  {
    description:
      "List issues for a repository. Use github_get_issue for full " +
      "details of one issue. Supports state/label filtering and " +
      "cursor pagination (max 50 per page).",
    inputSchema: {
      owner: z.string().describe("Repository owner, e.g. 'modelcontextprotocol'"),
      repo: z.string().describe("Repository name, e.g. 'typescript-sdk'"),
      state: z.enum(["open", "closed", "all"]).default("open"),
      labels: z.array(z.string()).optional()
        .describe("Return only issues carrying ALL of these labels"),
      cursor: z.string().optional()
        .describe("Pagination cursor from a previous response's next_cursor"),
    },
    outputSchema: {
      issues: z.array(z.object({
        number: z.number(), title: z.string(),
        state: z.string(), labels: z.array(z.string()),
      })),
      next_cursor: z.string().nullable(),
    },
    annotations: { readOnlyHint: true, openWorldHint: true },
  },
  async ({ owner, repo, state, labels, cursor }) => {
    // ... call upstream, project fields, map errors per §5.2
  },
);
```

---

## Appendix B — Relationship to the Companion Standards

- **Skill Standard** — skills tell the agent *when and how* to use
  tools; they reference MCP tools by their (stable) names and depend on
  the contracts here: if a server renames tools or lies in annotations,
  every skill built on it degrades silently.
- **Auto-Research Standard** — long-tail capabilities of the research
  pipeline (paper search, reference download, sandboxed scoring) are
  natural MCP servers; the pipeline's hard rules (a validator runs
  outside the attempt's reach) map onto this standard's transport and
  security requirements — e.g. a scoring MCP server reachable from
  candidate code would violate both standards at once.
