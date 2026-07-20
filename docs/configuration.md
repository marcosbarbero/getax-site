---
layout: doc
title: Configuration
permalink: /docs/configuration/
description: How a repository configures GetAX (.getax/settings.json), and which settings can move the score — with a live browser validator.
---

# Configuring GetAX: `.getax/settings.json`

How a repository configures GetAX, what each setting does, and — the load-bearing
part — which settings can move your score and which cannot. This is a reference for
the files themselves; the *reasoning* behind the two-layer split lives in
ADR-0019,
and what the signals mean lives in the AX Standard.

## Validate your config

{% include config-validator.html %}

## The one idea: config splits by whether it can touch the score

GetAX's whole value is a score you can **argue with** — deterministic, reproducible,
and explainable. That only holds if the score moves *only* through inputs that were
reviewed. So configuration is split into two files, divided by a single question:
**can this change the score of record?**

| File | Committed? | Who edits it | Can it move the score? |
|---|---|---|---|
| `.getax/settings.json` | yes — reviewed in the diff | you | **yes** |
| `.getax/settings.local.json` | no — gitignored, personal | you | **no, structurally** |

The **score of record** — the number your CI gates on, the ratchet's floor, anything
uploaded to the hosted dashboard — is computed from `.getax/settings.json` **alone**.
The local file is never read on that path. This is not a rule someone polices: the
local file *has no fields* that could affect a score (a `weights` key in it is a parse
error, not a silent no-op), and it is gitignored, so CI and the server never see it at
all. You can silence anything locally and it cannot reach the shared number, because
the shared number never looks at the local file.

Both files live under one directory, `.getax/`, alongside `.getax/baseline.json` (the
ratchet floor, which the tool writes — leave it alone).

Run `getax fix` once to scaffold both files and add the local one to `.gitignore`.
Run `getax config` any time to print every effective setting and which layer it came
from.

---

## `.getax/settings.json` — the committed, score-affecting config

A JSON object. **Unknown fields are rejected** (a typo'd `"wieghts"` is an error, not
a silent no-op), because a config that says one thing while the score means another is
wrong in a way you cannot see.

```json
{
  "$schema": "https://getax.app/schema/getax.v1.json",
  "version": 1,
  "model": "v6",
  "weights": { "T": 0.25, "B": 0.20, "D": 0.15, "S": 0.20, "E": 0.20 },
  "ignore": ["vendor/**", "**/*.generated.go"],
  "inputs": {
    "coverage": ["build/coverage.out"],
    "junit":    ["build/test-results/*.xml"],
    "sarif":    ["build/semgrep.sarif"],
    "openapi":  ["api/openapi.yaml"]
  },
  "overrides": [
    {
      "rule": "type-strictness",
      "reason": "These are Mixpanel event properties and GeoJSON — open-world JSON by definition; the allowlist at the boundary is where that check belongs.",
      "expires": "2027-01-01"
    }
  ],
  "issues": {
    "labels": ["type:task", "comp:api"],
    "assignees": ["your-triage-bot"]
  }
}
```

