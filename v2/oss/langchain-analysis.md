# OSS Case Study: `langchain-ai/langchain`'s CI Gate

Status: research only, no decisions made, no code changed in either repo.

## The question this answers

Both `1-pr-stack-research.md` and `2-per-dir-test-dispatch-research.md` reference industry patterns
(Nx, Turborepo, Graphite) in the abstract. This doc checks one specific, large, real-world polyglot
monorepo — `langchain-ai/langchain`, shallow-cloned to `tmp/langchain` — to see what its *actual*,
running CI does, as concrete evidence rather than vendor docs. It's structurally analogous to this
repo's situation: many separate packages under `libs/*` (`libs/core`, `libs/langchain`,
`libs/community`-style partner packages under `libs/partners/*`), each with its own dependency
relationships, needing scoped-not-global CI.

All paths below are relative to `tmp/langchain/`.

## 1. How does langchain detect which `libs/*` packages a PR touched?

A custom Python script, not a GitHub Action marketplace path-filter action. The mechanism:

- `.github/workflows/check_diffs.yml`, the `build` job, uses `Ana06/get-changed-files@25f79e676...`
  (lines 54-58) to get the full list of changed files in the PR as JSON, then pipes it straight into:
  ```
  python .github/scripts/check_diff.py "$ALL_CHANGED_FILES" >> $GITHUB_OUTPUT
  ```
  (`check_diffs.yml` line 65)
- `.github/scripts/check_diff.py` is ~400 lines of hand-written Python. It is plain path-prefix
  matching at its core — the same mechanism this repo's `detect-services.py` POC uses — but it layers a
  dependency-graph step on top (see Q2). Concretely, for a changed file it checks:
  - `file.startswith("libs/partners")` → extracts `partner_dir = file.split("/")[2]` and adds that one
    partner package to the test set (`check_diff.py` lines 345-361).
  - `any(file.startswith(dir_) for dir_ in LANGCHAIN_DIRS)` where `LANGCHAIN_DIRS = ["libs/core",
    "libs/text-splitters", "libs/langchain", "libs/langchain_v1", "libs/model-profiles"]` (lines
    28-34, 321-333) — a specific, explicitly ordered list of "core-ish" packages.
  - `file.startswith("libs/standard-tests")` → hardcodes testing five specific partner packages
    (mistralai, openai, anthropic, fireworks, groq) alongside it (lines 334-343), with an explicit
    `# TODO: update to include all packages that rely on standard-tests (all partner packages)` comment
    admitting this list is incomplete/manually maintained.
  - A safety valve: `if len(files) >= 300: ... dirs_to_run["lint"] = all_package_dirs()` (lines
    290-294) — if the diff is too large (GitHub's changed-files API truncates around 300 files), it
    falls back to testing *everything*, on the assumption that the file list is incomplete rather than
    risk silently testing too little.
  - Another safety valve: changes under `.github/workflows`, `.github/tools`, `.github/actions`, or
    `.github/scripts/check_diff.py` itself trigger `extended-test` for all of `LANGCHAIN_DIRS` (lines
    296-315), with an inline comment explaining why: "changes to CI/CD infrastructure don't inadvertently
    break package testing, even if the change appears unrelated."

So: path-prefix matching, same base mechanism as this repo's POC, but wrapped in a custom script (not a
marketplace action) with explicit edge-case handling for large diffs and for changes to the CI tooling
itself.

## 2. Does it understand dependents? (the core question from `2-per-dir-test-dispatch-research.md`)

**Yes — directly, with a parsed dependency graph, not just a hardcoded list.** This is the single most
relevant finding for this repo's demonstrated `conflict-types`/`calculated-fields`/`mcp-auth-core` gap.

- `dependents_graph()` (`check_diff.py` lines 64-114) builds a `package name -> set of consuming
  directories` map by **parsing every `libs/**/pyproject.toml`'s `[project.dependencies]` and
  `[dependency-groups.test]` entries**, plus each package's `extended_testing_deps.txt`, looking for any
  dependency whose name contains `"langchain"`. This is a real dependency graph built from the actual
  package manifests at CI time — not a hand-maintained map, and not Nx/Turborepo either. It's bespoke.
- `add_dependents()` (lines 117-127) then takes the raw set of directories whose files changed and
  **expands it**: for each changed dir, it looks up `dependents["langchain-" + last_path_segment]` and
  adds all of those consumer directories too, before building the test matrix.
