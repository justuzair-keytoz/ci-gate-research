# OSS Case Study: `vercel/next.js`'s CI Gate

Status: research only, no decisions made, no code changed in either repo. Read `langchain-analysis.md`
first — this doc assumes it and only calls out what Next.js does *differently or additionally*.

## The question this answers

`langchain-analysis.md` covered a Python monorepo's CI. This doc checks `vercel/next.js`, shallow-cloned
to `tmp/next.js` — a JS/TS monorepo from the same vendor (Vercel) as Turborepo itself, and the owner of
one of the largest E2E suites in the OSS JS ecosystem (`test/e2e`). It directly targets both open
questions: does a Vercel project actually use Turborepo's own affected-detection for PR gating, and how
does a suite this large get scoped or tiered for per-PR CI. It also turned up a **direct, concrete answer
to the PR-stack question** that neither `1-pr-stack-research.md` nor `langchain-analysis.md` found
anywhere else in this research pass — see Q4.

All paths below are relative to `tmp/next.js/`.

## 1. How does Next.js decide what to build/test for a given PR?

**Not Turborepo's `--affected`/`--filter`.** `turbo.json` exists (`turbo.json`, 35 lines) and defines
task graphs for `build`, `dev`, `storybook`, `test-storybook`, `pack-for-isolated-tests`, `typescript`,
and a repo-root `//#get-test-timings` task — real dependency-graph tasks that Turborepo understands and
can run `dependsOn: ["^build"]`-ordered. But **the PR-gating workflow itself never invokes
`turbo run --affected` or `turbo run --filter=[HEAD^1]`** anywhere in `.github/workflows/`. Turbo is used
within `build_and_test.yml` only for specific named tasks, always run in full for everyone the job
applies to — e.g. `build_reusable.yml`'s `afterBuild: pnpm dlx turbo@${TURBO_VERSION} run test-cargo-unit
${TURBO_ARGS}` (`build_and_test.yml` line 316) and `run rust-check ${TURBO_ARGS}` (line 368) — not
`turbo run test --affected`.

Instead, what gates full tiers on/off is a **custom path-matching script**, structurally the same
mechanism as this repo's `detect-services.py` and langchain's `check_diff.py`:

- `scripts/run-for-change.mjs` defines `CHANGE_ITEM_GROUPS` — a hand-written object mapping category
  names (`docs`, `cna`, `next-codemod`, `next-swc`, `turbopack`, etc.) to arrays of path prefixes
  (`scripts/run-for-change.mjs` lines 9-60+). It is invoked from `build_and_test.yml`'s `changes` job:
  ```
  echo "$(node scripts/run-for-change.mjs --not --type docs --exec echo 'false')" >> $GITHUB_OUTPUT
  ```
  (`build_and_test.yml` line 54, for `docs`) and identically for `type turbopack` (line 54 region,
  `turbopack-change` step). This produces two booleans, `docs-only` and `turbopack-only`
  (`build_and_test.yml` lines 68-78), that every expensive test job's `if:` condition checks — e.g.
  `if: ${{ needs.optimize-ci.outputs.skip == 'false' && needs.changes.outputs.docs-only == 'false' &&
  needs.changes.outputs.turbopack-only == 'false' }}` (line 962, and repeated across most `test-*` jobs).
- This is **coarse, binary gating** ("is this PR docs-only / turbopack-only, yes or no") — not
  dependency-graph-aware selection of *which* packages/tests to run. It only decides whether to skip
  entire tiers wholesale, not which subset of the huge test matrix applies.

**Does this give dependency-graph-awareness "for free" via Turborepo?** No — not in the PR-gating path.
`turbo.json`'s task graph is real and would support `--affected`, but the workflows don't call it that
way for test/build dispatch. The target repo's gap (shared-package consumers not re-tested) is **not
closed by anything found in Next.js's actual CI config** — Next.js's own PR-gating logic is, like
langchain's and the target repo's, fundamentally path-prefix-based, just wrapped in a different custom
script. The one place Turborepo's dependency graph is load-bearing is ordering *within* a single `turbo
run <task>` invocation (so `^build` dependents build in the right order before the task itself runs) —
that's an execution-order guarantee, not a "decide what to test for this diff" guarantee.

## 2. How is the huge `test/e2e` suite scoped or tiered? (most important question)

**Brute-force matrix sharding across the full suite, every non-docs/non-turbopack-only PR — not
git-diff-based selection of which e2e specs to run.** Concrete evidence:

- `test-dev` (`build_and_test.yml` lines ~954-990) runs the entire `test/e2e` + `test/development`
  suite split into **10 shards** via matrix: `group: [1/10, 2/10, 3/10, 4/10, 5/10, 6/10, 7/10, 8/10,
  9/10, 10/10]`, crossed with `react: ['', '18.3.1']` (two React versions) — up to 20 parallel jobs for
  this one tier alone. Each shard runs:
  ```
  node run-tests.js --timings --require-timings -g ${{ matrix.group }} --type development
  ```
- The *same* 10-way sharding pattern repeats for `test-turbopack-dev` (lines 446-489), `test-rspack-dev`
  (531-579), `test-rspack-production` (580-624), and analogous production-mode jobs — each is the **full**
  e2e suite for that mode, sharded 10 ways, not a changed-file-filtered subset.
- **Shard assignment is timing-based bin-packing, not git-diff-based.** `run-tests.js` (1164 lines) reads
  group args like `1/10` and partitions tests by **historical duration**, not by what changed:
  ```js
  // get the smallest group time to add current one to
  for (let i = 1; i < groupTotal; i++) { ... }
  groups[smallestGroupIdx].push(test)
  groupTimes[smallestGroupIdx] += prevTimings[test.file] || 1
  ```
  (`run-tests.js` lines 583-600) — a greedy load-balancer that assigns each test file to whichever shard
  currently has the least accumulated time, using a `test-timings.json` fetched from a Vercel KV store
  (`fetchKVTimings`/`KV_TIMINGS_KEY`, `run-tests.js` lines ~200, 388-395) or disk cache, falling back to
  plain round-robin (`idx % groupTotal === groupPos - 1`, line 622) if no timing data exists. `--require-timings`
  (line 174-177) can make a shard fail outright rather than silently fall back to round-robin, if timing
  data is expected but missing. **This is purely about balancing wall-clock time across shards evenly —
  it has nothing to do with which files a given PR touched.** Every PR pays for the full suite; the only
  optimization is making the 10-way split as evenly timed as possible.
- **No `--only-changed`-equivalent found for `test/e2e`.** `2-per-dir-test-dispatch-research.md` names
  Playwright's `--only-changed` flag as a documented-but-unadopted pattern; Next.js's own suite (which
  isn't Playwright — it's a custom runner via `run-tests.js` against `test/e2e`/`test/development`/
  `test/production`) has no analogous changed-file filter in the PR-gating path. The closest thing is the
  **separate** `test-new-tests-dev` / `test-new-tests-start` / `test-new-tests-deploy` /
  `test-new-tests-deploy-cache-components` jobs (`build_and_test.yml` lines 797-951), which run
  `scripts/test-new-tests.mjs --flake-detection --group N/5` — but these are **additive, not a
  replacement for the full suite**: they specifically re-run *new or changed test files* multiple times
  to catch flakiness (hence `--flake-detection`, and a 120-minute timeout "as tests are intentionally run
  multiple times to detect flakes," line 806 comment) — a flake-hunting job, not a scoping mechanism that
  reduces what the full-suite jobs above already run.

**Net for Q2, the central finding:** Next.js's answer to "e2e suite too big to run naively" is **sharded
brute-force parallelism with smart timing-based load balancing, not smart test selection.** Every
non-docs/non-turbopack-only PR runs the complete e2e suite, just split into up to 10 (or 5, for the
new-tests flake jobs) parallel shards per mode/React-version combination, dozens of matrix jobs total.
This is a valid, explicitly useful finding per the task brief: "they just run everything, sharded N ways"
— confirmed concretely, with the exact sharding mechanism and its timing-based (not diff-based) load
balancer.

## 3. Aggregating gate job

Yes — `tests-pass` (`build_and_test.yml` lines 1232-1276), explicitly named with intent in a code comment
right above it:

```yaml
  # status checks to pass" rule on the canary branch.
  # <https://github.com/vercel/next.js/settings/rules/15507172>
  tests-pass:
    needs:
      # Note: Some of these jobs are dependencies of other test jobs. If they
      # fail, they'll cause the dependent test job to be skipped, which could
      # cause this aggregation job to incorrectly pass if the dependency job is
      # not listed here.
      - optimize-ci
      - changes
      - build-native
      - build-next
      ... [31 jobs total]
    if: always()
    runs-on: ubuntu-latest
    # Coupled with retry logic in retry_test.yml
    name: thank you, next
    steps:
      - run: exit 1
        if: ${{ always() && (contains(needs.*.result, 'failure') || contains(needs.*.result, 'cancelled')) }}
