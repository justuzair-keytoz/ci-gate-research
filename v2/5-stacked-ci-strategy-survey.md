# v2 Research: A Wider Survey of Stacked-PR CI Cost Strategies

Status: research only, no decisions made, no code changed anywhere (`tmp/*` clones untouched,
`detect-services.py`/`pr-gate.yml` in the target repo untouched). Read `1-pr-stack-research.md`,
`oss/nextjs-analysis.md` §4, and `oss/calcom-analysis.md` §3 first — this doc
exists because those four sources, while real, are a narrow base (one concrete mechanism —
predecessor-polling — plus one confirmed-but-opaque vendor usage). This doc adds independent evidence
before anything gets designed around just that one example.

## What was inspected

Shallow-cloned to `tmp/` (all read-only, nothing modified):

- `tmp/ghstack` — `ezyang/ghstack`, PyTorch's stacked-diff submission tool (commit-per-PR, synthetic
  branches, not base-branch stacking)
- `tmp/spr` — `ejoffe/spr`, a from-scratch Go stacked-PR tool (real base-branch stacking)
- `tmp/git-spice` — `abhinav/git-spice`, actually cloned this time (prior research only read its docs
  site, not the repo)
- Meta's `facebook/sapling`: **not cloned** — `curl https://api.github.com/repos/facebook/sapling` reports
  `"size": 536579` (≈524 MB), too large for a useful shallow clone here. Checked via GitHub API instead:
  `curl https://api.github.com/repos/facebook/sapling/contents/.github/workflows` lists only
  infra/release/mononoke/edenfs workflow files (`edenfs_linux.yml`, `mononoke_linux.yml`,
  `sapling-cli-*-release.yml`, etc.) — nothing named anything like a PR-stack gate. No further sapling
  evidence found; not fabricating any. This is a **negative result**, not a skipped source.
- Three real GitHub PRs/issues found via web search, each inspected directly with `gh pr diff` / `gh pr
  view` / `gh issue view` (not summarized from a blog, the actual diffs and bodies, quoted below)

## Finding 1 — none of the three cloned stacking tools build stack-aware CI for their own repos

This is the single most important finding of this pass, because all three are projects whose maintainers
stack PRs constantly (building a stacking tool implies dogfooding one), yet none of their own CI treats
a stack specially:

- **`tmp/ghstack/.github/workflows/test.yml`**: `on: pull_request` / `on: push: branches: [main]`, full
  `python-version` × `os` matrix (5 × 3 = 15 jobs), no stack-detection logic anywhere in the file.
  `tmp/ghstack/.github/workflows/lint.yml` — same, single job, no stack logic.
- **`tmp/spr/.github/workflows/ci.yml`**: `on: [push, pull_request, workflow_dispatch]`, one `build` job,
  full `go build` + `go test -race`, no stack-detection logic.
