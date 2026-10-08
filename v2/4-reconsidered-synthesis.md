# v2 Research → Reconsidering Pick #1 (default-direction for change detection)

Status: a recommendation, not yet implemented. This doc revisits exactly one item from
`3-recommended-adoptions.md` — pick #1, the cal.com-derived "exclusion-based default" — at the explicit
request of the person running this project, who pushed back on literally adopting cal.com's
"match-everything-by-default, exclude a short list" direction for this repo. Everything else in
`3-recommended-adoptions.md` is carried forward unchanged (see "What survives untouched" below).

Organizing lens for every recommendation below, per the senior's own words: **"we want to know if the
build passes and whatever changed does get built."** Not "run everything, always" — correctly detect what
changed, and confirm that specific thing gets built and tested. That principle is the test every item below
is held to, including the ones carried over unmodified.

## Reconciling with pick #1: what changes, why, and what survives

**What `3-recommended-adoptions.md` pick #1 said:** adopt cal.com's `dorny/paths-filter` pattern directly —
match `'**'` by default, explicitly exclude a short list of known-safe non-code paths (`.vscode/**`,
`**/*.md`, `docs/**`, etc.), falling back to "run the full matrix" for anything not excluded.

**Why that doesn't actually transfer to this repo, on a closer look — not just a taste preference:**

cal.com's `'**'`-by-default pattern and this repo's `detect-services.py` are not doing the same *kind* of
work, even though both produce a boolean/array used to gate jobs. Re-reading `pr.yml`
(`oss/calcom-analysis.md` §2) against this repo's actual matrix mechanics in
`/Users/0xjustuzair/Desktop/keytoz/databonder-enterprise-search/.github/scripts/detect-services.py`
surfaces a structural mismatch the earlier pass didn't call out:

- cal.com's `has-files-requiring-all-checks` boolean feeds *generic, repo-wide* commands (`turbo run
  lint`, `turbo run type-check`, one Next.js build, one NestJS build) that already know how to walk the
  whole workspace tree on their own. A new, unlisted top-level directory in cal.com's repo still gets
  built/tested the moment `'**'` matches it, because "everything" there really does mean "every workspace
  Turborepo/pnpm can see" — no second per-directory registration step is required for the generic commands
  to pick it up.
- This repo's gate is architecturally different: `fast-checks` and `docker-build` run over an **explicit,
  enumerated JSON matrix** built from the 23-entry `SERVICES` list in `detect-services.py` (lines 41-66).
  Each entry carries per-service metadata a generic command can't infer — `standalone` (is it even a
  registered Yarn workspace, confirmed via `yarn workspaces list`, or does it need its own `cd && npm/yarn
  install`?), `target` (`production` vs. `runner`, confirmed per-Dockerfile), `context` (repo-root vs.
  service-own-directory for `COPY` resolution), and `ecr_blocked` (does this Dockerfile's base image need
  AWS creds to even pull?). **A "run everything" fallback in this repo's architecture still only means
  "run the matrix over the 23 entries that already exist in `SERVICES`"** — it is incapable of testing a
  brand-new, unregistered 24th service, because there is no matrix entry describing *how* to build it. Flip
  cal.com's switch here and the actual gap (new service gets zero coverage) is **not closed** — you'd just
  be running the existing 23 services' full checks on every PR that touches anything outside the small
  excluded list, which is exactly the "heavy, messy, slow" outcome the person running this project flagged,
  for a fallback that still wouldn't test the new service that justified it.
- This is also where the "heavy packages, heavy deps" concern is concrete, not just a feeling: 14 of 18
  Dockerfiles pull from a private ECR registry (`TEMP-PLAN.md` §3) and most Node workspaces have real
  install/build costs (`yarn install --immutable` plus `tsc`/`next build`). A "match everything except docs"
  default means any PR touching, say, one `workers/*` worker's README-adjacent non-`.md` config, or any file
  under a path not on the short exclude list, would schedule **all 23 services'** build/lint/test plus
  whichever Docker builds are unblocked — every time, for PRs that didn't need it. That cost is structural
  to this repo's matrix shape, not avoidable by tuning the exclude list, because the exclude list approach
  only ever produces a binary "run the known matrix or don't" — it has no concept of "run just the one new
  thing," which is the actual problem to solve.

**So: pick #1 as literally specified (exclusion-based default-direction) does not survive.** The underlying
gap it was chosen to close — "a new/unlisted service gets zero coverage" — is real and worth closing. What
changes is the mechanism: instead of inverting the default-match direction for every file on every PR, treat
"new unregistered service directory" as its own narrow, cheap, loudly-failing check, independent of the
existing per-service matrix. See the revised recommendation below.

**What survives from the original pick #1 write-up:** the diagnosis (not the fix). `3-recommended-adoptions.md`
was correct that a brand-new top-level directory gets zero coverage under the current design, and that this
is a real, demonstrated gap in this repo's own `SERVICES` table — not a hypothetical. That finding is reused
below; only the proposed mechanism changes.

## Revised recommendation: a registration check, not a default-direction flip

The core idea: keep the existing inclusion-list matrix exactly as it is (it is precise, cheap, and already
proven against 23 real services) — but add one small, independent, always-cheap job that answers a
different, narrower question than "what changed": **"does every directory that looks like a service
actually have a matrix entry describing how to build it?"** This only needs to run a lightweight structural
check, not a build, so it stays fast regardless of which PR triggers it — including the common case where
nothing new was added and it should do effectively nothing.

Two inputs already exist in this repo as sources of truth and don't need to be invented:

1. `yarn workspaces list --json` — the real, canonical list of the 19 registered Yarn workspaces
   (`TEMP-PLAN.md` §3, confirmed directly: `ruleengine/rules-service`, `ruleengine/rules-studio`,
   `StrapiMCP`, and `packages/document-preview` are real directories with their own `package.json` that
   `yarn workspaces list` does **not** return, because the root `package.json`'s `workspaces` array doesn't
   match them).
2. A filesystem scan for "does this top-level directory contain a `package.json` or `Dockerfile`" — the
   same two signals `detect-services.py` already keys its own `has_package_json`/`dockerfile` fields on.

A new directory that contains either signal, and isn't accounted for by either source, is the exact case
this needs to catch — not "any file under any unmatched path," which is what made cal.com's `'**'` pattern
too broad for this repo's matrix shape.

| Idea | Source | How it works there | What we'd adapt for this repo | Effort |
|---|---|---|---|---|
| **Workspace-array diff as a validation step, not a default-direction change** | This repo's own `package.json` `workspaces` array + `yarn workspaces list` (already the source of truth per `TEMP-PLAN.md` §3); pattern of "parse real manifests, don't hand-maintain a shadow copy" from `oss/langchain-analysis.md` §2 (`dependents_graph()` reading `pyproject.toml` directly) | langchain never hand-maintains a dependents map — it reads `pyproject.toml` at CI time and trusts that over anything hardcoded | Add a small check (in `detect-services.py` or a tiny sibling script) that runs `yarn workspaces list --json`, plus a scan of top-level dirs (and one level into `packages/`, `workers/`, `ruleengine/`) for `package.json`/`Dockerfile`, and diffs that combined set against the `path` values already in `SERVICES`. Anything present on disk/in the workspace graph but absent from `SERVICES` is flagged. This reuses existing ground truth instead of inventing a second list to keep in sync — same principle `2-per-dir-test-dispatch-research.md` already named as the risk with any hand-maintained map | Small — one new function, no new dependency (`yarn workspaces list --json` already works in this repo today) |
| **Fail loudly and specifically, not "run everything"** | `oss/langchain-analysis.md` §7 (`check_extras_sync.yml` — a dedicated workflow that fails with an exact list of exactly which manifest is out of sync, rather than silently working around it); `oss/vault-analysis.md` §2 (`check-go`'s "Check that the Go tests cover every module" step — a coverage-completeness check, explicitly *not* an affected-selection mechanism) | Both repos treat "is our test coverage declaration internally consistent" as its own small, independent check that fails fast with a specific message, separate from the main test-dispatch logic | When the diff above finds an unregistered directory, the gate fails immediately with a message naming the exact path(s) and asking for a one-line addition to `SERVICES` — the same shape of fix as adding a new entry to the table today, just forced to happen *before* merge instead of silently never happening. This directly serves "whatever changed does get built": a PR that introduces a new service can't land claiming to be checked when it wasn't — it either gets registered (and genuinely tested, with correct `standalone`/`target`/`context`/`ecr_blocked` metadata, same as every other entry) or the PR is blocked until it is. Critically, this does **not** fall back to running the existing 23-entry matrix — as shown above, that fallback wouldn't test the new directory anyway, so a loud, specific failure is strictly more honest than a "full matrix ran successfully" green check that never touched the actual new code | Small — a few lines of comparison logic plus an `exit 1` with a clear message; no new job needed, this can be a step inside the existing `detect` job |
| **Scope this to only fire on PRs that could plausibly introduce the gap, not every PR** | Reasoning from this pass, not a specific OSS precedent — but consistent with the general principle every case study showed (langchain's `len(files) >= 300` fallback, Vault's `go-test-modules` narrowing) that a safety-valve should be as narrow as the risk it covers | The workspace-array diff only needs to actually run its filesystem scan when the changed-file set includes a path outside every existing `SERVICES` entry's prefix — i.e. this doesn't add meaningful cost to the common-case PR that only touches already-registered services. For PRs that don't touch anything new, the check is a fast no-op; it is not an "everything matches by default" tax applied to every single PR the way cal.com's pattern would be | Trivial — this is really just "only do the expensive-ish filesystem walk when the cheap path-prefix check already found an unmatched file," which the existing `detect-services.py` loop (lines 79-82) already computes as a side effect (`touched` is false for every `SERVICES` entry on an unmatched file) |

**Net effect versus pick #1 as originally written:** the "new service gets zero coverage" gap still gets
closed — a PR introducing an unregistered directory with a `package.json`/`Dockerfile` can no longer merge
silently uncovered. But the fix is scoped to exactly the failure mode that motivated it (a genuinely new,
unregistered service directory), not every file in the repo that happens not to be a `.md`/`.vscode` file.
PRs that only touch already-registered services pay zero extra cost. This is a closer match to "correctly
detect what changed and confirm it builds" than either extreme: it is stricter than today's silent gap, and
cheaper/narrower than cal.com's blanket default-direction flip.

**What was considered and not recommended:**

- **CODEOWNERS-based detection** — checked as a possible third source of truth per the task's instructions.
  Not pursued further: CODEOWNERS maps paths to *reviewers*, not to "does this path have a registered
  build/test mechanism." A path could have a CODEOWNERS entry (for review routing) without ever having a
  `package.json`/`Dockerfile`, and vice versa — it answers a different question than the one this gap needs
  answered, and this repo's `CODEOWNERS` file (if any) wasn't found to contain build-relevant metadata worth
  reusing. `yarn workspaces list` plus a filesystem scan for `package.json`/`Dockerfile` is both more direct
  and already proven as ground truth in this exact repo (`TEMP-PLAN.md` §3).
- **Running the existing 23-service matrix as the fallback for unmatched files** — this is literally what
  cal.com's pattern would reduce to in this repo's architecture, and the "Reconciling" section above already
  shows why it fails to close the actual gap (new service still wouldn't be in the matrix) while still
  paying the broad, "heavy and slow" cost the person running this project was worried about. Not recommended
  for either reason.

## What survives untouched from `3-recommended-adoptions.md`

Nothing else in that doc is in question here — these are restated only to confirm they aren't affected by
the above, not re-litigated:

- **Pick #2 / #2a — parse real manifests into a dependency graph for the shared `packages/*` consumers gap**
  (`conflict-types`, `calculated-fields`, `mcp-auth-core`), via langchain's `dependents_graph()` pattern, with
  langchain's "it's OK to hand-special-case your one worst-offender package" fallback. Unchanged — this is a
  different gap (existing shared packages' consumers) from the one reconsidered here (brand-new, unregistered
  service directories). Worth noting the new registration check above and this pick are complementary, not
  overlapping: the registration check catches "forgot to add the service at all"; the dependents graph catches
  "the service is registered, but a shared package it depends on changed and it wasn't retested."
- **Pick #3 — consolidate repeated install steps into one composite action** (`uv_setup`-style, per
  `oss/langchain-analysis.md` §7). Unchanged, pure refactor.
- **Pick #4 — skip the expensive Docker tier while a PR is in draft** (Vault's `is-draft == 'false'` pattern).
  Unchanged.
- **Pick #5 — a small, repo-wide manifest-drift check** (`check_extras_sync.yml` pattern, catching the kind of
  bug `StrapiMCP`'s `yarn.lock`/`package.json` drift already demonstrated). Unchanged, and arguably now a
  closer sibling to the revised pick #1 above than it was before — both are "does our declared metadata match
  reality" checks, just checking different metadata (lockfile consistency vs. service registration).
- **Items 6-9 (home-grown PR-stack delay mechanism, adopting Graphite, three-way trust split, e2e tiering)** —
  correctly out of scope per the original doc, unaffected by this reconsideration.

## Sources

- `/Users/0xjustuzair/Desktop/keytoz/databonder-enterprise-search/.github/scripts/detect-services.py` (full
  file read directly from the repo, not from docs describing it — `SERVICES` table lines 41-66, matching
  loop lines 79-94)
- `/Users/0xjustuzair/Desktop/keytoz/databonder-enterprise-search/.github/workflows/pr-gate.yml` (full file
  read directly from the repo — confirms `fast-checks`/`docker-build` both consume the enumerated JSON
  arrays from `detect`, lines 33-40, 110-117, i.e. the matrix really is built only from `SERVICES` entries,
  not a generic repo-wide command)
- `ci-gate-research/TEMP-PLAN.md` §"Workspace inventory" and §"`ruleengine/*`, `StrapiMCP`,
  `packages/document-preview` are not real Yarn workspaces" — source for the 19-registered-vs-4-standalone
  split and the confirmed `yarn workspaces list` behavior
- `ci-gate-research/v2/oss/calcom-analysis.md` §2 (`pr.yml`'s `has-files-requiring-all-checks` filter) — the
  pattern being reconsidered
- `ci-gate-research/v2/oss/langchain-analysis.md` §2 (`dependents_graph()`), §7 (`check_extras_sync.yml`) —
  "parse real manifests, don't hand-maintain a shadow list" and "fail loudly and specifically" precedents
- `ci-gate-research/v2/oss/vault-analysis.md` §2 (`check-go`'s module-coverage-completeness step) — the
  "coverage-completeness check, not an affected-selection mechanism" precedent
- `ci-gate-research/v2/3-recommended-adoptions.md` — the doc being reconciled with