```

Same pattern as the target repo's `gate` job and langchain's `ci_success`: `if: always()` so it runs even
when upstream jobs are conditionally skipped by `docs-only`/`turbopack-only`/`optimize-ci` gating, then
fails only if any dependency actually failed or was cancelled (implicit success — no explicit "echo
pass" step, the job just doesn't `exit 1`). The in-code comment explicitly names the exact GitHub quirk
the earlier research already describes (skipped checks hanging a required-status
rule) and links the actual branch-protection rule it satisfies
(`github.com/vercel/next.js/settings/rules/15507172`) — the most direct "yes, this is deliberately the
required check" evidence found in any of the three repos analyzed so far. Its job name, `thank you,
next`, is **not cosmetic** — it is the literal string the PR-stack gate (Q4) polls for as the signal that
a predecessor PR's CI succeeded. Worth noting as a design choice: a comment admits the `needs:` list is
manually kept in sync and can silently under-include a job, with no tooling catching a missed addition —
the same manual-sync risk `2-per-dir-test-dispatch-research.md` already flags for a hand-maintained
dependency map.

## 4. Evidence of PR-stack handling — a direct, concrete answer, unlike langchain

Unlike langchain (where `1-pr-stack-research.md`'s companion doc found nothing beyond circumstantial
`merge_group` wiring), **Next.js has a dedicated, purpose-built, actively-maintained mechanism for
stacked PRs**, found at `.github/workflows/pr_stack_optimizer.yml` and
`.github/actions/pr-stack-ci-gate/` (`action.yml`, `src/gate.ts` — 485 lines, `src/index.ts`, with its
own `README.md`, `jest.config.cjs`, and test fixtures captured from a real PR, `#99095`).

