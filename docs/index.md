---
layout: doc
title: Docs
permalink: /docs/
description: GetAX guides — get started, wire it into your workflow, and configure it.
---

# GetAX docs

GetAX scores how ready a repository is for autonomous agents — deterministically — and
shows the reasoning behind every point. These guides take you from a first scan to a
gate that blocks regressions on every push.

## Start here

- **[Getting started](/docs/getting-started/)** — install, scan a repository, and read
  the result. Five minutes.
- **[Wire GetAX into your workflow](/docs/wiring/)** — turn that one-off scan into a
  gate: a baseline, a pre-push hook, and CI, so your agent-readiness can only ratchet
  up. This is where GetAX stops being a tool you remember and becomes one you rely on.

## Going deeper

- **[Judgment — the model-graded probe](/docs/judgment/)** — the readiness questions a
  deterministic check cannot answer (is this long file a monolith or a flat registry?),
  asked of a model you configure. Opt-in, advisory, and it never moves your score.

## Reference

- **[Configuration](/docs/configuration/)** — `.getax/settings.json`: exemptions,
  posture, and archetype, and which settings can move your score. Includes a live
  browser validator.
