# OSS Case Study: `hashicorp/vault`'s CI Gate

Status: research only, no decisions made, no code changed in either repo. Read `oss/langchain-analysis.md`
first — this doc focuses on where Vault does something different, rather than re-deriving ground already
covered there.

Vault was shallow-cloned to `tmp/vault`. All paths below are relative to `tmp/vault/` unless stated
otherwise.

## Which CI system is primary

**GitHub Actions, not CircleCI.** `find . -iname ".circleci"` / direct check for `.circleci/config.yml`
turned up nothing in this clone — no `.circleci` directory exists at all. `.github/workflows/` has 34
workflow files and is clearly the active, maintained CI system (recent-looking code, inline comments
explaining rationale, references to live infrastructure like Vault-the-product-itself for secrets, Slack
webhooks, Datadog, Instana). HashiCorp's CircleCI-heavy era (true of many of their older repos) does not
apply to this repo's current state. Everything below is GitHub Actions.

## 1. How does Vault decide what to build/test for a given PR?

**A custom, declarative, HCL-configured path-matching system, parsed by a bespoke Go CLI — not a
marketplace action, not `go list`-driven dependency detection, and not "run everything."**

The mechanism, concretely:

- `.github/workflows/ci.yml`'s `setup` job calls `./.github/actions/changed-files` (line 85), which is a
  composite action (`.github/actions/changed-files/action.yml`) that shells out to a custom-built Go CLI:
  ```
  pipeline github list changed-files --owner hashicorp --repo '${{ github.event.pull_request.head.repo.name }}' --pr '${{ github.event.pull_request.number }}' --github-output
  ```
  (`.github/actions/changed-files/action.yml`, lines 30-36). `pipeline` is `tools/pipeline`, a full Go
  module living inside the Vault repo itself (its own `go.work` entry) — roughly a dozen `internal/cmd/*`
  subcommands for changed-file detection, Go module/workspace checking, backport automation, branch
  syncing, and more.
- The actual grouping rules are **declarative, not hardcoded in Go**: a single HCL config file,
  `tools/pipeline/internal/pkg/config/fixtures/pipeline.hcl` (used as the real schema/example; the
  production config is the equivalent file loaded at runtime), defines named `group` blocks — `app`,
  `autopilot`, `changelog`, `community`, `docs`, `enos`, `enterprise`, `github`, `gotoolchain`,
  `pipeline`, `proto`, `ui` — each with `match`/`ignore` blocks keyed on `extension`, `base_dir`,
  `base_name`, `base_name_prefix`, and `contains` (substring) predicates. Example:
  ```hcl
  group "ui" {
    match {
      base_dir = ["ui"]
    }
  }
  ```
  (lines 210-214) — this is plain path-prefix matching, the same base mechanism as this repo's
  `detect-services.py` and langchain's `check_diff.py`, just declared in HCL instead of imperative code.
- `tools/pipeline/internal/pkg/changed/config.go`'s `FileGroups()` (lines 46-71) evaluates every changed
  file against every group's `Match`/`Ignore` matchers and returns the set of groups a file belongs to. A
  single file can belong to multiple groups (e.g. a `.go` file under `vault_ent/` matches both `app` and
  `enterprise`).
- `ci.yml`'s `setup` job exposes `changed-files` as a JSON blob of `{"groups": [...]}", and every
  downstream job's `if:` condition is a boolean expression over `contains(fromJSON(needs.setup.outputs.changed-files).groups, 'go_app')`-style checks (e.g. `ci.yml` lines 175-181, 210-215, 326-330). This is
  functionally identical in shape to this repo's own `detect-services.py` → `pr-gate.yml` wiring (map
  changed paths to named groups, then gate jobs on group membership) — just with the mapping table
  externalized into one shared HCL file instead of embedded directly in the detection script.

**Net for Q1:** path-based filtering, same family as this repo's and langchain's approach, but
implemented as a dedicated internal tool with declarative config rather than an imperative per-repo
script or a marketplace action. Vault does **not** fall back to "run everything on every PR" at the
top-level workflow-selection layer — jobs are conditionally skipped when their group isn't touched (e.g.
`test-ui` only runs if `ui`/`test_ui`/`test_ci` groups are present, `ci.yml` lines 326-330).

## 2. Does it understand dependents, or is testing scoped a different way?

**Neither a parsed-manifest dependency graph (like langchain) nor a `go list`-based import-graph
"affected" calculation. Once the coarse "did any Go code change" gate trips, Vault tests the entire Go
module(s) in full, every time — relying on Go's own build cache and heavy parallelism (not selective
dependency-graph test picking) to keep this affordable.**

