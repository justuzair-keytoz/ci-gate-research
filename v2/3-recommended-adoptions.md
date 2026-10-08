# v2 Research → What We'd Actually Pick

Status: a recommendation, not yet implemented. Scoped specifically to "improve `pr-gate.yml` /
`detect-services.py` as they exist today" — not "adopt a new build system or tool." Everything below
traces back to a specific finding in `1-pr-stack-research.md`, `2-per-dir-test-dispatch-research.md`, or
one of the `oss/*.md` case studies; nothing here is invented new.

## Pick now — cheap, high value, fits the current architecture exactly

| # | Idea | Source | How it works there | What we'd adapt for this repo | Effort |
|---|---|---|---|---|---|
| 1 | **Exclusion-based default, not inclusion-based** | `oss/calcom-analysis.md` §2 | `dorny/paths-filter` matches `'**'` (everything) by default, then explicitly excludes a short list of known-safe non-code paths (`.vscode/**`, `**/*.md`, `docs/**`, etc.) | `detect-services.py` currently does the opposite — a file only triggers a service if it matches one of 23 hardcoded path prefixes; anything unmatched (including a brand-new service folder) is silently skipped. Add a fallback: if a changed file doesn't match *any* known service path and isn't in a small safe-exclusion list (docs, `README.md`, `.github/*.md`), treat it as "unknown — run the full matrix," reusing the same full-matrix fallback already proven for root-file changes | Small — one new branch in the existing path-matching loop |
| 2 | **Parse real manifests into a dependency graph, don't hand-maintain one** | `oss/langchain-analysis.md` §2 | `dependents_graph()` reads every package's own `pyproject.toml` `[project.dependencies]` at CI time and builds a package→consumer map from the actual source of truth, no new tool, nothing to go stale | We already proved the gap: `packages/conflict-types` has 4 real consumers (`Temporal`, `UI`, `ruleengine/rules-service`, `Strapi`) that current detection misses. Read each shared `packages/*`'s consumers straight out of every workspace's `package.json` `dependencies` field (data that already exists) and expand the selected-services list to include them | Medium — one new function, ~30-40 lines, in `detect-services.py` |
| 2a | **It's OK to hand-special-case your one worst-offender package** | `oss/langchain-analysis.md` §2 | Even langchain doesn't trust their generic dependents graph for `libs/core` (highest fan-out) — they hand-code a separate ordered cascade for it instead, with a comment admitting "core has so many dependents" | If one of our 5 shared packages turns out to have too many consumers to expand cleanly via #2's generic logic, don't force it — hardcode that one package's consumer list directly, same as langchain does for `libs/core` | Small — fallback plan for #2, not separate work |
| 3 | **Consolidate repeated install steps into one composite action** | `oss/langchain-analysis.md` §7 (the follow-up addendum) | `.github/actions/uv_setup` wraps "install Python + package manager + configure cache" in one place, called identically from every workflow (`_lint.yml`, `_test.yml`, `check_extras_sync.yml`, etc.) | `fast-checks` currently repeats corepack-enable + install logic inline, branching for standalone vs. workspace services. Pull this into a local composite action (e.g. `.github/actions/yarn-setup`) called once per job instead of duplicated | Small — pure refactor, no new behavior |

## Worth doing soon — slightly bigger, still shaped like a POC improvement

| # | Idea | Source | How it works there | What we'd adapt for this repo | Effort |
|---|---|---|---|---|---|
| 4 | **Skip the expensive tier while a PR is in draft** | `oss/vault-analysis.md` §3, Axis 1 | `needs.setup.outputs.is-draft == 'false'` gates the race-detector job; it auto-enables the moment a PR is marked ready for review (the workflow's trigger block explicitly adds `ready_for_review` as a type for this reason) | Add `if: github.event.pull_request.draft == false` to the `docker-build` job. Cheap now; gets genuinely valuable once the ECR-credential blocker is resolved and the full 18-service Docker matrix is live (that's the tier actually worth gating this way) | Small — one `if:` condition |
| 5 | **A small, repo-wide "manifest drift" check** | `oss/langchain-analysis.md` §7 (`check_extras_sync.yml`) | A dedicated workflow, triggered only when manifest files change, iterates every package's `pyproject.toml` and fails with an exact list of whichever manifest(s) are internally inconsistent | We already hit this exact bug class live — `StrapiMCP`'s `yarn.lock` silently didn't match its own `package.json`, caught only because the POC happened to touch it. Add a small, separate, path-filtered job that checks every workspace's lockfile/manifest consistency on its own, independent of whether that specific service's directory was touched this PR | Medium — new small workflow, reuses logic `yarn install --immutable` already does per-service, just surfaced repo-wide |

## Correctly out of scope — real investments, not polish on the existing POC

| # | Idea | Source | Why it's a bigger decision, not a POC tweak |
|---|---|---|---|
| 6 | Home-grown PR-stack CI delay mechanism | `oss/nextjs-analysis.md` §4 (`pr_stack_optimizer.yml`) | ~500 lines of custom TypeScript action with GitHub API polling logic, not a config change. The best answer to the 8-deep-stack redundant-CI cost, but a standalone build, not something you bolt onto `pr-gate.yml` in an afternoon |
| 7 | Adopting Graphite | `oss/calcom-analysis.md` §3 | A team git-workflow and tooling decision affecting everyone's day-to-day, not something that touches `pr-gate.yml` at all |
| 8 | Three-way trust/aggregating split, fork-PR trust logic | `oss/vault-analysis.md` §4, `oss/calcom-analysis.md` §1/§3 | Solves a problem this repo doesn't have yet — zero fork PRs confirmed currently open. Revisit if that changes |
| 9 | E2E tiering (cron-only, timing-based sharding) | `oss/langchain-analysis.md` §3, `oss/nextjs-analysis.md` §2 | Not applicable yet — Playwright/e2e isn't in the gate at all, by the original brief's own scope. These are the right patterns to come back to *when* that's revisited, not before |

## If you want these implemented

Picks #1-#3 are the ones I'd actually do first — they're small, each traces to a concretely demonstrated
gap (not a hypothetical), and none of them require a decision from anyone beyond "yes, make this change."
#4-#5 are good next, slightly bigger. #6-#9 need an explicit decision first (worth it? who owns it?)
before any code gets written.
