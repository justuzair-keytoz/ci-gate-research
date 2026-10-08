# OSS Case Study: `calcom/cal.com`'s CI Gate

Status: research only, no decisions made, no code changed in either repo. Read `langchain-analysis.md`
first — this doc focuses on what's different or new, same convention as the other `v2/oss/` docs.

## The question this answers

You pointed at this repo specifically to look at `.github/workflows/ci.yml` for "multi-app aggregate
gating (root check)." There's no file literally named `ci.yml` in this clone — the real root gate is
`.github/workflows/pr.yml` (the one that actually runs on every pull request), with a second, related
file `all-checks.yml` that runs the same checks unconditionally for GitHub's merge queue. This is the
richest example of an aggregating root gate found across all four OSS case studies so far, and it
directly surfaces **two findings that change how the target repo's own gate should be evaluated**: a
confirmed, production-grade PR-stacking integration (Graphite), and a materially safer default-direction
for change detection than this repo's own POC currently uses.

Cal.com is a JS/TS monorepo (Next.js app, a separate `apps/api/v2` NestJS API, an `atoms`
component-library package, many `packages/*`) — structurally close to the target repo's own shape
(a Next.js UI plus several separate Node services).

All paths below are relative to `tmp/cal.com/`.

## 1. The root gate: `pr.yml`, and how it's actually built

Unlike every repo analyzed so far (langchain, Next.js, Vault — all single large workflow files with many
inline jobs), cal.com's `pr.yml` is a thin **orchestrator** that calls out to ~20 separate reusable
workflow files via `uses: ./.github/workflows/<name>.yml` (`workflow_call`-triggered — confirmed e.g.
`check-types.yml`'s trigger block is just `on: workflow_call`, nothing else). Each concern — type
checking, linting, unit tests, API-v2 unit tests, security audit, Prisma migration checks, multiple
separate builds, multiple separate e2e suites — lives in its own file, called conditionally from `pr.yml`
based on boolean outputs from one `prepare` job. This is a distinct architectural choice from the other
three repos' "one big file, many inline jobs" pattern — same effect (conditional jobs feeding one
aggregating gate), different packaging (composition via reusable workflows instead of one monolithic
file).

### The aggregating `required` job (`pr.yml` lines ~267-305)

```yaml
required:
  needs: [trust-check, prepare, lint, type-check, unit-test, api-v2-unit-test, security-audit,
          check-prisma-migrations, integration-test, build, build-api-v2, build-atoms, setup-db,
          e2e, e2e-api-v2, e2e-embed, e2e-embed-react, e2e-app-store]
  if: always()
  runs-on: ubuntu-latest
  steps:
    - name: Fail if trust-check did not succeed
      run: echo "::error::..." && exit 1
      if: needs.trust-check.result != 'success'
    - name: Fail if PR is not trusted (external contributor without run-ci label)
      run: echo "::error::..." && exit 1
      if: needs.trust-check.outputs.is-trusted != 'true' && needs.trust-check.result == 'success'
    - name: Fail if conditional jobs failed
      run: exit 1
      if: |
        (needs.prepare.outputs.has-files-requiring-all-checks == 'true' &&
         (needs.lint.result != 'success' || needs.type-check.result != 'success' || ...))
```

Same `if: always()` + inspect-upstream-results shape as every other repo's aggregating job (this repo's
own `gate`, langchain's `ci_success`, Next.js's `tests-pass`, Vault's `tests-completed`) — but with a
**three-way split** none of the other three repos have: (1) did the trust-check infra itself run
successfully, (2) is this PR actually trusted to run CI at all, (3) did the real work jobs pass. Compare
to this repo's own `gate` job, which only has one binary check (upstream result is `success`/`skipped` or
it fails) — cal.com's version explicitly distinguishes "the security gate itself broke" from "this PR
isn't authorized" from "the tests genuinely failed," three different operator-facing error messages for
three different root causes.

## 2. Change detection: an inverted default-direction from this repo's own POC — the most actionable finding here

`prepare`'s `filter-exclusions` step (`pr.yml`, `dorny/paths-filter` with `predicate-quantifier: "every"`):

```yaml
filters: |
  has-files-requiring-all-checks:
    - '**'
    - '!.vscode/**'
    - '!**/*.md'
    - '!**/*.mdx'
    - '!.github/CODEOWNERS'
    - '!docs/**'
    - '!help/**'
    - '!packages/i18n/locales/**/common.json'
    - '!i18n.lock'
```

This says: **everything (`'**'`) matches by default, except a short, explicit list of known-safe
non-code paths.** Contrast with this repo's own `detect-services.py`: a hardcoded table of 23 known
service paths, where a changed file only triggers testing if it falls under one of those 23 specific
prefixes — **everything defaults to *not* matching unless explicitly listed.**