**What it infers and from what, exactly** (file header comment, `pr_stack_optimizer.yml` lines 1-6):

> "Avoid running the full CI concurrently on every PR in a branch-based stack. A stack is inferred from
> ordinary PR branch relationships (the next PR's base branch is the previous PR's head branch). The
> first three PRs and every top PR run immediately. A middle PR waits until any of its previous three
> PRs has passed `thank you, next`, while a five-hour deadline fails open."

Key mechanics, confirmed in `README.md` and `gate.ts`:

- **No GitHub native stacking API, no Graphite integration** — purely inferred from base/head branch
  name matching across open PRs (`README.md` line 3: "no GitHub stack API or Graphite integration is
  required").
- **Fork PRs always bypass the gate and run full CI immediately** (`pr_stack_optimizer.yml` lines 36-49)
  — "It cannot be part of a same-repository branch stack, so let its full CI run immediately" (comment,
  line 36). This is the same fork-safety posture the target repo's own POC already verified (zero fork
  PRs currently open, per `PLAN.md`), but Next.js encodes it as an explicit, permanent rule rather than a
  point-in-time observation.
- **Tiering within the stack itself:** the first three PRs in a stack and every "leaf" (topmost, nothing
  built on it yet) PR run full CI immediately, no waiting. A PR buried deeper in the stack polls its
  three closest predecessors' `thank you, next` required-check status every 5 minutes, refreshing the
  inferred branch topology every 20 minutes, and **opens (runs full CI) the moment any one of those three
  predecessors passes** — not all three, any one (`gate.ts` — `REQUIRED_CHECK = 'thank you, next'`,
  `gateDecision()` function, line 90). If all three predecessors fail, the gate fails closed. A 5-hour
  wait ceiling exists inside a 6-hour job timeout, so an indefinitely-stuck wait fails open rather than
  hanging forever — directly analogous to the "required check that never runs" trap this repo's own
  `gate` job pattern already solves, but applied to *stack depth* rather than *skipped jobs*.