- **`tmp/git-spice/.github/workflows/ci.yml`**: `on: push: branches: [main]` / `pull_request` /
  `workflow_dispatch`, full lint + generated test matrix (`test-matrix` job generates a matrix via
  `go run ./tools/ci/test-matrix`, then `test` runs it in full), no stack-detection logic. Its aggregating
  `ok` job (lines ~113-124) is the exact same `if: always()` + "fail if any dependency didn't succeed"
  shape as this repo's own `gate` job — explicitly commented `# Workaround for GitHub marking this job as
  skipped, and allowing a bad PR to merge anyway.`

None of these three skip, delay, or tier CI based on stack position. **Evidence tier: code** — this is
the actual, currently-live CI config of each project, not a description of intent.

What *is* stack-aware in all three is **merge-time gating, not CI-trigger-time gating** — each tool
refuses to consider a layer mergeable until that layer's own CI has independently passed:

- `tmp/spr/github/pullrequest.go` lines 61-76, `Mergeable()`:
  ```go
  func (pr *PullRequest) Mergeable(config *config.Config) bool {
      if !pr.MergeStatus.NoConflicts { return false }
      if !pr.MergeStatus.Stacked { return false }
      if config.Repo.RequireChecks && pr.MergeStatus.ChecksPass != CheckStatusPass { return false }
      if config.Repo.RequireApproval && !pr.MergeStatus.ReviewApproved { return false }
      return true
  }
  ```
  `tmp/spr/spr/spr.go` lines 457-468, `MergePullRequests`: walks the stack bottom-up, stopping at the
  first PR that `!pr.Mergeable(...)` — i.e. **every** layer up to the merge point must individually show
  `ChecksPass == CheckStatusPass`. There is no "trust the top PR's CI for everyone beneath it" logic
  anywhere in this file.
- `tmp/git-spice/doc/src/guide/merge.md` lines 106-121: "Before git-spice merges, it waits for the CR to
  be ready... git-spice will poll the forge until the CR is reported as mergeable... If a CR is not ready
  to merge after the configured timeout, that CR is assumed to be blocked and it, and its upstack branches,
  will not be merged." This is per-CR, bottom-up, same shape as spr.
- `tmp/ghstack/src/ghstack/land.py` (231 lines, read in full) is different again: `ghstack land <PR>`
  cherry-picks **every** commit in the stack (lines 114-119, 149-183) and pushes them all directly onto
  `main` in one `git push` (line 188-190) — bypassing GitHub's merge button entirely. Each commit's PR
  already ran its own full CI independently when pushed (per `test.yml` above, against a synthetic
  `gh/user/N/base` branch, not against current `main`). Landing does **not** re-run CI against current
  `main` before the direct push — only a `--force-with-lease` conflict check. This is the closest thing
  found in any of these three tools to "trust prior per-layer CI, no final re-validation," and it is a real,
  named risk in `ghstack`'s own code comment (`land.py` lines 196-201): "It might be helpful to advance
  orig to reflect the true state of upstream at the time we are doing the land... as opposed to this
  synthetic thing I'm doing right now."

## Finding 2 — git-spice's own repo, re-checked directly (not just docs this time)

`1-pr-stack-research.md` cited git-spice's docs site but never cloned the actual repo. Re-checked now:
`tmp/git-spice/.github/workflows/` has 8 files (`ci.yml`, `autofix.yml`, `changelog-merge.yml`,
`changelog-check.yml`, `doc.yml`, `prepare-release.yml`, `publish-release.yml`, `sponsors.yml`) — none
stack-aware (confirmed above for `ci.yml`, the only one that runs tests). `tmp/git-spice/doc/src/guide/`
has a `merge.md` guide (quoted above) but no CI-recommendation doc for *users'* CI pipelines at all —
git-spice's merge-readiness polling is forge-agnostic (`spice.merge.ready.command`, any exit-code-driven
script) and explicitly does not prescribe a CI architecture; it only waits on whatever the forge already
reports. **Net: git-spice, inspected directly, adds no new stack-CI-cost mechanism beyond what
`1-pr-stack-research.md` already found from its docs** — this re-check is a confirmed negative result,
not a new finding, but it closes the "didn't actually clone it" gap the task named.

## Finding 3 — leaf-only and bottom+top strategies, found as real shipped code elsewhere

Two real PRs turned up via web search, each fetched and read in full via `gh pr diff`/`gh pr view`
(exact diffs, not paraphrased):

### Leaf-only: `pomerium/agentops` PR #10, "ci: run the checks only at the top of a PR stack"

`.github/scripts/stack-top.sh` (24 lines, quoted in full from the diff):
```bash
if [[ "${EVENT_NAME}" != "pull_request" ]]; then emit true; exit 0; fi
if [[ "${HEAD_REPO}" != "${REPO}" ]]; then emit true; exit 0; fi
above=$(gh pr list --repo "$REPO" --base "$HEAD_REF" --state open --json number \
  | jq -r --argjson self "$PR_NUMBER" 'map(select(.number != $self) | "#\(.number)") | join(" ")')
