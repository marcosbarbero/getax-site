---
layout: doc
title: Getting started
permalink: /docs/getting-started/
description: Install GetAX, score a repository, and read the result — the five-minute path from curious to your first score.
---

# Getting started

Five minutes from curious to your first score. The next guide —
[Wire GetAX into your workflow](/docs/wiring/) — turns that one-off scan into a gate
that blocks regressions on every push, which is where GetAX earns its keep.

## 1. Get a licence

GetAX is a licensed product — [request a demo](https://getax.app) and you'll be issued
a key. From your dashboard, **Settings → Generate CLI licence key** gives you a one-line
install command. Keep the key secret; it identifies your organisation.

## 2. Install and scan

No install step of its own — `npx` fetches the right binary for your platform:

```bash
npx getax license install <your-key>    # once, per machine (or set GETAX_LICENSE in CI)
npx getax scan .
```

That's it. `scan` reads your repository and any reports your existing tools already
produce (SARIF, lcov, JUnit XML, OpenAPI) — **your source never leaves your machine;**
the score is computed locally.

## 3. Read the score

```
AX 74  C   model v3   confidence 0.78
├─ T determinism     54   coverage 41% · suite <5m · mutation UNMEASURED (blind spot)
├─ B boundaries      82   82% of files under 400 lines an agent can read whole
├─ D intent         100   README, CLAUDE.md, ADRs, and every commit traces to intent
├─ S type safety     88   8% of files use `any` — the escape hatch from the types
└─ E enforcement     61   a gate exists, but nothing proves it blocks a merge
```

Five signals — the five things an autonomous agent needs to work in a repository
without breaking it:

| | Signal | The question it asks |
|---|---|---|
| **T** | **T**ests | can the agent tell if it succeeded? |
| **B** | **B**oundaries | does the relevant code fit in its context? |
| **D** | **D**ocs | can it know *why*? |
| **S** | **S**chema / static types | is a wrong guess caught before runtime? |
| **E** | **E**nforcement | does anything stop a bad change landing? |

Two things to notice, because they are the point:

- **Every point lost names its cause.** There are no unattributed deductions — the
  detail line beside each signal says exactly what to fix.
- **Unmeasured is not zero.** A blind spot (no coverage report, no test results)
  *widens the confidence range* rather than scoring you a zero — so *adding* a report
  can only ever raise your certainty, never punish you for being honest.

Below the signals, `scan` lists the **cheapest points to recover**, ranked — the
shortest path up.

## 4. What next

- **[Wire GetAX into your workflow](/docs/wiring/)** — make it a gate, not a one-off.
  This is the step that matters: a score you run by hand is a report; a score in your
  gate is a ratchet nobody can quietly walk backwards.
- **[Configuration](/docs/configuration/)** — `.getax/settings.json`: exemptions,
  posture, and archetype, and which settings can move your score.