- **Explicit API-budget engineering**, documented with real numbers in `README.md`'s "API budget and
  correctness" section: 8 REST reads on first poll/every-20-minutes (current PR, successor, 3 predecessor
  links, 3 required checks), 3 reads on each 5-minute interim tick — "51/hour... roughly 47% fewer"
  normal pending reads than a naive 5-minute-interval full-topology refresh (96/hour), against GitHub's
  "typically 1,000/hour/repository" `GITHUB_TOKEN` budget.
- **A bypass label** (`BYPASS_LABEL: CI Bypass PR Stack Optimization`, `pr_stack_optimizer.yml` line 19)
  lets a human override the wait manually.

**Net for Q4 — this is the single most directly relevant finding in this whole analysis pass for the
senior's original PR-stack question.** `1-pr-stack-research.md` already correctly concluded that GitHub's
*default*, tool-free behavior gives each stacked PR a base-relative diff "for free," and that the
remaining gap is wasted redundant full-CI runs down a stack plus unverified cross-layer integration. Next.js
has shipped a real, working, from-scratch (not Graphite, not a marketplace action) answer to exactly the
first half of that gap — "how do you avoid paying full CI cost at every layer of a stack" — by **delaying
expensive CI** at non-leaf, non-early-stack PRs until a nearby predecessor already proved green, rather
than skipping CI outright or trying to compute a cumulative diff. It does **not** address the second half
(stack drift against a moving `main`, or whether layer N's code genuinely integrates with layer N-1's) —
nothing in `gate.ts` touches that; it only decides *when* to run full CI, not *what* to diff.

## 5. Genuinely novel / different from both the target repo and `langchain-analysis.md`

1. **A real, working, home-grown stacked-PR optimizer — the thing both prior v2 docs said nobody in this
   research pass had found evidence of.** This is the standout finding of this whole analysis. It proves
   the general pattern `1-pr-stack-research.md` described abstractly from Graphite's docs ("tiered, not
   cumulative; delay expensive CI until a predecessor is proven green") is not just vendor marketing — a
   major OSS project with 27-PR-deep-equivalent stacking pressure built exactly that, without adopting
   Graphite or git-spice, using nothing but the GitHub REST API and branch-name inference. Directly
   actionable precedent if this repo ever wants to stop paying full-gate cost at every one of its 8-deep
   stack layers.
2. **The aggregating gate job's name is load-bearing elsewhere in the same CI system.** `thank you, next`
   isn't just a required check — the PR-stack gate polls GitHub for exactly that check-run name on
   predecessor PRs to decide when to open. This is a subtle but real coupling: the two mechanisms
   (aggregating gate + stack gate) only work together because they agree on one shared string. Worth
   naming for this repo if a similar stack-gate is ever built on top of its existing `gate` job — the
   dependency would be the same shape.
3. **Timing-based load-balanced sharding, backed by a persistent store (Vercel KV), as the actual answer
   to "suite too big."** Neither the target repo's POC nor langchain's CI do anything like this — langchain
   tiers by *moving slow tests off the PR path entirely* (daily cron); Next.js instead keeps the full
   suite on every PR but makes the 10-way split as evenly timed as possible using real historical
   per-file timing data, with an explicit `--require-timings` strict mode and a round-robin fallback. This
   is a distinct strategy from both "scope down" (dependency-graph affected-detection) and "move off the
   PR path" (cron) — it's "run everything, but balance the cost." Relevant context for this repo's own
   e2e/Playwright scoping question: Next.js's answer is evidence that at least one large, resourced OSS
   project judged smart *selection* of e2e tests not worth building, and chose smart *distribution*
   instead — a legitimate third option alongside the two named in `2-per-dir-test-dispatch-research.md`.
4. **Coarse category-gating (`docs-only`, `turbopack-only`) as a cheap, separate layer in front of the
   expensive matrix, distinct from per-test selection.** This is a smaller-scale, binary version of what
   this repo's `detect-services.py` already does continuously (it already picks per-service, not just
   on/off) — so it's not a gap for this repo, but it does confirm the general industry pattern (coarse
   gating in front of expensive matrices) is used even by a project that also does fine-grained sharding
   elsewhere.
