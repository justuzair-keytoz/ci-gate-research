# v2 Research: PR Stacks

Status: research only, no decisions made, no code changed.

## The question this answers

Your senior asked: when a PR stack is open, does the gate test the cumulative diff across the whole
stack, or just the diff between each branch and its immediate parent/child? How do we verify things work
end-to-end across the stack?

## First finding: this isn't hypothetical, it's already the dominant workflow here

Checked this repo's real, currently-open PRs (49 total, via `gh pr list --json number,baseRefName`).
**27 of 49 — more than half — target a base branch other than `main`.** This repo is already deep into
PR stacking today, just without a dedicated tool (no Graphite, no git-spice — people are doing it by
hand, setting a PR's base to another open branch).

Two stacks go remarkably deep:

- `DATA-261` series: PR #323 is **8 layers** deep (`#323 ← #322 ← #321 ← #320 ← #318 ← #317/#316 ← #315 ← #287 ...` down to `main`).
- `DATA-345` series: PR #335 is **6 layers** deep (`#335 ← #334 ← #333 ← #332 ← #331 ← #330 ← main`).

Full chain data is reproducible:
```
gh pr list --limit 60 --state open --json number,baseRefName,headRefName
```

**Why this matters immediately:** the `pr-gate.yml` workflow built in the POC (see `PLAN.md`) already
runs on every `pull_request` event — including these stacked PRs, today, as-is. Nobody designed it with
stacking in mind. The question is whether its current behavior is actually correct for a stack, or just
accidentally hasn't broken yet because the gate isn't required yet.

## What GitHub itself does with stacked PRs (default behavior, no extra tooling)

GitHub shipped a native "stacked pull requests" feature in 2026. Its documented CI behavior, by default:

- **Every PR's diff is base-relative** — if PR B's base is PR A's branch (not `main`), B's diff only
  shows B's own changes, compared to A. This is true whether or not you use GitHub's stacking UI — it's
  just how `base`/`head` has always worked. **So the answer to "cumulative vs. just-the-parent" is: by
  default, GitHub already scopes each PR's diff to its immediate base, not the cumulative diff against
  `main`.** This repo's manual stacking (no tool) gets this for free already.
- **But CI still runs separately, in full, for every layer.** GitHub's own docs: "CI checks triggered by
  pull requests on your default branch run for all layers, not just the bottom one." Branch protection
  and required-reviewer rules are also enforced at every layer, even mid-stack ones that don't target
  `main` directly.
- **Net effect, unmodified:** our `pr-gate.yml` already tests "just the diff vs. the immediate parent"
  correctly (that's just what `pull_request` events give you) — but it also means a stack 8 layers deep
  triggers the full gate workflow 8 separate times, once per layer, each time it's pushed to.

Source: [GitHub Docs — About stacked pull requests](https://docs.github.com/en/pull-requests/get-started/about-stacked-prs)

## What dedicated stacking tools add on top (Graphite, git-spice)

Neither is in use in this repo today (no `graphite.json`, no `.git-spice` state found), but both
converge on the same answer for "how do you avoid re-testing the same thing N times down a stack":

**Tiered checks, not cumulative re-validation.** The consistent industry pattern, across every source
checked:

> "CI for stacked PRs should be tiered (smoke per PR, full suite once per stack), or you'll pay N× for
> the same signal." — independent guidance cited in Graphite's own docs

Concretely: fast required checks (lint, build, unit tests — a few minutes) run on *every* layer of the
stack, because they're cheap enough that redundancy doesn't matter. Slow/expensive checks (full
integration suites, e2e) run once — either only on the bottom-most PR, only at merge time, or on a
schedule — specifically to avoid multiplying an expensive signal across every layer for no new
information.

Graphite goes further with an opt-in "CI Optimizations" feature: a small API your CI calls to decide
whether a given layer's CI even needs to run at all, configurable per-repo (e.g. "only run full CI on
the bottom 2 PRs and the top PR of each stack"). This is a paid/tooling-specific feature, not something
this repo gets without adopting Graphite.

Sources:
- [Graphite Docs — CI Optimizations](https://graphite.com/docs/stacking-and-ci)
- [git-spice](https://abhinav.github.io/git-spice/) / [git-spice README](https://github.com/abhinav/git-spice/blob/main/README.md)

## "How does it verify end-to-end across the stack" — the real gap

Base-relative diffs plus tiered per-layer checks answer *most* of your senior's question, but there's a
real hole: **testing every layer against its immediate parent never proves the whole stack works
together against current `main`.** Two independent, real failure modes neither Graphite's nor GitHub's
default behavior fully closes:

1. **Stack drift.** If `main` moves forward (someone else merges) while a stack sits open, every layer's
   PR is still diffed against its *original* parent — not against where `main` is *now*. A layer can
   stay green while the stack, merged as a whole, would conflict or break against current `main`.
2. **Integration-only bugs.** Two layers can each pass their own narrow, scoped tests and still break
   when actually combined — e.g. layer 3 changes a function's signature, layer 5 (built on top) still
   calls it the old way, but layer 5's own CI only re-tests layer 5's diff, not the combined code it
   actually runs.

Industry answer to both: **a full-stack validation gate before the whole thing lands**, run against the
stack's current state merged onto current `main` — either at "merge the whole stack" time (what
Graphite's "Merge (N) PRs" button does, re-checking and re-running CI at each merge step bottom-up,
re-validating against the real current state of `main` as it goes) or via a **merge queue**, which tests
a candidate merge (the PR plus whatever's ahead of it in queue) against current `main` right before
actually merging — explicitly the standard fix for "two individually green PRs combine into a red
`main`." The original brief already lists merge queue as out of scope for now ("revisit after the gate
is stable") — this research doesn't change that, just names it as the actual mechanism that closes this
gap, for whenever it's revisited.

## What this means for this repo's gate, concretely (not yet implemented)

Given the glossary above and these findings, specifically for `pr-gate.yml`:

- **No change needed to "which diff gets tested."** `pull_request` events already give a base-relative
  diff. A PR against a parent branch already only sees its own changes. This matches industry default
  behavior and doesn't need fixing.
- **Tiered checks already partially exist, by accident.** The current gate is fast-only (build/lint/test
  + Docker build-validate) — there's no slow/e2e tier yet to worry about re-running down a stack. This
  becomes relevant the moment e2e tests get added (see `2-per-dir-test-dispatch-research.md`) — at that
  point, the tiering pattern above (e2e once per stack, not per layer) needs deciding.
- **The real open question for this repo specifically:** with 27 of 49 PRs already stacked, and some 8
  layers deep, how much redundant gate-running is already happening today, invisibly, because nobody's
  looked? That's measurable once the gate goes live (count: how many times does `pr-gate` run per stack,
  total, across its full depth, before the bottom PR merges) — flagging as a thing to actually watch once
  this is live, not something to solve speculatively now.
- **Stack drift and cross-layer integration bugs are real gaps, not solved by this gate alone,** and the
  standard fix (merge queue) is already correctly out of scope per the brief. Worth naming explicitly so
  it isn't mistaken for "solved" once the basic gate ships.

## Sources

- [GitHub Docs — About stacked pull requests](https://docs.github.com/en/pull-requests/get-started/about-stacked-prs)
- [Graphite Docs — Merge Pull Requests](https://graphite.com/docs/merge-pull-requests)
- [Graphite Docs — CI Optimizations](https://graphite.com/docs/stacking-and-ci)
- [git-spice](https://abhinav.github.io/git-spice/)
- [git-spice README](https://github.com/abhinav/git-spice/blob/main/README.md)
- This repo's own `gh pr list` output (see "First finding" above) — reproducible, not a third-party claim.
