---
layout: doc
title: Judgment — the model-graded probe
permalink: /docs/judgment/
description: Some readiness questions a filesystem cannot answer — is this 1,500-line file a monolith or a flat registry? Judgment asks a model, as an opt-in, advisory pass that never moves your deterministic score.
---

# Judgment — the model-graded probe

The [AX score](/docs/getting-started/) is deterministic: same repository, same number,
every time, computed on your machine with no model in the loop. That is the whole point
of it — a score you can argue with, reproduce, and gate on.

But a few readiness questions are genuinely beyond what a deterministic check can see.
*Is this 1,500-line file a monolith an agent cannot safely change one thing in — or a
flat registry of independent entries it can extend by adding one line?* Both are long.
Only reading the code tells them apart, and reading is what a model does. That is
**judgment**, and it is a separate, optional layer on top of the score.

Two promises hold, and they are the reason this feature is safe to turn on:

- **Judgment is advisory. It never moves your score.** The AX number stays
  deterministic and reproducible. Judgment adds *flags* — "this file looks too tangled
  to hold" — it never adds or removes a point. Your CI ratchet cannot step because of it.
- **Your source still never leaves your machine — unless you ask.** The default probe
  is fully offline. The model pass is opt-in, points at *your* endpoint, and announces
  every byte before it sends it.

## The deterministic probe: `getax probe`

Before the model, there is the part that needs no model. `getax probe` attempts the
tasks an agent would on a fresh repository — read the contract, run each command it
promises, find each module it maps — and reports where the repository stopped it. No
network, no key, always available:

```bash
npx getax probe .
```

```
Probe of . — an agent completed 7 of 9 tasks it could try here.

  stuck [contract-lies] CLAUDE.md names `make test-integration`, but it does not exist.
  stuck [map-incomplete] internal/billing/ is a real module the contract never names.

agent-reached: 78 / 100
```

This is the same "an agent that goes looking for the promised gate and finds nothing"
check GetAX runs on itself. It is offline and deterministic, like the score.

## The judgment pass: `getax probe --judge`

`--judge` adds the questions a filesystem cannot answer. It is **opt-in**, **advisory**,
and it sends source to a model **you** configure — so it tells you before it does:

```bash
export GETAX_JUDGE_KEY=...                       # your key
export GETAX_JUDGE_ENDPOINT=https://...           # your endpoint (defaults to OpenAI)
export GETAX_JUDGE_MODEL=...                       # your model

npx getax probe --judge .
```

```
⚠ --judge is sending 8 file(s) to https://your-endpoint/v1 — source is leaving this machine.
Probe of . — an agent completed 9 of 9 tasks it could try here.

Judgment advisories (advisory — does NOT affect any score):
  flag [judgment] internal/orders/orders.go:19 — fuses HTTP, SQL, pricing, and email
       in one type; a scoped change cannot be made without reading the whole file.
```

Point it at anything that speaks the OpenAI chat API — OpenAI, Azure OpenAI, a gateway,
or a self-hosted model on your own hardware. Nothing is sent anywhere until `--judge`
is present *and* a model is configured; without a model, `--judge` is an error, never a
silent call. It grades the largest source files (where tangle is both likeliest and
costliest), up to a bounded number per run, and skips tests and generated code.

Every flag carries a **verified citation** — a `path:line` that GetAX confirmed exists
in your tree before showing it. A model that hallucinates a location gets its finding
dropped, not shown. That is what makes a flag worth reading regardless of which model
produced it.

## Certify your model first — the flagship is not automatically the best judge

Judgment is only as good as the model behind it, and *good* here is not what you would
guess. In our own benchmark, a smaller, cheaper model scored **precision 0.93 / recall
0.87**, while a larger, more capable one scored **1.00 / 0.27** — near-perfect trust but
almost blind, because it read the "cite one specific line" instruction so literally it
refused to flag whole-file tangles at all.

The lesson: **measure your model, do not assume it.** GetAX ships a labelled benchmark
corpus and the harness to run any model against it, so a deployment certifies its own
endpoint and learns where it lands:

- **precision** — of what it flagged, how much was real. This guards *trust*: a false
  flag trains people to ignore the tool.
- **recall** — of the real tangles, how many it caught. This guards *value*: a weak
  model reaches everything and finds nothing.

A high-precision, modest-recall model is a fine advisor — it will not cry wolf, it will
just miss some things. A low-precision model belongs nowhere near a gate. Because
judgment is advisory here, a miss costs you a flag you did not get; it never costs you a
point.

## Where it runs

- **Locally, in your CLI or CI** — `getax probe --judge`, against your own endpoint.
  Nothing central, no server: the repository, your key, your model.
- **Server-side, on a licensed dashboard** — the same judgment runs on the scans GetAX
  performs for you, attaching the same advisories to the report, so a team sees them
  without configuring anything per machine.

Either way the rule is the same one that makes the whole product trustworthy: the score
is deterministic and yours to reproduce, and the model only ever *advises* on top of it.