### `version` — required
Must be `1`. The config format version (not the scoring model — that's `model`). A
build that doesn't understand your version refuses to score rather than guess.

### `model` — required
The scoring model version to pin, e.g. `"v6"`. **This is the most important field.**

A CLI upgrade must **never silently move your score** — someone's CI ratchet depends
on the number. Pinning the model means a new GetAX release scores you against the
model you chose, not whatever shipped last night. Changing a band, a weight default,
or a penalty curve is a **model version bump plus an ADR** on our side, never a patch
— and you adopt it deliberately by changing this one line.

A repo with no `.getax/settings.json` is *unpinned*: it gets the current model, and
the report says so (`getax config` shows `model … default — not pinned`). Unpinned is
fine for a first look; pin before you gate CI on the number.

**The current default is `v6`.** The versions, and what each adds:

- **`v6`** (default) — everything below, plus the two that changed what a score *means*:
  a **blind sub-signal no longer scores as well as your measured average**. It is priced at
  an empirical prior (0.700 — the median measured value across a corpus of real repositories),
  so publishing a merely-good measurement *raises* your score instead of lowering it. Before
  this, hiding a test report beat publishing one. And a **test report is scored only if you
  declare it** in `inputs.junit` — an auto-discovered one is build output, so scoring it made
  the same commit score differently depending on which suite ran last.
- **`v5`** — a **defunct CI provider is not credited** (a `.travis.yml` where Travis no longer
  runs is dead infrastructure, not a pipeline), and **co-change-coupling blinds on a shallow
  clone** rather than reporting a different number from truncated history. That one matters in
  CI: `actions/checkout` defaults to a depth-1 clone, so **set `fetch-depth: 0`**. See the
  [wiring guide](/docs/wiring/).
- **`v4`** — verified-claim configuration (`exclude`, `posture`; see below), *without* v5's or
  v6's fixes. If you pinned v4 for `exclude`/`posture`, **v6 now carries both** — it is the
  first version that does, and moving to it is a one-line change.
- **`v3`** — the model before any of the above.

Pinning `v3` or `v4` is fully supported — your score does not move until you change this line.

### `weights` — optional
The five signal weights: `T` (determinism), `B` (context/boundaries), `D` (intent),
`S` (type safety), `E` (enforcement). **Must sum to 1.0** (rejected, not rescaled, if
they don't — a config that quietly renormalises is a config you can't reason about).
Omit to use the model's defaults. Changing weights **moves the score**, so it lives in
the committed layer, where the diff reviews it.

### `ignore` — optional
Path globs excluded from probing. Compiled at load, so a bad pattern fails the scan
with the pattern named rather than silently matching nothing. Affects the score (it
changes what is measured), so it's committed.

### `inputs` — optional
Where your build reports actually are, when they aren't where the conventions guess.
Four kinds: `coverage`, `junit`, `sarif`, `openapi` — each an array of path globs.

Under the **`v6` default this is load-bearing for `junit`**: a test report is scored *only*
if you declare it here. An auto-discovered report is build output — it exists only after a
run, and says whatever that run said — so scoring it would make the same commit score
differently depending on which suite ran last on which machine. Declare a **glob**, not a
directory (`["build/test-results/**/*.xml"]`); a pin that matches nothing is reported loudly
rather than silently ignored.

This matters most for **coverage**: coverage reports are build artifacts (gitignored,
produced wherever your build puts them), so any path GetAX hard-codes is a guess about
your repo. Tell us where the report is and **configured paths win over conventions** —
if you point at a path and nothing is there, that's an error we surface, not a cue to
go hunting elsewhere and report a number from a file you didn't mean.

> Note: a report *path* is committed config because finding a real report changes the
> score (it measures a blind spot — honestly). A machine-specific local path that
> should not affect the shared number is a different thing and is not yet supported in
> the local layer; put the canonical, CI-reachable path here.

### `overrides` — optional
Tell GetAX a rule is **wrong for your repo**, and why. This is "Override & Teach": the
finding stops counting against you, deterministically, because it's a reviewed input —
not a learned parameter.

Each override:

| Field | Required | Meaning |
|---|---|---|
| `rule` | yes | the sub-signal key, e.g. `"change-traceability"` |
| `reason` | **yes** | why it doesn't apply here — **min 20 characters** |
| `expires` | no | `YYYY-MM-DD`; the override lapses after this date |
| `scope` | no | a path glob narrowing the override to part of the tree |
| `author` | no | who decided (git blame answers this too) |

The **`reason` is mandatory on purpose** and it's the whole point. A reasonless
override is just a lower standard with extra steps. The reasons are a teaching signal:
in aggregate, "22 of 37 teams overriding this rule cite generated code" is evidence the
*rule* is wrong, and a human fixes it in the next model version. 20 characters doesn't
guarantee thought — it guarantees skipping thought is a deliberate act.

`expires` exists so overrides **rot loudly, not silently**. An override with no expiry
is a decision nobody revisits — which is how a repo ends up scoring well by permanent
exemption. Set a date; when it passes, the finding comes back and you re-decide.

The score report lists every applied override and where it came from, because holding
one silently would make the number unexplainable to the person who most needs to
explain it. Determinism is "same inputs → same score, **and the report names its
inputs**."

### `issues` — optional
How this repo wants findings filed: `labels` and `assignees` applied when you file a
finding as a GitHub issue. **Inert to the score** — it's repo configuration that
happens to share the file. Empty by default and deliberately: GitHub *silently drops*
a label that doesn't exist in the target repo, so a guessed taxonomy files unlabelled
and looks like it worked. Tell us your labels; we won't guess them.

---

## Verified config — `exclude` and `posture` (requires `"model": "v4"`)

The principle (ADR-0020): committed config supplies **claims the engine verifies and
recomputes from**, never values it trusts. A raw `ignore` glob is trusted; a verified
claim is *earned*. These require a **Claims-capable model — `v4` or `v6`** — and a repo that
writes them under an older pin gets a clear error, never a silent no-op.

> **Prefer `v6`.** `v4` carries verified claims but none of the correctness fixes that came
> after it; `v6` carries both, and it is the default. If you pinned `v4` only to get
> `exclude`/`posture`, moving to `v6` is a one-line change that costs you nothing.

### `exclude` — verified exemptions
Instead of trusting a glob, you make a claim about a set of paths, and the engine
checks it:

```json
"exclude": [
  { "path": "**/i18n/**", "claim": "data",
    "reason": "Localisation catalogs — key/value data, not logic.", "expires": "2027-01-01" }
]
```

| Field | Required | Meaning |
|---|---|---|
| `path` | yes | a glob for the files this claim covers |
| `claim` | yes | `generated` \| `vendored` \| `data` — the closed, verifiable vocabulary |
| `reason` | yes | why (min 20 chars), same as an override |
| `expires` | no | `YYYY-MM-DD`; the exemption lapses after this |

The engine verifies the claim against the matched files — `generated` by codegen
convention or a `DO NOT EDIT` header, `vendored` by directory, `data` by content shape
(overwhelmingly key/value, almost no control flow). **If it holds**, those files are
exempt from the affected signal. **If it fails**, nothing is exempt, and the report's
**suppression ledger** names the evidence: `hand.go generated → REJECTED (1/1 failed)`.
You cannot raise the score by writing a claim — only by the claim being true.

### `posture` — declare your gate, and we verify it
For a repo whose gate is a committed pre-push hook rather than CI:

```json
"posture": { "gate": "pre-push-hook" }
```

This does **not** waive the enforcement signal — it *redirects* it. The scanner points
at `.githooks/pre-push` and checks whether it actually runs your tests. A hook that
runs them earns the same ceiling as a CI that does (a hook is per-clone, so we see the
mechanism but can't confirm it's installed); a hook that runs nothing earns nothing.
`gate` is a closed vocabulary — today, `pre-push-hook`.

### `archetype` — one rubric, fair across repo kinds
Declare what kind of repo this is, and the scorer uses a weight profile suited to it —
so a CLI isn't marked down for having no HTTP API, and a data pipeline isn't marked
down for long files:

```json
"archetype": "service"
```

| archetype | leans into |
|---|---|
| `general` *(default)* | the balanced base — today's weights |
| `service` | determinism + enforcement (deployed: verify the change, bind the gate) |
| `library` | type safety + intent (the API surface *is* the product) |
| `cli` | tests + gate; `api-schema` doesn't apply |
| `data-heavy` | correctness at the data boundary; long files are the nature, not a defect |

It's a **closed vocabulary** (so scores stay comparable) and it's **verified with
evidence** — as strictly as an `exclude` claim. The archetypes that *ease* a signal have
to earn it: `library` (which carries the lowest enforcement weight) is rejected if the
repo has deploy markers; `data-heavy` (which eases the boundaries weight) is rejected
unless data is a real share of the repo — *"data is 4% of your code+data files — this
leads with code."* A rejected archetype falls back to `general` and the report names why.
The archetypes that only *raise* the hard-to-fake signals (`service`, `cli`) are
accepted. You **can't** set both `archetype` and `weights` (an archetype *is* a weight
profile). The profiles are a calibration point and will be refined as we score real
repos of each kind.