Concrete evidence:

- `ci.yml`'s `test-go` job (lines 173-206) triggers if **any** of the groups `go_app`, `go_modules`,
  `go_toolchain`, `go_tests`, `pipeline`, or `test_ci` appear anywhere in the changed-files set — i.e. the
  decision is "did Go-relevant code change at all," not "which packages are affected by this specific
  change."
- Once triggered, `test-go.yml`'s `test-matrix` job calls `pipeline go list packages --format json --tags
  "$GO_TAGS" "${module_flags[@]}"` (`test-go.yml` line 194). The underlying implementation,
  `tools/pipeline/internal/pkg/golang/list_packages.go`'s `Run()` method (lines 37-80), literally runs
  `go list ./...` (line 56) in each module's directory — **this lists every package in the module, with
  no filtering for "changed" or "affected."** The doc comment on the function type even says: "Modules
  are module directories relative to the workspace root. When empty, every module in go.work is listed"
  (lines 24-26) — selection granularity stops at the **module** level (am I testing this whole Go module
  or not), never descends to "which packages within it actually changed or are reachable from a changed
  file."
- The one module-level scoping mechanism that does exist, `go-test-modules` (`ci.yml` lines 41-55), is a
  narrow special case, not a general dependency graph: if the **only** changed-file group is `pipeline`
  (i.e. only `tools/pipeline` itself changed) and no `go_app`/`go_modules`/`go_tests`/`go_toolchain`/
  `test_ci` group is also present, it sets `go-test-modules: tools/pipeline`, limiting the `go list
  packages` call to that one module — "because no other module depends on it," per the inline comment
  (lines 41-44). This is the **inverse** of dependents-awareness: it's a hand-verified claim that
  `tools/pipeline` has zero dependents, used to narrow scope, not a mechanism that would **expand** scope
  if a widely-depended-on module changed. If any other Go file changes alongside it, or if `go_modules`/
  `go_app` is also touched, this narrowing doesn't apply and `modules` stays empty, meaning **every**
  module in `go.work` gets listed and fully tested (the `total-runners` input even swaps between `'16'`
  and `'2'` based on exactly this condition, `ci.yml` line 205) .
- `check-go`'s "Check that the Go tests cover every module" step (`ci.yml` lines 528-574) is a
  **coverage-completeness check, not an affected-selection mechanism** — it independently calls `pipeline
  go list modules` and `pipeline go list packages`/`pipeline go group packages` and diffs the results to
  make sure no module in `go.work` is silently missing from the test matrix. This is a safety net against
  the test matrix *under*-covering modules, which only reinforces that the design intent is "test full
  modules, always," with no selective affected-subset path to accidentally narrow.

**Net for Q2:** Vault doesn't need the "which dependents also need retesting" problem this repo
demonstrated for `conflict-types`/`calculated-fields`/`mcp-auth-core`, because it doesn't attempt
package-level selective testing at all once a Go-code change is detected — it tests the **entire**
affected module's full package list, every time, and relies on 16 parallel runners plus Go's build/test
cache (and `gotestsum`'s timing-file-driven partitioning, `test-go.yml` lines 164-171, 204-206) to make
"test everything in scope" fast enough. The brief's "valid finding if true" caveat about large Go
monorepos applies here, but with an important nuance: it's not "run everything in the whole repo
unconditionally" (the coarse group-level gate in Q1 still exists and still skips jobs entirely when e.g.
only UI or only docs changed) — it's "once Go is in scope at all, stop trying to narrow further than the
module level."

## 3. Is there a tiered structure?

**Yes, with more tiers and more distinct tiering dimensions than langchain's PR-blocking-vs-cron split —
Vault tiers along four separate axes: speed/confidence, draft-vs-ready, license/edition, and
release-vs-PR.**

### Axis 1 — Fast required tests vs. opt-in/best-effort tests (per-PR, same workflow)

- `test-go` (standard tags) and `test-go-testonly` are unconditionally required once Go changed
  (`ci.yml` lines 173-237).
- `test-go-race` (data race detector) **only runs once a PR leaves draft status** —
  `needs.setup.outputs.is-draft == 'false'` is the very first condition in its `if:` (`ci.yml` line 243)
  — an explicit, code-enforced "don't pay for the expensive check while the PR is still a draft" tier.
  This is a **different tiering axis than anything in langchain's analysis**: langchain tiers by
  trigger-event (PR vs. cron); Vault additionally tiers by **PR readiness state within the same
  `pull_request` trigger.**
- `tests-completed`'s final status rollup (`ci.yml` lines 632-635) explicitly **excludes** `test-go-fips`
  and `test-go-race` from the required-job calculation: `del(.["test-go-fips"], .["test-go-race"]) as
  $required` — with the inline comment "We allow fips and race tests to fail so we don't consider their
  result here" (lines 627-628). This is a tier of tests that **run and report, but don't block merge** —
  a middle ground between "required" and "not run at all" that neither this repo's POC nor langchain's
  analysis surfaced.

### Axis 2 — License/edition-gated tests

- `test-go-fips` (`ci.yml` lines 280-321) only runs when `needs.setup.outputs.is-ent-branch == 'true'`
  **and** the trigger is a `push` (merge to main/release) **or** the PR carries the `fips` label — i.e.
  FIPS 140-3 compliance testing (`go-tags: ...,fips,fips_140_3`, `GOEXPERIMENT: boringcrypto`) is gated
  both by enterprise-edition-only **and** by default-off-unless-labeled on PRs. This is exactly the
  "license-gated test suite only Vault Enterprise would need" the task description anticipated, found
  concretely.
- `test-autopilot-upgrade` (`ci.yml` lines 101-171) is similarly enterprise + main-branch-only gated
  (`needs.is-ent-branch == 'true' && (github.base_ref == 'main' || github.ref == 'refs/heads/main')`), and
  separately depends on a **cross-repository** artifact — `.release/versions.hcl` plus live
  `gh -R hashicorp/vault-enterprise release list` calls (lines 122-159) — to determine which old Vault
  versions to upgrade-test against. Not found in langchain or in this repo's own research: a test tier
  whose *scope* (which versions to test) is computed dynamically from a separate repository's release
  history at CI time.

### Axis 3 — Full multi-cloud / infrastructure integration tests, decoupled entirely from `pull_request`

- `enos` (`.github/workflows/test-run-enos-scenario*.yml`, `test-enos-scenario-ui.yml`) is HashiCorp's
  infrastructure-provisioning test framework — real cloud resources, scenario matrices
  (`test-run-enos-scenario-matrix.yml` takes `sample-max`/`sample-name` inputs describing how many
  scenario combinations to sample, lines 1-38). `enos-release-testing-oss.yml`'s **only** trigger is
  `repository_dispatch: types: [enos-release-testing-oss, enos-release-testing-oss::*]` (lines 1-6) —
  **not** `pull_request`, **not** `push`, **not** even `schedule`; it's fired externally by a separate
  release-build pipeline, gated further on `github.event.client_payload.payload.branch` starting with
  `release/` (line 10). This is a stronger decoupling than langchain's daily-cron `integration_tests.yml`
  — Vault's heaviest tier isn't on a timer at all, it's purely event-driven from the release pipeline,
  meaning it never runs against an arbitrary PR under any circumstance, only against a built release
  artifact.
- `enos-lint.yml` is a separate, lighter, PR-triggered check that only validates the Enos HCL scenario
  definitions themselves compile/lint correctly — distinguishing "does the test definition look right"
  (cheap, PR-gated) from "does the infrastructure scenario actually pass" (expensive, release-gated
  only).

### Axis 4 — CE vs. Enterprise build variants, as a first-class build-time fork, not just a test tier

- `build-artifacts-ce.yml` and `build-artifacts-ent.yml` are two separate, parallel-structured workflows
  (confirmed to share an explicitly-documented common input/output "interface," per the comment at the
  top of `build-artifacts-ce.yml`: "The inputs and outputs for this workflow have been carefully defined
  as a sort of workflow interface... must be consistent across the build-artifacts-ce workflow and the
  build-artifacts-ent workflow," lines 3-5). `go-tags` (computed once, in `metadata`'s `workflow-metadata`
  step) is the single switch threaded through nearly every job (`ci.yml` lines 198, 229, 314, 198) to
  select CE-only vs. enterprise-tagged (`_ent.go`, `enterprise` build tag) code paths from the **same**
  source tree and **same** workflow structure, rather than maintaining two different CI pipelines.

**Net for Q3:** this is a materially richer tiering structure than langchain's, along axes langchain
didn't need (draft-state, license/edition, cross-repo release-pipeline gating) — concrete, code-verified
evidence directly useful for this repo's eventual e2e-tiering decision (`2-per-dir-test-dispatch-research.md`).

## 4. Required checks / aggregating job

**Yes — `tests-completed` in `ci.yml` (lines 604-779), functionally the same pattern as this repo's
`gate` job and langchain's `ci_success`, but with one added wrinkle: it treats some upstream jobs as
"tracked but not required."**

```yaml
tests-completed:
  needs:
    - setup
    - check-go
    - test-autopilot-upgrade
    - test-go
    - test-go-testonly
    - test-go-race
    - test-go-fips
    - test-ui
  if: always()
  ...
  steps:
    - name: Determine status
      id: status
      run: |
        if results=$(jq -rec 'del(.["test-go-fips"], .["test-go-race"]) as $required
            | $required | keys as $jobs
            | reduce $jobs[] as $job ([]; . + [{job: $job}+$required[$job]])' <<< '${{ toJSON(needs) }}'
        ); then
          if jq -rec 'length as $expected
            | [.[] | select((.result == "success") or (.result == "skipped"))] | length as $got
            | $expected == $got' <<< "$results"; then
            msg="All required test jobs succeeded!"
            result="success"
          else
            msg="One or more required test jobs failed!"
            result="failed"
          fi
        ...
