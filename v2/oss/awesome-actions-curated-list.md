# Curated List Check: `sdras/awesome-actions`

Status: research only, no decisions made, no code changed.

## What this is, and why it's handled differently from the other `v2/oss/` docs

`sdras/awesome-actions` isn't a working CI setup to analyze line-by-line like `langchain-analysis.md` —
it's a curated README of links to third-party GitHub Actions, grouped by category. So this doc is a
different shape: what's actually in the list, what's genuinely useful for the two senior-review
questions, and — just as importantly — what's conspicuously *not* in it, which turned out to be the more
interesting finding.

Checked by fetching the raw README directly (`raw.githubusercontent.com/sdras/awesome-actions/master/README.md`,
572 lines) and grepping it for everything relevant, rather than trusting a summarized read — so every
claim below is a direct line match, not a paraphrase.

## What's actually relevant

**Change detection / path filtering** (`README.md` line 211, 231):
- [`dorny/paths-filter`](https://github.com/dorny/paths-filter) — "Conditionally run actions based on
  files modified by PR, feature branch or pushed commits." This is the same action the *first* pass of
  this repo's own research (`2. Ci-Gate-Research.md`, written before repo access existed) considered and
  deliberately rejected in favor of a hand-written Python/bash script, partly over a low third-party
  supply-chain score. Its presence here is just confirmation it's a well-known, commonly-recommended
  action — not new information.
- [`MarceloPrado/has-changed-path`](https://github.com/MarceloPrado/has-changed-path) — same category,
  a smaller/less common alternative. Not evaluated further; no evidence it does anything `paths-filter`
  doesn't.

**Docker layer caching** (lines 467-468):
- [`whoan/docker-build-with-cache-action`](https://github.com/whoan/docker-build-with-cache-action) and
  [`crazy-max/ghaction-docker-buildx`](https://github.com/crazy-max/ghaction-docker-buildx) — both listed,
  **neither is what this repo's own `pr-gate.yml` actually uses.** The target repo uses the official
  `docker/build-push-action` + `docker/setup-buildx-action` (Docker-maintained, not community), which
  **do not appear anywhere in this list at all** (confirmed: zero matches for "build-push-action" or
  "setup-buildx-action" in the full 572-line file). This is the most concrete finding from this doc: the
  list predates the now-standard official Docker actions and only points to older community
  alternatives — a sign of how old this list is, not a reason to switch away from what's already working.

**A PR-merge-blocking action adjacent to the stacking question** (line 370):
- [`cirrus-actions/branch-guard`](https://github.com/cirrus-actions/branch-guard) — "Block PR merges when
  Checks for target branches are failing." Relevant because it's explicitly about a PR's mergeability
  depending on its *target branch's* check status — conceptually adjacent to the stacked-PR question of
  "should a child PR's merge be blocked if its parent branch's own checks are failing," though this
  specific action weirdly phrases it as "target branch," not "base PR," and wasn't tested against an
  actual stacked-PR setup by its own README (not verified further — flagging the concept match, not
  endorsing the specific action).

**Monorepo, in exactly one place** (line 152):
- [`olivr/copybara-action`](https://github.com/olivr/copybara-action) — "Move and transform code between
  repositories (ideal to maintain several repos from one monorepo)." This is about *splitting* a monorepo
  out into multiple separate repos (Google's Copybara tool, repurposed as an Action), which is a
  different problem than this project's "test only what changed within one monorepo." Not applicable.

## What's conspicuously absent

This is the real finding. Searched the full list for every term this research cares about:

- **No `turbo`, no `nx`, no mention of Turborepo or Nx anywhere in the file.** Zero hits. The two
  dependency-graph-aware monorepo tools that `2. Ci-Gate-Research.md`, `2-per-dir-test-dispatch-research.md`,
  and both the langchain/Next.js/Vault analyses in this same folder all discuss at length are entirely
  absent from this list.
- **No dependency-graph-aware "affected" detection concept at all** — the list's only changed-file
  tooling is plain path-filtering (`paths-filter`, `has-changed-path`), the same category of mechanism
  this repo's own POC already uses and has already found to be insufficient on its own (the
  `conflict-types`/`calculated-fields`/`mcp-auth-core` dependents gap).
- **No stacked-PR tooling** — no Graphite, no git-spice, no ghstack, no mention of "stacked" or "stack"
  anywhere in the file.
- **No aggregating-gate-job pattern, no required-status-check guidance** — nothing matching the
  `if: always()` / `needs.*.result` pattern this repo's `gate` job (and langchain's `ci_success` job)
  both use.

## Honest conclusion

This list is a reasonable *beginner's index* of "here are some third-party Actions that exist," useful
for discovering individual point-solutions (a labeler, a Slack notifier, a cache action) — but it has
**no coverage at all** of the specific, harder problems this research is actually working through
(monorepo dependency graphs, PR stacking, tiered CI, aggregating required checks). Every substantive
finding in this `v2/oss/` folder so far has come from reading real, working CI configs in large active
projects (langchain, and pending: Next.js, Vault) — not from a curated links list. Worth knowing before
pointing anyone else at this list expecting it to answer these two questions; it won't.

## Sources

- [`sdras/awesome-actions` README, raw, as fetched for this analysis](https://raw.githubusercontent.com/sdras/awesome-actions/master/README.md)
  — all line numbers above refer to this exact fetch.
