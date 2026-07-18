---
layout: doc
title: GetAX in your agent's toolbelt (MCP)
permalink: /docs/mcp/
description: Run GetAX as an MCP server your coding agent connects to. It hands the agent your findings; the agent, which already has your whole codebase in context, adjudicates them. Nothing leaves your machine.
---

# GetAX in your agent's toolbelt

The [AX score](/docs/getting-started/) tells you *where* a repository is not ready for an
agent. The obvious next question is *"is each of these findings actually real, here, in my
repo?"* — and that is a judgment call. It needs the whole codebase in context.

You already have something with the whole codebase in context: **the coding agent you're
running** — Claude Code, Cursor, whatever's open in this repo. So GetAX hands *it* the
findings. `getax mcp` runs GetAX as an [MCP](https://modelcontextprotocol.io) server your
agent connects to; the agent reads the findings, decides with full context which are real,
fixes those, and refutes the rest **with a citation GetAX re-verifies.**

Two things stay true, and they're the point:

- **Nothing leaves your machine.** This is a local process — the agent, the repo, and
  GetAX, all on your box. No model of ours, no endpoint, no egress. The intelligence was
  already here; GetAX just gives it something to work on.
- **The score stays deterministic.** The agent *advises* and *proposes*; it never sets the
  number. An adjudication only moves the score once it's a **verified claim** GetAX
  re-checked and you committed — reviewed like any code change.

## Wire it up

`getax mcp` is a stdio MCP server. Point your agent at it. For most clients that's one
line — e.g. Claude Code:

```bash
claude mcp add getax -- npx getax mcp
```

Or the equivalent JSON in any MCP-capable client:

```json
{
  "mcpServers": {
    "getax": { "command": "npx", "args": ["getax", "mcp"] }
  }
}
```

Run it from the repository you want scored — the server works on the current directory.

## The three tools

- **`get_readiness`** — the current AX score and its five signals. Deterministic.
- **`get_findings`** — the ranked, attributed gaps a scan found, each a *candidate* to
  adjudicate: an id, the signal, the claim, and the points at stake.
- **`propose_exemption`** — the agent's verdict on a finding. `confirmed` means it's real
  and the agent will fix it. `refuted` means it's a false finding — and **must** carry a
  `path:line` citation in your repo that justifies it. GetAX re-verifies that the citation
  resolves and only then stages a verified-claim exemption into `.getax/`. An unverifiable
  citation is rejected and the finding stands.

## The loop

Ask your agent: *"use getax to check this repo's agent-readiness and work the findings."*
What happens:

1. The agent calls **`get_findings`** and reads them with the context you don't have to
   give it — it already has the tree.
2. For a **real** finding, it fixes the code and you re-run to watch the score rise.
3. For a **false** one — a large file that's a flat registry, a rule your docs already
   grandfather, a policy stated in an ADR — it calls **`propose_exemption`** with a
   citation. GetAX verifies the citation and stages the exemption in `.getax/`.
4. You **review the staged exemption in the diff and commit it.** It carries its reason and
   its citation, so a teammate (or a future you) can see exactly why the number is what it
   is.

That committed `.getax/` is the repo's memory: a newcomer who clones it inherits the
settled adjudications, and GetAX stops re-raising what's already been answered.

## Why it can't be gamed

The one rule underneath all of it: **config supplies verified claims, never accepted
values.** Your agent can say *"exclude `i18n/` from file-size, they're data catalogs"* — a
claim GetAX re-checks against the actual files. It can never just *write a higher number*.
And an adjudication only touches the score of record once it's committed and reviewed;
anything an agent quiets locally stays local and can't move the shared score. The judgment
gets all the context it needs, and none of the power to lie about it.