---

## `.getax/settings.local.json` — the personal, display-only layer

Gitignored, per-developer, and **structurally unable to move the score of record**.
`getax fix` scaffolds it and adds it to `.gitignore`. Unknown fields are rejected — you
*cannot* put a `model`, `weights`, or `overrides` key here; that's a parse error
telling you it belongs in `.getax/settings.json`, where it's reviewed.

```json
{
  "quiet": ["suite-speed", "type-strictness"]
}
```

### `quiet` — optional
Sub-signal keys to hide from **your** `getax scan` recovery list — "I've seen this,
stop reminding me on my machine." It does **not** change the score, which findings
count, or the `--json` output; the finding still appears in the signal breakdown and
still counts. And it's never silent: the report ledgers `N quieted locally … still
scored`, so a local mute is visible, not invisible state.

Use it for things you've acknowledged but haven't yet committed an override for. To
make a suppression *travel* — reviewed, applied by CI and the CLI — write an
`override` in `.getax/settings.json` instead.

---

## Seeing what's in effect: `getax config`

Prints every effective setting under its layer, so the firewall is visible rather than
merely asserted:

```
  getax configuration  .

  COMMITTED  .getax/settings.json — reviewed in the diff, may move the score
    model       v6                pinned
    version     1
    weights     default
    ignore      vendor/**
    inputs      coverage:1
    overrides   1 — type-strictness
    issues      labels:type:task/comp:api

  LOCAL      .getax/settings.local.json — personal, gitignored, cannot move the score
    quiet       suite-speed       display only — these findings still count
```

## Determinism guarantees, in one place

- **Same inputs → same score**, byte for byte. No wall-clock, no network, no
  randomness, no generative AI in scoring.
- **A CLI upgrade never silently moves your score** — the model is pinned in
  `.getax/settings.json`; adopting a new model is a line change you make.
- **Every point lost names its cause**, and every override that changed the number is
  listed with its reason. Nothing is deducted unattributed.
- **The local layer cannot touch the number that gates your repo** — it's not read on
  the score-of-record path, and it's gitignored, so CI and the server never see it.

## Reference

- ADR-0019 — why config splits by whether it can touch the score
- The AX Standard — what T/B/D/S/E mean and how they score
- Override & Teach / ADR-0008 — why an override is a reviewed input, not learning
