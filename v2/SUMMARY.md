# v2 Research — Executive Summary

Each entry: what the doc actually found, the mechanism (briefly), and which of your senior's two concerns
it speaks to. **[Stack]** = PR-stack testing scope / end-to-end stack verification. **[Dispatch]** =
per-directory test dispatch, avoiding a global script, dependency-aware selection. Full detail and exact
citations are in each linked doc.

## Our own research (not OSS)

**`1-pr-stack-research.md`** — **[Stack]**
Checked this repo's real, currently-open PRs via `gh pr list`. Found 27 of 49 target a base branch other
than `main` — this repo is already deep into manual PR stacking, with two chains 7-8 layers deep
(`DATA-261`, `DATA-345` series). Then checked what GitHub itself does by default: a stacked PR's diff is
already scoped to its immediate parent, not cumulative against `main` — that part needs no fix. The gap
industry research converges on: running the *same* full gate at every layer of a deep stack is wasteful,
since each layer's own diff is small. The fix everyone recommends is tiering — cheap checks (lint, build,
unit tests) on every layer, expensive checks (e2e, full suites) once per stack rather than N times. A
second, harder gap this doc names but doesn't solve: "stack drift" — if `main` moves while a stack sits
open, each layer is still diffed against its *original* parent, not current `main`, so the stack can
still go stale. The standard fix for that (a merge queue) is already out of scope per the original brief.

**`2-per-dir-test-dispatch-research.md`** — **[Dispatch]**
Confirmed the existing POC already does the hard part right: it only runs build/lint/test for services a
PR actually touched, never a global script. The gap is narrower and more specific than "no global
script" — it's that path-matching alone can't see who *depends on* a changed directory. Proved this
concretely, not hypothetically: grepped every `package.json` in the repo for `"@databonder/conflict-types"`
and found 4 real consumers (`Temporal`, `UI`, `ruleengine/rules-service`, `Strapi`) that the current
`detect-services.py` would silently skip if only `conflict-types` itself changed. Same gap confirmed for
`calculated-fields` and `mcp-auth-core`. Two ways to close it are named (adopt Nx/Turborepo, or
hand-maintain a small dependency map) — neither implemented yet.

## OSS case studies

**`oss/langchain-analysis.md`** — mostly **[Dispatch]**, some **[Stack]**
The strongest *dependents* answer found anywhere in this pass. A ~50-line Python function
(`dependents_graph()`) reads every package's own `pyproject.toml` dependency declarations at CI time and
builds a real package→consumer map from the source of truth itself — no new build system, nothing to go
stale, cheaper than both options named in our own research doc. Caveat worth knowing: even langchain
doesn't trust this generically for their single highest-fan-out package (`libs/core`) — they explicitly
hand-code a separate cascade for it instead, with a comment admitting "core has so many dependents" that
auto-expanding would be unwieldy. So "parse the graph generically, but hand-special-case your one
worst-offender package" is apparently normal, not a shortcut to be ashamed of. Also has the clearest
tiering example seen: live-credential, real-API integration tests aren't a slow PR job — they're a wholly
separate workflow that only runs on a daily cron, never on `pull_request` at all. On stacking: nothing
found directly, but their CI listens for GitHub's `merge_group` event, meaning the plumbing for a merge
queue (the thing that actually fixes "stack drift") already exists, even if not confirmed to be switched
on.

**`oss/nextjs-analysis.md`** — mostly **[Stack]**, some **[Dispatch]**
The single best finding across all five case studies. Next.js has a real, actively maintained,
purpose-built system for exactly this problem: `.github/workflows/pr_stack_optimizer.yml` plus a custom
TypeScript action. It infers a stack purely from ordinary branch base/head relationships (no Graphite, no
GitHub stacking API) — the same way this repo's own stacks already work. The first three PRs in a stack,
and every "leaf" PR (nothing built on top of it yet), run full CI immediately with no waiting. Anything
deeper polls every 5 minutes and only opens up (runs full CI) once *any one* of its three closest
predecessor PRs has already passed the same required check — with a 5-hour ceiling so a stuck wait fails
open instead of hanging forever. Fork PRs always bypass the whole mechanism, since they can't be part of
a same-repo stack. This is a direct, copyable architecture if the 8-deep-stack redundant-CI cost is ever
worth fixing. On dispatch: despite being Vercel's own flagship project, with a real `turbo.json`
dependency graph, their actual PR-gating logic never calls `turbo run --affected` — it's a hand-written
path-matching script, same category as everyone else's. Useful negative evidence: adopting Turborepo
would *not* automatically close this repo's dependents gap, even for the company that makes Turborepo.
Their e2e answer, separately: don't try to select a subset — run the entire suite, split 10 ways, with
shard assignment balanced by real historical per-file timing data rather than git diff.