if [[ -n "$above" ]]; then emit false; else emit true; fi
```
Wired into `docker.yaml`/`helm.yaml`/(presumably) `test.yaml` via a reusable `stack-top.yaml` workflow;
every expensive job gets `needs: stack if: needs.stack.outputs.top == 'true'`. The PR body (quoted
directly) states the mechanism and names the exact risk: **"Worth a close look: GitHub reports a skipped
job as passing for required checks. A lower PR in a stack can therefore merge on the evidence of the top
PR's run, not its own."** Evidence tier: **code**, plus the author's own stated risk assessment.

### Bottom+top hybrid: `metabase/metabase` PR #83715, "Run CI on the top of a PR stack, not just the base"

PR body (quoted directly): "Today the test gate only lets the lowest unmerged PR of a stack run CI. This
also lets the top of the stack run, so the combined change gets tested before anything merges. PRs in the
middle stay force-skipped unless they carry `ci:run-all`." Verdict table from the PR body:

| PR in stack | Verdict |
|---|---|
| Bottom (base ref is the stack base) | `defer` (normal path-filter logic decides) |
| Top (`position == size` on GitHub's native stack payload) | `defer` |
| Middle | `force-skip` |
| Middle with `ci:run-all` label | `force-run` |

This uses **GitHub's own native stacked-PR payload fields** (`github.event.pull_request.stack.position`
and `.size`) — confirming GitHub ships first-party metadata specifically to support this pattern. Evidence
tier: **code** (merged-looking diff against `.github/actions/create-test-plan/action.yml` and
`.github/file-paths.yaml`), a second independent real-world implementation of "run full CI only at the
two ends of a stack, skip the middle."

### Bottom+top as a named, cited open request: `yugabyte/yugabyte-db` issue #34607

Not implemented code — an open feature request, quoted directly: "GitHub's 'Optimizing CI for stacked
pull requests' guidance is to run expensive jobs only where they are needed, using
`github.event.pull_request.stack`... Build the lowest unmerged PR of a stack... Build the top PR... Run
`NO-BLD` for the layers between them." The issue **explicitly names the same risk pomerium's PR body
names**, independently: "Trade-off: if several layers merge in one operation, the intermediate commits on
`master` were not built on their own. Only the combined result was." Evidence tier: **official-docs-cited
practitioner account** (cites GitHub's own documented guidance, not yet shipped in yugabyte's repo —
labeled accordingly, not claimed as code).

## Comparison table

| Strategy | Evidence tier (best found) | Implementation complexity | Failure mode if predecessor/leaf never goes green | Does an interior layer ever merge without full CI run against *it specifically*? |
|---|---|---|---|---|
| **Predecessor-polling** (Next.js `pr_stack_optimizer.yml`) | **Code** — `.github/actions/pr-stack-ci-gate/src/gate.ts`, 485 lines, own tests, own README (see `oss/nextjs-analysis.md` §4) | High — custom topology inference from branch names, 5-min poll loop, 20-min topology refresh, 5-hour fail-open ceiling inside a 6-hour job timeout, explicit API-budget engineering (51 req/hr vs. naive 96/hr), a bypass label | Fails open after 5 hours (runs full CI anyway) — bounded, but a genuinely broken predecessor silently delays every PR above it for up to 5 hours before anyone notices | **No** — every PR still gets its own full CI eventually; this strategy only delays *when* it starts, never skips it |
| **Leaf-only-full-CI** (pomerium/agentops PR #10; also the top-half of metabase's hybrid) | **Code** — shipped, ~24-line bash script + reusable workflow, author's own PR body names the risk | Low — one script, one reusable workflow, no polling, no external state | No failure mode to "never going green" (nothing waits) — but if the top PR is retargeted/closed without the next-highest PR re-triggering, **nothing re-runs full CI at all** until someone notices (a gap metabase's own PR search results flagged independently: "if a child PR closes or is retargeted without changing the lower PR's head, the lower PR becomes the top of the stack, but its workflows do not rerun") | **Yes, by design** — pomerium's own PR body: "A lower PR in a stack can therefore merge on the evidence of the top PR's run, not its own." This is the whole point of the strategy, and its stated cost |
| **Bottom-N-full-CI** (here, bottom+top hybrid, N=2 — metabase PR #83715, shipped; yugabyte issue #34607, proposed) | **Code** (metabase, merged) + **official-docs-cited proposal** (yugabyte, open issue quoting GitHub's own "Optimizing CI for stacked pull requests" guidance) | Low-medium — metabase's version is a label override (`ci:run-all`) plus two cheap position checks (`position == size` for top, base-ref-match for bottom) on top of an already-existing path-filter gate | Bottom is *never* skipped (it's always the next thing that can actually merge, so it always gets its own full CI) — removes the single worst risk of leaf-only. Top failing just blocks the top PR as normal | **No for the bottom** (always runs its own CI) — **yes for the middle** (same trade-off yugabyde's issue names explicitly: "the intermediate commits on master were not built on their own. Only the combined result was") — a strictly smaller version of leaf-only's risk, not a different risk |
| **No optimization, just tiering** (ghstack, spr, git-spice's own dev repos; also this target repo's status quo today) | **Code** — all three stacking-tool authors' own live CI configs, confirmed above, run full CI on every layer unconditionally | Lowest — zero new code, this is what happens by default | N/A — there's nothing to wait on; every layer always gets full, independent CI | **No** — the strongest-possible guarantee on this axis, at the cost of paying full CI N times down an N-deep stack |

## Recommendation

**Start with the bottom+top hybrid (metabase's shipped pattern), not predecessor-polling and not pure
leaf-only.** Argument, from the evidence above, not from preference:

1. **The repo's actual shape makes the worst case of leaf-only unacceptable as a first move.** With PR
   #323 eight layers deep (`1-pr-stack-research.md`), pure leaf-only means seven of those eight PRs could
   merge having never had full CI run against their own exact diff — pomerium's own PR author flagged this
   as the cost of their design, and it is a materially bigger blast radius at 8 layers than at the 2-3
   layer stacks leaf-only is usually described for. Bottom+top removes the specific layer that actually
   matters most for safety — whichever PR is about to really merge next always gets its own full CI,
   every time, because "is this the bottom of the stack" is re-evaluated continuously as lower layers
   land (metabase: "The bottom is still detected by its base ref matching the stack base, which keeps
   working as lower PRs merge").

2. **It is the cheapest option that isn't the status quo.** Three real stacking-tool projects
   (`ghstack`, `spr`, `git-spice`) looked at this exact trade-off for their own repos and chose "no
   optimization" — that's legitimate, confirmed-in-code evidence that doing nothing is a defensible
   default, not an oversight. But this repo already knows its status quo is expensive (27/49 PRs
   stacked, 8 deep, `1-pr-stack-research.md`'s own open question: "how much redundant gate-running is
   already happening today, invisibly"). Predecessor-polling is the most capable fix found in this whole
   research pass, but it is a 485-line custom GitHub Action with its own test suite and API-budget
   tuning — disproportionate for a team with no stacking tool at all, doing this by hand. Bottom+top is a
   same-shape, much smaller version of the identical underlying idea (only test where it matters), built
   from the same primitives this repo's `gate` job already has (skip-tolerant aggregating job,
   `dorny/paths-filter`-style conditional jobs) plus one new cheap check: "is any open PR's base my head
   branch" (bottom check) and "is my base another open PR's head" (not-bottom check) — both answerable
   with a single `gh pr list --base <branch>` call, same primitive pomerium's 24-line script already uses
   in production.

3. **GitHub itself ships first-party metadata for exactly this** — metabase's `position`/`size` fields on
   the native stack payload are **official-docs tier** evidence (cited directly by the yugabyte issue
   too) that this is GitHub's own recommended shape for stacked-PR CI optimization, not a one-off hack.
   That lowers the risk of building something non-standard.

4. **Predecessor-polling remains the better long-term answer if this repo later wants every layer tested,
   just delayed rather than skipped** — worth revisiting once/if a dedicated stacking tool is adopted, or
   once bottom+top's "middle layers never individually tested" trade-off (explicitly accepted by both
   metabase and yugabyte, independently) stops being acceptable at this repo's depth. Not recommended as
   the *first* move: its complexity (custom topology inference, polling, fail-open timers) isn't justified
   until the cheaper bottom+top option is shown to be insufficient in practice.

## Sources

- `tmp/ghstack/.github/workflows/test.yml`, `lint.yml` (full files read)
- `tmp/ghstack/README.md` (full file read, "Structure of submitted pull requests" + "Design constraints"
  + "Ripley Cupboard" sections)
- `tmp/ghstack/src/ghstack/land.py` (full file, 231 lines)
- `tmp/spr/.github/workflows/ci.yml` (full file read)
- `tmp/spr/readme.md` (Merge status bits / merge command sections)
- `tmp/spr/spr/spr.go` lines 437-512 (`MergePullRequests`)
- `tmp/spr/github/pullrequest.go` lines 1-110 (`Mergeable`, `Ready`, `CheckStatus`)
- `tmp/spr/config/config.go` lines 20-120 (`RequireChecks`, `RequiredChecks`, `MergeCheck`, `MergeMethod`)
- `tmp/git-spice/.github/workflows/ci.yml` (full file read)
- `tmp/git-spice/doc/src/guide/merge.md` (full file, 373 lines)
- `curl https://api.github.com/repos/facebook/sapling` → `"size": 536579` (reason full clone skipped)
- `curl https://api.github.com/repos/facebook/sapling/contents/.github/workflows` → file listing (no
  PR-stack-gate file found; negative result)
- `gh pr diff 83715 --repo metabase/metabase` and `gh pr view 83715 --repo metabase/metabase --json
  title,body` (full diff and body read)
- `gh pr diff 10 --repo pomerium/agentops` and `gh pr view 10 --repo pomerium/agentops --json title,body`
  (full diff and body read)
- `gh issue view 34607 --repo yugabyte/yugabyte-db --json title,body` (full body read)
- `../1-pr-stack-research.md`, `../oss/nextjs-analysis.md` §4,
  `../oss/calcom-analysis.md` §3 (read first per task instructions, not re-summarized beyond what's needed
  for comparison)