```
(`ci.yml` lines 604-651)

Same shape as `ci_success`/this repo's `gate`: `if: always()` so conditionally-skipped jobs don't block
it, success defined as "every required job's result is `success` or `skipped`," and a final
`exit 1` (line 778, under `Check for failed status`) if not. The genuinely different piece: `fips` and
`race` are **explicitly deleted from the required set before evaluating** (`del(.["test-go-fips"],
.["test-go-race"])`) — i.e. Vault's aggregating job has a **built-in notion of "this ran, reports to
Slack/PR comments if it fails, but doesn't block the merge gate,"** a third state beyond this repo's and
langchain's binary required/not-run. Worth naming for this repo if a similar "informational, not
blocking" tier is ever wanted (e.g. a flaky-prone suite you want visibility into without blocking small
PRs on it).

No evidence in this pass of how `tests-completed` is wired into GitHub branch protection rules (that's a
repo-settings concern, not visible in workflow YAML) — same caveat langchain's analysis noted for
`ci_success`.

## 5. Evidence of PR-stack handling

**None found, same as langchain.** `grep -rli "stack" .github/ tools/pipeline/` style searches surfaced
nothing about stacked/chained PRs specifically. No Graphite, no git-spice artifacts. `ci.yml`'s
`concurrency` block (`group: ${{ github.head_ref || github.run_id }}-ci`, lines 20-22) cancels
in-progress runs on new pushes to the same branch — useful for any PR including a stacked one, but that's
generic GitHub Actions hygiene, not stack-specific logic. No `merge_group:` trigger was found in `ci.yml`
(unlike langchain's `check_diffs.yml`, which does listen for `merge_group`) — so unlike langchain, there
isn't even circumstantial evidence of merge-queue wiring here.

## 6. Genuinely novel findings, not covered by this repo's approach or langchain's analysis

These are the findings most relevant to the senior's two review questions — automated handling of a
**tree** of long-lived parallel branches, which is structurally the closest thing in this research pass
to "stacking," even though it is not PR-stacking in the literal sense.

1. **Automated backport-PR creation on merge, computed from a decoded release-version config — the
   single most relevant finding to the stacking question.** `tools/pipeline/internal/cmd/github_create_backport.go`
   and `tools/pipeline/internal/pkg/github/create_backport.go` implement a `CreateBackportReq` designed
   to run from a `pull_request_target: types: closed` event, gated on `github.event.pull_request.merged`
   (doc comment, `create_backport.go` lines 27-41). On every merge to `main`, this automatically opens
   new PRs against every **active** release branch (`VersionsDecodeRes`, sourced from the same
   `.release/versions.hcl` the autopilot-upgrade job reads, lines 60-64) — i.e. instead of a human
   manually stacking or cherry-picking a change across `release/1.17.x`, `release/1.18.x`, etc., a bot
   does it the moment the source PR lands. This is the direct structural analog to "verifying end-to-end
   across a stack" from `1-pr-stack-research.md`: Vault's "stack" is the tree of active release branches,
   not a chain of unmerged feature PRs, but the same underlying problem — a change needs to propagate
   correctly through multiple dependent branches, and each propagated copy needs its **own** independent
   CI run before it's trusted — is handled by generating a brand-new PR per branch (which then goes
   through the exact same `ci.yml` gate as any other PR) rather than by any special-cased "stack-aware"
   CI logic. The takeaway for this repo: when (if) a PR-stacking tool is adopted, the right comparison
   point isn't just Graphite/git-spice (as researched in `1-pr-stack-research.md`) — it's also "does
   propagating a change across dependent branches/PRs need a dedicated automation path separate from
   manual rebasing," which Vault answers with a purpose-built bot rather than tooling off the shelf.

2. **The backport automation knows how to *subtract* content per-destination, not just replay it
   wholesale.** `CreateBackportReq.CEExclude` (changed-file groups to drop when backporting from
   Enterprise down to CE, `create_backport.go` lines 70-72) and `CEAllowInactiveGroups` (lines 75-77)
   mean the automated backport isn't a naive cherry-pick — it reads the **same** HCL-configured
   `changed_files` group taxonomy from Q1 to decide which files in the merged commit should be **excluded
   entirely** when the backport target is a CE (non-enterprise) branch. This reuses the Q1 path-matching
   detector for a second, unrelated purpose (branch propagation filtering, not just test selection) — a
   concrete example of "one well-built changed-file classifier backs more than one kind of downstream
   decision," the same principle langchain's analysis noted for `codspeed.yml` reusing `check_diff.py`,
   but applied here to branch-merge content filtering rather than job dispatch.

3. **A separate, generic branch-sync primitive (`SyncBranchReq`) for keeping CE and Enterprise trunks
   aligned via merge, independent of the backport-PR mechanism.** `tools/pipeline/internal/pkg/github/sync_branch_request.go`
   implements "synchronize two GitHub-hosted branches with a git merge from one into another"
   (lines 24-27), parameterized by arbitrary owner/repo/branch on both sides, with an optional
   `DisallowedGroups` filter (reusing the same group taxonomy again). This is infrastructure for keeping
   **two long-lived parallel branches** (CE `main` and Enterprise `main`, in different repos) from
   drifting apart — the closest thing in this whole research pass to a generic, reusable answer to
   "stack drift" (`1-pr-stack-research.md`'s Finding: "if `main` moves forward while a stack sits open,
   every layer's PR is still diffed against its original parent"). Vault's answer, for its specific case
   of two permanently-diverged-but-related trunks, is a scheduled/triggered merge-sync bot rather than a
   merge queue — a different tool for a structurally similar problem (parallel branches that must not be
   allowed to silently diverge).

4. **Draft-PR-state as its own CI tier, enforced in code, not just a GitHub UI convention.** Covered in
   Q3 — `needs.setup.outputs.is-draft == 'false'` gating the race-detector job (`ci.yml` line 243) is a
   tiering axis neither this repo's POC nor langchain's analysis has: "don't run the expensive check
   while the author is still iterating in draft," automatically re-enabled the moment the PR is marked
   ready (`ci.yml`'s trigger block explicitly adds `ready_for_review` to the default `pull_request` types
   for exactly this reason, lines 4-12: "when a draft pr is marked ready, we run everything, including
   the stuff we'd have skipped up until now"). Directly applicable to this repo's own Playwright/e2e
   tiering question: "skip the expensive e2e tier while in draft, run it when marked ready" is a cheap,
   already-proven pattern this repo doesn't need to invent from scratch.

5. **A "runs but doesn't block" tier inside the aggregating job itself** (`del(.["test-go-fips"],
   .["test-go-race"])`, Q4 above) — a third state between "required" and "doesn't run at all" that is
   genuinely new relative to both this repo's binary required/optional model and langchain's `ci_success`
   (which has no such carve-out; all of `check-go`'s `needs` list is implicitly required by omission of
   any `del()`-style filtering).

## Sources

- `.github/workflows/ci.yml` (full file read)
- `.github/workflows/test-go.yml` (full file read)
- `.github/actions/changed-files/action.yml`, `.github/actions/metadata/action.yml` (full files read)
- `tools/pipeline/internal/pkg/changed/config.go` (full file read)
- `tools/pipeline/internal/pkg/config/fixtures/pipeline.hcl` (full file read — used as the representative
  schema for the `changed_files` group taxonomy)
- `tools/pipeline/internal/pkg/golang/list_packages.go` (full file read)
- `tools/pipeline/internal/pkg/github/create_backport.go`, `sync_branch_request.go` (partial reads —
  struct definitions and doc comments)
- `.github/workflows/oss.yml`, `.github/workflows/enos-release-testing-oss.yml`,
  `.github/workflows/test-run-enos-scenario-matrix.yml`, `.github/workflows/build-artifacts-ce.yml`,
  `.github/workflows/bob-review-gate.yml`, `.github/workflows/do-not-merge-checker.yml` (header/partial
  reads)
- `find . -iname ".circleci"` / repo-root listing — confirmed no CircleCI config exists in this clone
- `grep -rli "stack" .github/ tools/pipeline/` — no stacking-specific CI logic found, consistent with the
  langchain analysis's equivalent search