- Concrete example this directly answers for "if `libs/core` changes, do `libs/langchain`,
  `libs/partners/*` also get triggered?": **Explicitly NOT via the generic dependents mechanism** — the
  code calls this out directly:
  ```python
  # handle core manually because it has so many dependents
  if "core" in dir_:
      updated.add(dir_)
      continue
  ```
  (`check_diff.py` lines 121-123, inside `add_dependents`). So `libs/core`'s dependents are **excluded**
  from automatic expansion — a file comment explains why at the top of the file:
  ```python
  # When set to True, we are ignoring core dependents
  # in order to be able to get CI to pass for each individual
  # package that depends on core
  # e.g. if you touch core, we don't then add textsplitters/etc to CI
  IGNORE_CORE_DEPENDENTS = False
  ```
  (lines 42-46) — note the flag is currently `False`, meaning this special-case-skip for core is not
  presently toggled on generically; it's hardcoded as unconditional special-casing in `add_dependents`
  regardless of the flag value (the `if "core" in dir_` check runs independent of `IGNORE_CORE_DEPENDENTS`).
  Instead, `libs/core` changes are handled by a **different, separate mechanism**: the `LANGCHAIN_DIRS`
  cascade (lines 321-333) — if a changed file starts with `libs/core`, the code walks forward through the
  ordered `LANGCHAIN_DIRS` list and adds every directory *from `libs/core` onward* (`text-splitters`,
  `langchain`, `langchain_v1`, `model-profiles`) to `extended-test`, on the theory these are ordered by
  dependency depth. **Partner packages (`libs/partners/*`) are NOT included in this core cascade** — so
  touching `libs/core` alone does not re-test `libs/partners/openai`, etc., under the normal path.
- For genuinely shared non-core packages (anything whose name shows up as a `langchain-*` dependency in
  another package's `pyproject.toml`), the generic `add_dependents()` graph **does** work and does expand
  the test set to real consumers — this is the mechanism this repo lacks today for `conflict-types` etc.
- One more explicit manual override exists: `IGNORED_PARTNERS = ["huggingface"]` (lines 48-53) — removed
  from the dependents graph entirely "because of CI instability specifically in huggingface jobs," with
  a comment acknowledging this is a deliberate, named exception, not an oversight.

**Net for Q2:** langchain does have graph-aware dependent detection, built by parsing real manifests
(not Nx/Turborepo, not fully hand-maintained) — but `libs/core` itself is deliberately exempted from the
generic version and handled by a separate, hand-ordered list instead, with an explicit code comment
admitting this is because core "has so many dependents" that auto-expanding would be unwieldy. Partner
packages are not retested when only `libs/core` changes.

## 3. Is there a tiered structure?

Yes, and it's the clearest tiering example found in this research pass.

**Fast tier — runs on every PR, scoped to changed dirs only, via `check_diffs.yml`:**
- `lint` (`_lint.yml`) — ruff-based static checks (`make lint_package`, `make lint_tests`)
- `test` (`_test.yml`) — unit tests, run twice per package: once with current deps, once reinstalled
  with each package's own minimum supported dependency versions (`_test.yml` lines 55-72, via
  `get_min_versions.py`)
- `test-pydantic` (`_test_pydantic.yml`) — re-runs tests against every Pydantic 2.x minor version in the
  range the package and `libs/core` both support (computed dynamically from `uv.lock` + `pyproject.toml`,
  `check_diff.py` lines 158-210)
- `compile-integration-tests` (`_compile_integration_test.yml`) — only checks that integration test
  *files* import/compile correctly; does not execute them against live APIs
- `vcr-tests` (`_test_vcr.yml`) — integration tests replayed against pre-recorded HTTP cassettes
  ("VCR"), restricted to `VCR_PACKAGES = {"libs/partners/openai"}` (`check_diff.py` lines 36-40) — this
  is playback-only, no live credentials, explicitly to "catch stale cassettes"
- `extended-tests` — broader test suites needing extra installed dependencies

**Slow tier — does NOT run per-PR, runs on a schedule instead:**
- `.github/workflows/integration_tests.yml` — real integration tests against **live partner APIs with
  real credentials**. Header comment: "Routine integration tests against partner libraries with live API
  credentials... Runs daily with the option to trigger manually" (lines 1-5). Triggered by
  `schedule: cron: "0 13 * * *"` (daily, line 58) or manual `workflow_dispatch` (lines 11-56), **never**
  by `pull_request`. This is the closest langchain analog to this repo's excluded Playwright/e2e tier —
  real external dependencies, too slow/flaky/credential-sensitive to gate every PR on, so it's moved off
  the PR-blocking path entirely and run on a timer instead.

This is a concrete, real-world precedent for exactly the tiering pattern described abstractly in
`1-pr-stack-research.md`'s "Tiered CI" section: cheap checks block every PR; expensive/flaky/live-credential
checks run on a schedule, decoupled from individual PRs.