**Why this matters concretely for the target repo's own gate, not just as an abstract design
preference:** under the target repo's current exclusion-is-the-default (inclusion-list) model, a
brand-new top-level service directory added to the repo tomorrow gets **zero gate coverage** until
someone remembers to add it to `detect-services.py`'s `SERVICES` table. Under cal.com's model, a new
directory is covered automatically the moment it exists, because it isn't one of the handful of
explicitly-excluded non-code paths. This is a real, concrete blind spot in the target repo's current POC
that none of the three prior OSS analyses (langchain, Next.js, Vault) surfaced this clearly, because all
three of them use the same inclusion-list family of detector as this repo's own POC (hardcoded groups:
`LANGCHAIN_DIRS`, `CHANGE_ITEM_GROUPS`, Vault's HCL `group` blocks) — cal.com is the first one in this
whole research pass that defaults the other way.

Cal.com layers **narrow inclusion filters on top of this broad exclusion-based one**, for checks that are
specifically expensive and only relevant to a known subset — `has-api-v2-changes` and
`has-prisma-changes` are separate, ordinary inclusion-style `dorny/paths-filter` rules (same shape as
everyone else's detector), used only to gate `check-prisma-migrations` and the API-v2-specific jobs. So
the actual pattern is a **hybrid**: broad, safe-by-default coverage for "does this PR need checking at
all," plus narrow, precise inclusion rules only for the small number of checks expensive enough to be
worth skipping when clearly irrelevant.

## 3. Confirmed, production evidence of Graphite (PR stacking) in active use — the second major finding

`.github/workflows/run-ci.yml` (triggered by `pull_request_target: types: [labeled]`, fires when the
`run-ci` or `ready-for-e2e` label is added) contains:

```js
const trustedBotLogins = ['graphite-app[bot]'];
let isAuthorized = false;

if (senderType === 'Bot' && trustedBotLogins.includes(adder)) {
  console.log(`Authorized: trusted GitHub App`);
  isAuthorized = true;
}
```

(`run-ci.yml` lines 24-30). This is **direct, concrete, production evidence that cal.com uses Graphite**
for PR stacking (or at least some Graphite-automated label workflow) in their actual development process
— not vendor documentation, an actual named bot login hardcoded into their CI trust logic, specifically
so that labels Graphite's bot applies are trusted the same way a human maintainer's label would be.

This is the **first repo in this whole v2 research pass with confirmed real-world Graphite usage** —
`1-pr-stack-research.md` and the langchain/Next.js/Vault analyses all discussed Graphite from its own
documentation or found no stacking tool at all; this is the first actual sighting of it wired into a real
company's trust model. Unfortunately, the trace stops there — only one reference to `graphite` exists
anywhere in `.github/` (confirmed via `grep -rli graphite .github/`), so there's no visibility here into
*what* Graphite's bot actually does with those labels, or whether cal.com relies on Graphite's own
"CI Optimizations" feature (described in `1-pr-stack-research.md`) on top of this. What's confirmed: the
trust boundary had to be deliberately widened to accommodate a stacking tool's automation — a concrete
reminder that adopting a stacking tool isn't free from a CI-trust perspective; it requires explicitly
deciding which of that tool's bot identities get to influence what CI does.

## 4. `run-ci.yml` — a self-cancel-and-rerun pattern, useful independent of the fork-trust angle

The rest of `run-ci.yml` (not relying on Graphite specifically) handles the general "a maintainer adds a
label to approve/re-trigger CI" case: find the latest `pr.yml` run for the PR's head SHA, and if it's
still in progress, actively **cancel it (with retry-on-409 up to 10 attempts) and poll until confirmed
cancelled** before calling `reRunWorkflow` on the same run ID (so it reruns in its original PR context,
rather than needing a new push). This is a more deliberate, explicit version of what `concurrency:
cancel-in-progress` does automatically in every repo analyzed so far (including this repo's own
`pr-gate.yml`) — here it's done imperatively, with retry logic and explicit polling, because the trigger
is a label event (which `concurrency` grouping wouldn't naturally cancel the way a new push does).

## 5. Tiering: a human-label gate for the expensive tier, not an automatic one

`ready-for-e2e` is a **manually-applied label**, not an automatic draft-state check (contrast with
Vault's `is-draft == 'false'` auto-gate, found in `vault-analysis.md`). `prepare`'s
`check-if-pr-has-label` step explicitly re-fetches the PR's current labels from the API (not the
webhook's payload, which can be stale) and gates every expensive job (`setup-db`, all builds, all five
separate e2e workflow files) on `needs.prepare.outputs.run-e2e == 'true'`. This means: cal.com's e2e tier
requires an **explicit, human (or Graphite-bot) decision** that this PR is ready for the expensive suite,
not an automatic signal like "left draft state" or "files changed." A stricter, more deliberate gate than
Vault's automatic one — useful to name as a third concrete tiering mechanism alongside Vault's automatic
draft-check and Next.js's "run everything, just sharded" for this repo's own eventual Playwright/e2e
scoping decision.

## 6. Turborepo is present, but — same as Next.js — not what decides what to test

`check-types.yml` (and presumably the other reusable workflow files) sets `TURBO_TOKEN`/`TURBO_TEAM` env
vars, confirming cal.com uses Turborepo with Vercel's remote build cache. But the actual "what changed,
what needs checking" decision in `pr.yml` is still `dorny/paths-filter`-driven, not `turbo run
--affected`. This is the **second** confirmed case (after Next.js) of a real, large, Vercel-ecosystem
project having Turborepo available and still not using its dependency graph for PR-gating dispatch
decisions — reinforcing `nextjs-analysis.md`'s finding that "adopt Turborepo" cannot be assumed to
automatically solve the shared-package dependents-gap this repo already demonstrated.

## 7. `all-checks.yml` — the merge-queue variant, and a stricter failure condition

```yaml
on:
  merge_group:
  workflow_dispatch:
...
required:
  needs: [lint, type-check, unit-test, ..., e2e-app-store]
  if: always()
  steps:
    - name: fail if conditional jobs failed
      if: contains(needs.*.result, 'failure') || contains(needs.*.result, 'skipped') || contains(needs.*.result, 'cancelled')
      run: exit 1
```

Two things worth naming. First, `all-checks.yml` is the thing that actually runs in GitHub's merge queue
(`on: merge_group`) — every job here runs **unconditionally**, no `prepare`/path-filter gating at all,
unlike `pr.yml`. Second, its `required` job's failure condition explicitly includes `'skipped'` as a
failure — the **opposite** of this repo's own `gate` job (and every other repo's aggregating job
analyzed so far), which all treat `skipped` as an acceptable, passing state (that's the entire point of
the aggregating-gate pattern). This is a clean, concrete illustration
of the right way to vary gate strictness by trigger type: **on a PR, skipping an irrelevant check is
fine; right before something actually merges via the queue, nothing may be silently skipped** — every
check must have actually run and passed. Directly relevant to `1-pr-stack-research.md`'s unresolved
"stack drift" gap: this is the concrete mechanism (merge-queue-triggered, skip-intolerant full run) that
closes it, now confirmed in a second real repo after langchain's circumstantial `merge_group` wiring.

## What this means for the target repo's gate, concretely (not yet implemented)

- **The exclusion-based (safe-by-default) change-detection direction is the single most actionable
  finding in this entire `v2/oss/` folder.** It's a concrete, demonstrated way to close a gap the target
  repo's own POC has that none of langchain/Next.js/Vault's inclusion-list detectors would have
  surfaced: new services/directories get zero coverage by default under the current `detect-services.py`
  design. Worth considering independently of, and probably before, the shared-package-dependents question
  already raised in `2-per-dir-test-dispatch-research.md` — this is a different gap (new/unlisted
  directories, not existing shared packages).
- **Graphite is confirmed in real production use at a real company**, with a concrete, minimal
  description of what adopting it costs from a CI-trust perspective (one explicitly-trusted bot login).
  If the target repo's own 27-of-49-stacked-PRs problem (`1-pr-stack-research.md`) is ever revisited with
  Graphite specifically in mind (rather than Next.js's home-grown `pr_stack_optimizer.yml` approach, found
  in `nextjs-analysis.md`), this is real evidence it works in practice, not just in Graphite's own docs.
- **Merge-queue-triggered gates should be stricter than PR-triggered gates — skip-intolerant, not
  skip-tolerant.** Confirmed in a second repo now (after langchain's circumstantial finding). Directly
  relevant if the target repo ever adopts a merge queue (already correctly out of scope for now, per the
  original brief) — the right design, when it happens, is two gate variants, not one.
- **A three-way aggregating-job failure split (infra broke / not authorized / tests failed)** is a
  usability improvement worth considering for the target repo's own `gate` job once there's more than one
  reason it could fail — right now it's a single binary check, which is fine at this POC's current
  complexity, but cal.com's version is evidence of where that naturally grows as more concerns (like a
  trust gate, if fork PRs are ever allowed) get added.

## Sources

- `.github/workflows/pr.yml` (full file read)
- `.github/workflows/all-checks.yml` (full file read, 84 lines)
- `.github/workflows/run-ci.yml` (full file read, 140 lines)
- `.github/workflows/check-types.yml` (header/trigger block read, to confirm `workflow_call` +
  Turborepo env vars)
- `grep -rli "graphite" .github/` — confirmed exactly one reference, in `run-ci.yml`
- `.github/workflows/` directory listing (`ls`) — confirmed no file literally named `ci.yml` exists;
  `pr.yml` and `all-checks.yml` are the real root gates
- `../langchain-analysis.md`, `../nextjs-analysis.md`, `../vault-analysis.md` (read first per convention,
  to avoid duplicating findings)
- `../../1-pr-stack-research.md`, `../../2-per-dir-test-dispatch-research.md`,
  `../../PLAN.md` (read first per convention)