**`oss/vault-analysis.md`** — mostly **[Dispatch]**, some **[Stack]**
The opposite philosophy from langchain on dependents: Vault doesn't try to track them at all. Once any
Go code changes, it tests the *entire* affected Go module in full, every time — no package-level
selection, leaning on Go's own build cache and 16 parallel runners to keep that affordable. This is a
legitimate strategy specifically because Go's toolchain makes "test everything in scope" cheap in a way
this repo's Node/Yarn setup generally isn't — not directly transferable, but useful as the "valid
alternative for the right language" data point. The most directly reusable idea: an explicit,
code-enforced tier that skips the expensive data-race-detector test entirely while a PR is still in draft
status, auto-enabling the moment it's marked ready for review — a cheap, already-proven pattern for
eventually gating Playwright the same way. Also has a "runs, but doesn't block merge" tier (FIPS
compliance and race-detector results are explicitly excluded from the required-check calculation) — a
third state beyond simple required/not-required. On stacking: no PR-stack tool found, but there's a
structurally similar automation — a bot auto-opens a backport PR against every active release branch the
moment a PR merges to `main`, so a change propagates and gets independently re-verified across a tree of
long-lived branches without a human doing it by hand. Different problem (release branches, not unmerged
feature PRs) but the same underlying shape.

**`oss/calcom-analysis.md`** — strong findings on **both**
Two separate, concrete, high-value findings. First, on dispatch: cal.com's change-detection defaults the
*opposite* direction from this repo's own `detect-services.py`. Theirs matches **everything** by default
and explicitly excludes a short list of known-safe non-code paths (docs, markdown, `.vscode`); ours
matches **nothing** by default and only tests a service if it's explicitly listed in a hardcoded table of
23 known services. Concretely: under this repo's current design, a brand-new service folder added
tomorrow gets **zero gate coverage** until someone remembers to add it to that table. None of the other
three OSS repos exposed this, because all three default the same risky direction this repo does — cal.com
is the first one that doesn't, and it's arguably the single most actionable, lowest-cost fix in this whole
research pass. Second, on stacking: cal.com's CI trust logic has a hardcoded, named allowance for
`graphite-app[bot]` to add CI-triggering labels — direct, concrete proof that a real company runs
Graphite in production and had to deliberately decide to trust its automation. First actual sighting of a
stacking tool in the wild in this whole research pass, versus just reading about it in Graphite's own
docs. Smaller bonus finding: their merge-queue-triggered gate (`all-checks.yml`) explicitly treats a
*skipped* job as a failure — the opposite of their own PR-time gate — confirming the right general
pattern is "skipping is fine on a PR, not fine right before something actually merges."

**`oss/awesome-actions-curated-list.md`** — answers **neither**
Checked this thoroughly rather than assuming. Grepped the full curated README for every relevant term:
zero mentions of Turborepo or Nx anywhere, no stacking tools (no Graphite, no git-spice, no "stack"
anywhere in the file), and no dependency-graph-aware detection concept at all — its only change-detection
entries are plain path-filtering actions (`dorny/paths-filter` and a lesser-known equivalent), the same
category this repo's POC already uses and has already found insufficient on its own. It's a reasonable
beginner's index for discovering simple point-solution actions (a PR labeler, a cache action) but has no
coverage of either of your senior's actual questions. Don't expect it to answer either one if you do skim
it yourself.

## If you only act on three things from all of this

1. **Fix the detection default-direction** (from cal.com) — this is a real bug class in the current POC,
   not a hypothetical: new services get silently zero coverage. Worth fixing before the dependents
   question, since it's cheaper and more dangerous if missed.
2. **Decide whether the 8-deep-stack redundant-CI problem is worth solving now** — if yes, Next.js's
   `pr_stack_optimizer.yml` is a concrete architecture to copy from; if Graphite itself is ever considered
   instead, cal.com is real evidence it works in production.
3. **Close the shared-package dependents gap** using langchain's cheaper "parse existing manifests"
   approach, rather than adopting Nx/Turborepo (which, per both Next.js and cal.com, wouldn't solve it
   automatically even if adopted).