## 4. Aggregating gate job

Yes, functionally identical in purpose to this repo's `gate` job, named `ci_success`:

```yaml
ci_success:
  name: "✅ CI Success"
  needs:
    [build, lint, test, compile-integration-tests, vcr-tests, extended-tests,
     test-pydantic, check-release-options]
  if: |
    always()
  runs-on: ubuntu-latest
  env:
    JOBS_JSON: ${{ toJSON(needs) }}
    RESULTS_JSON: ${{ toJSON(needs.*.result) }}
    EXIT_CODE: ${{!contains(needs.*.result, 'failure') && !contains(needs.*.result, 'cancelled') && '0' || '1'}}
  steps:
    - name: "🎉 All Checks Passed"
      run: |
        echo $JOBS_JSON
        echo $RESULTS_JSON
        echo "Exiting with $EXIT_CODE"
        exit $EXIT_CODE
```
(`check_diffs.yml` lines 208-235)

Same pattern as this repo's POC: `if: always()` so it runs even when upstream matrix jobs were
conditionally skipped (`needs.build.outputs.test != '[]'` etc. on each matrix job), then inspects
`needs.*.result` for any `failure`/`cancelled` and exits non-zero only then — i.e. "nothing to check
here" still reports a clean pass, which is exactly the GitHub quirk the glossary describes and the
target repo's `gate` job already solves the same way. No evidence here of *how* `ci_success` is wired
into branch protection (that's a repo-settings concern not visible in workflow YAML), but the mechanism
itself is a direct match.

## 5. Evidence on PR chains/stacks

No direct evidence found, positive or negative. Searched `.github/` for the word "stack" — the only
hits were an internal git tool directory name (`.github/tools/git-restore-mtime`) and an unrelated issue
template field, neither about PR stacking. No `graphite.json`, no stacking-specific CI logic, no
mention of stacked/chained PRs in any workflow comment read during this pass. `check_diffs.yml` does
trigger on `merge_group:` (line 20) in addition to `push`/`pull_request` — meaning it participates in
GitHub's native merge queue — which is the mechanism `1-pr-stack-research.md` names as the real fix for
"stack drift" (testing a candidate merge against current `main` right before it actually merges, not
just against an original base). This wasn't explored further (no evidence of whether the merge queue
is actually turned on in branch protection, just that the workflow is wired to respond to it), so this
is circumstantial, not confirmation that langchain solves stacking — only that the plumbing for a merge
queue exists in this workflow, which is adjacent to the senior's underlying "how do you verify
end-to-end across the stack" concern.

## 6. What's genuinely novel or worth considering for this repo's gate

1. **A parsed dependency graph from existing manifests, not a new build system.**
   `2-per-dir-test-dispatch-research.md` frames the choice as either "adopt Nx/Turborepo" or
   "hand-maintain a small map." langchain's `dependents_graph()` is a third option: a ~50-line function
   that reads `pyproject.toml`'s `[project.dependencies]` directly off disk and builds the map at CI
   time, every run, from ground truth — no new tool, no stale hand-maintained map to fall out of sync.
   This repo's workspace equivalent (`package.json`'s `dependencies`/`devDependencies` referencing
   `@databonder/*` packages) is directly analogous data already sitting in every workspace's own
   manifest. This is a concretely cheaper path than either option named in
   `2-per-dir-test-dispatch-research.md` for closing the `conflict-types`/`calculated-fields`/`mcp-auth-core`
   gap, worth considering before reaching for Nx/Turborepo or a hand-maintained map.
2. **Explicit exemption of the highest-fan-in package from the generic graph, with a one-line
   rationale left in the code.** langchain doesn't try to make its generic dependents mechanism cover
   `libs/core` (its "conflict-types"-equivalent, but with far more consumers) — it says outright "core
   has so many dependents" and hand-codes a separate cascade instead. This is a direct, named precedent
   that "parse the graph generically, except special-case the one package with too much fan-out" is a
   legitimate, intentional design choice other large projects make — not something to treat as
   automatically wrong if this repo's gate ends up doing something similar for whichever of its shared
   packages has the most consumers.
3. **Oversized/truncated diffs default to full-matrix fallback, with a numeric trigger.** The `len(files)
   >= 300` fallback (`check_diff.py` line 290) is a specific, concrete answer to "what if changed-file
   detection itself can't be trusted" — this repo's own POC already proved an analogous "root file change
   → test everything" fallback works (`PLAN.md`'s root-`package.json`-touch test), but langchain's
   additional trigger — size of the diff itself, independent of *which* files changed — is a distinct
   safety valve this repo's POC doesn't appear to have (not confirmed either way in this pass; worth
   checking `detect-services.py` directly if adopting this).
