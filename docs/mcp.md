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
- **`propose_exemption`** — the agent's verdict on a finding, one of three, chosen by
  whether the finding is false, being fixed, or true-but-managed:
  - **`confirmed`** — it's real and the agent will fix it. Nothing is staged; the
    deduction stands until the code changes.
  - **`refuted`** — it's *false*: the rule mis-measures your repo (say, a mandatory
    `any` at a JSON boundary scored as an escape hatch). Must carry a `path:line`
    citation proving it. GetAX re-verifies the citation and stages a verified-claim
    exemption that **restores the points once you commit it.**
  - **`managed`** — it's *true, but under a documented remediation policy* (a grandfather
    clause, a shrink-only ratchet). Must cite the policy. GetAX records it; **no points
    are restored** — the debt is real — but it's marked *tracked*, not pretended away.

  Refute only what's false; *manage* what's true-but-policied — don't refute real debt
  just because a grandfather clause exists. An unverifiable citation is rejected and the
  finding stands.

## The loop

Ask your agent: *"use getax to check this repo's agent-readiness and work the findings."*
What happens:

1. The agent calls **`get_findings`** and reads them with the context you don't have to
   give it — it already has the tree.
2. For a **real** finding, it fixes the code and you re-run to watch the score rise.
3. For a **false** one — a rule mis-measuring your repo, like a flat registry counted as a
   monolith — it calls **`propose_exemption`** with verdict `refuted` and a citation that
   *proves it's false*. GetAX verifies the citation and stages the exemption.
4. For a finding that's **true but under a policy** — a monolith on a shrink-only ratchet,
   a documented grandfather clause — it uses verdict `managed` and cites the policy. The
   debt is recorded and stays visible; **no points come back**, because the debt is real.
   This is the honest middle: you don't pretend it away, and you don't over-promise a fix.
5. You **review the staged entries in the diff and commit them.** Each carries its reason
   and citation, so a teammate (or a future you) can see exactly why the number is what it
   is — and a refutation only moves the score once it's committed.

That committed `.getax/` is the repo's memory: a newcomer who clones it inherits the
settled adjudications, and GetAX stops re-raising what's already been answered.

## Why it can't be gamed

The one rule underneath all of it: **config supplies verified claims, never accepted
values.** Your agent can say *"exclude `i18n/` from file-size, they're data catalogs"* — a
claim GetAX re-checks against the actual files. It can never just *write a higher number*.
And an adjudication only touches the score of record once it's committed and reviewed;
anything an agent quiets locally stays local and can't move the shared score. The judgment
gets all the context it needs, and none of the power to lie about it.

## Two things to know

- **Reconnect after adding it mid-session.** If you `claude mcp add getax` while an agent
  session is already running, the tools aren't live until the client reconnects. Add it,
  then start (or restart) the session.
- **`get_readiness` scores the build in your hands.** It runs the local scan, so the
  number reflects your working tree and the CLI version you have installed — which can
  differ from a floor committed by a teammate on a newer build. If a number surprises you,
  check `getax version` against what set the baseline.
