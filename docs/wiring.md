---
layout: doc
title: Wire GetAX into your workflow
permalink: /docs/wiring/
description: Turn a one-off scan into a gate — set a baseline, block regressions on every push and in CI, so your agent-readiness can only ratchet up.
---

# Wire GetAX into your workflow

A score you run by hand is a report. A score in your gate is a **ratchet** — a floor
your repository can rise above but never quietly slip below. This is the difference
between GetAX being a tool you remember and infrastructure you rely on, and it takes
about ten minutes to set up once.

Everything here is exactly what GetAX's *own* repository does to itself — the same
`getax check` runs in its pre-push hook and its CI. We gate our pushes on our own score.

## 1. Set the floor

`getax check` compares today's score against a committed baseline and **fails if it
went backwards.** Record the baseline once:

```bash
npx getax check --update      # writes .getax/baseline.json — the floor, in your repo
git add .getax/baseline.json && git commit -m "getax: set the AX baseline"
```

The baseline lives **in your repository**, not in our cloud — it travels with the code,
it is reviewed like the code, and your CI reads it without asking us anything.

## 2. Block bad pushes locally (pre-push hook)

Add `getax check` to a pre-push hook so a push that lowers the score never leaves your
machine:

```bash
# .githooks/pre-push
#!/usr/bin/env bash
npx getax check || {
  echo "getax: this push would lower your agent-readiness. Run 'npx getax scan .' to see why."
  exit 1
}
```

```bash
chmod +x .githooks/pre-push
git config core.hooksPath .githooks     # per clone; commit the hook so the team shares it
```

Now the ratchet is automatic: the score can go up (run `getax check --update` when you
*earn* a higher floor), but it cannot drift down a point at a time unnoticed.

## 3. Block bad merges in CI (GitHub Actions)

The hook covers your machine; CI covers everyone else's — outside contributors, and
anyone whose clone never set `core.hooksPath`. Add a workflow:

```yaml
# .github/workflows/getax.yml
name: getax
on: [push, pull_request]
jobs:
  ax:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0            # full history — GetAX reads it for change-traceability
      - run: npx getax check
        env:
          GETAX_LICENSE: ${{ secrets.GETAX_LICENSE }}   # your CLI key, as a repo secret
```

Then make it a **required status check** in your branch protection rules — that is the
one thing that makes the gate actually *block* a merge rather than merely report on it.

## 4. Gate on whether an agent could work at all

The composite score wobbles as blind spots open and close. Two checks are rock-steady
and worth gating on directly:

```bash
npx getax check --probe        # would an autonomous agent get stuck here? (deterministic)
npx getax check --signal enforcement --min 70   # hold one signal to a floor
```

`--probe` runs the deterministic probe — it tries the tasks a repository should let an
agent complete (find the gate the contract promises, locate the module the map names)
and fails if the agent would get stuck. It is the most direct answer to "is this repo
ready," and it does not move as coverage reports come and go.

## 5. Commit the config so it travels

Two files make GetAX reproducible for your whole team and your CI:

- **`.getax/baseline.json`** — the floor (step 1).
- **`.getax/settings.json`** — exemptions, posture, and archetype
  ([configuration reference](/docs/configuration/)). Committed and reviewed, because
  anything that can move the score belongs in the same place the code does.

Commit both. From then on `npx getax check` means the same thing on every laptop and in
CI — which is the whole point of a ratchet: one floor, one meaning, nobody walking it
back.

## The result

A pull request that makes the repository *harder* for an agent to work in now fails a
check, with a named cause and the cheapest fix — before it merges, not after someone
notices the number slipped. That is GetAX doing its job while you do yours.