4. **Tiering is also used to decouple live-credential/flaky tests from the PR-blocking path entirely**,
   not just to make them "slower" — `integration_tests.yml` isn't a slow job on the PR gate, it's a
   wholly separate, non-`pull_request`-triggered workflow on a daily cron. This is a stronger form of the
   tiering already recommended in `1-pr-stack-research.md` and is directly relevant to this repo's
   deliberately-excluded Playwright/e2e suite: rather than inventing a new "when does e2e run" answer,
   langchain's daily-cron-plus-manual-dispatch pattern is a ready-made template to point to.

No claim is made here about *why* langchain chose any of these designs beyond what's in the code's own
comments — where a comment states a rationale (e.g. "handle core manually because it has so many
dependents," "CI instability specifically in huggingface jobs"), it's quoted directly above; where no
comment exists, no rationale is invented.

## 7. Other workflows checked afterward — three more worth knowing about

The rest of `.github/workflows/` (26 files total) is mostly issue/project-management automation
(`close_unchecked_issues.yml`, `require_issue_link.yml`, `reopen_on_assignment.yml`,
`tag-external-issues.yml`, `openwiki-update.yml`, `auto-label-by-package.yml` — the last one labels
*issues* from a dropdown field in the issue template, not PRs, and isn't diff-based) — unrelated to CI
gating, not covered further. Three files are genuinely relevant and weren't in the first pass:

**`check_extras_sync.yml` + `.github/scripts/check_extras_sync.py`** — a small, dedicated workflow
triggered only on `libs/**/pyproject.toml` changes. It iterates every package's manifest and checks for
internal drift (its optional-dependency "extras" declarations matching what they should), failing with
an explicit list of exactly which manifest(s) are out of sync. This is a direct, working precedent for
the kind of lightweight "catch manifest drift automatically" workflow this repo's own research already
flagged as worth having — specifically, this is exactly the shape of check that would have caught
`StrapiMCP`'s `yarn.lock`/`package.json` drift (found in `PLAN.md`) the first time it happened, instead
of it sitting silently broken until this gate's POC exercised it. (`.github/workflows/check_extras_sync.yml`,
`.github/scripts/check_extras_sync.py`)

**`.github/actions/uv_setup`** — a reusable *composite action* (not a reusable workflow) that wraps
"install Python + `uv` + configure dependency caching" in one place, called identically from `_lint.yml`,
`_test.yml`, `check_extras_sync.yml`, and others. This repo's `pr-gate.yml` currently repeats
`corepack enable` + `yarn install --immutable` inline in both the `fast-checks` and `docker-build`
(indirectly, via standalone installs) jobs — a composite action equivalent would consolidate that into
one place, so a future fix (like the `lint-staged` skip or the per-service install-path logic) only needs
updating once instead of once per job. (`.github/actions/uv_setup/action.yml`)

**`codspeed.yml`** — performance-regression benchmarking, out of scope for this task, but notable for
*how* it decides what to benchmark: it reuses the exact same `check_diff.py` changed-file detection this
doc already analyzed for Q1/Q2, just reading a different output key (`codspeed` instead of `test`/`lint`).
Concrete proof that one well-built "what changed, what are the dependents" detector can back more than
one kind of job — not just build/lint/test dispatch — without needing a second, parallel detection
mechanism. (`.github/workflows/codspeed.yml`)

One more checked and judged *not* directly relevant, included for completeness: **`pr_labeler.yml`** —
a sophisticated PR auto-labeling workflow (size, files-touched, title, contributor-tier labels), safely
using `pull_request_target` (never checks out PR code, per its own inline warning comment). Not about
build/test gating, but the "files-touched" labeling piece is a reusable idea: this repo's own
`detect-services.py` output could drive an auto-label step the same way, so a reviewer looking at a PR
*stack* (see `1-pr-stack-research.md`) could see at a glance which services each layer touches without
reading every diff. Noting as a nice-to-have, not a gating mechanism.

## Sources

- `.github/workflows/check_diffs.yml` (full file read)
- `.github/scripts/check_diff.py` (full file read)
- `.github/workflows/_lint.yml`, `.github/workflows/_test.yml` (full files read)
- `.github/workflows/integration_tests.yml` (header + trigger block read)
- `.github/workflows/block_fork_main_prs.yml` (header comment only, not stack-related, included for
  completeness of what was checked)
- `grep -rli stack .github/` over the full `.github/` tree — no stacking-specific CI logic found
- No `CONTRIBUTING.md` found at repo root in this shallow clone (`find . -iname "CONTRIBUTING*"` returned
  nothing) — CI-related contributor docs, if any, were not available to check in this pass.