5. **Turborepo's own dependency graph exists in this repo but isn't used for PR-gating dispatch.** Worth
   stating plainly since it was the first thing this analysis checked for: a Vercel-owned project, with a
   real `turbo.json` task graph, does **not** use `turbo run --affected`/`--filter` to decide what to
   test on a PR. This is a negative but useful finding — it means "adopt Turborepo" cannot be assumed to
   automatically close this repo's dependents-gap (see `2-per-dir-test-dispatch-research.md`) just because
   Turborepo supports `--affected` in theory; even Turborepo's own creator's flagship project solves "what
   changed" with a hand-written script instead, same as langchain and this repo's own POC.

No claim is made here about *why* Next.js chose any of these designs beyond what's in the code's own
comments — where a comment states a rationale (e.g. "It cannot be part of a same-repository branch stack,
so let its full CI run immediately," "as tests are intentionally run multiple times to detect flakes"),
it's quoted directly above; where no comment exists, no rationale is invented.

## What this means for the target repo's gate, concretely (not yet implemented)

- **Q1 (dependency-graph-awareness):** Next.js does not give evidence that adopting Turborepo would
  automatically solve the `conflict-types`/`calculated-fields`/`mcp-auth-core` dependents-gap "for free" —
  it has Turborepo and still uses a custom path-matching script for PR-gating decisions, same category of
  mechanism as langchain's `check_diff.py` and this repo's `detect-services.py`. The two options already
  named in `2-per-dir-test-dispatch-research.md` (adopt a graph-aware tool, or hand/parse a small
  dependency map) remain the live options; langchain's parsed-manifest approach is still the cheaper
  precedent of the two case studies checked so far.
- **Q2 (e2e scoping):** Next.js adds a concrete, validated third pattern to the two already documented in
  `2-per-dir-test-dispatch-research.md` (Playwright `--only-changed`, tag-based `@smoke` tiering): **shard
  the full suite and balance shards by historical timing**, rather than trying to select a subset. This is
  directly relevant precedent if/when this repo's own excluded Playwright/e2e suite is revisited — "just
  run everything, sharded, load-balanced by past duration" is a proven-at-scale alternative to building
  diff-based test selection, with a known cost (every PR always pays for the full suite, just spread
  across more runners) rather than a correctness risk (selection logic missing a dependent test).
- **Q4 (PR stacks):** this is the first and only concrete, real-world "ship a stack-aware CI gate without
  adopting Graphite" precedent found across all OSS case studies in this research pass. If this repo
  revisits its own 27-of-49-stacked-PRs redundant-CI problem (flagged but left unsolved in
  `1-pr-stack-research.md`), `pr_stack_optimizer.yml` + `.github/actions/pr-stack-ci-gate/` is a direct
  architecture to study line-by-line before building anything from scratch.

## Sources

- `turbo.json` (full file read)
- `.github/workflows/build_and_test.yml` (full file read, 1276 lines)
- `.github/workflows/build_reusable.yml` (grep + targeted reads)
- `.github/workflows/pr_stack_optimizer.yml` (full file read, 68 lines)
- `.github/actions/pr-stack-ci-gate/action.yml`, `README.md` (full files read)
- `.github/actions/pr-stack-ci-gate/src/gate.ts` (grep over all 485 lines for function/constant names;
  targeted reads of `gateDecision`, `REQUIRED_CHECK`, `discoverTopology`)
- `scripts/run-for-change.mjs` (targeted read, `CHANGE_ITEM_GROUPS` definition)
- `run-tests.js` (1164 lines; grep + targeted reads of sharding/timing logic, lines ~160-630, ~1089-1140)
- `scripts/test-new-tests.mjs` (grep for diff/changed-file logic — confirmed it is a flake-detection
  runner invoked with an explicit `--group`, not inspected line-by-line for its own changed-file
  heuristic; flagged as not fully read, unlike the files above)
- `../langchain-analysis.md` (read first per instructions, to avoid duplicating its findings)
- `../../1-pr-stack-research.md`, `../../2-per-dir-test-dispatch-research.md`,
  `../../PLAN.md` (read first per task instructions)
