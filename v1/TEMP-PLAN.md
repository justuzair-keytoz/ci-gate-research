# CI Gate — TEMP PLAN (working draft, not for review)

Status: draft. This is the scratchpad where repo-specific findings get pinned down before anything
goes in `PLAN.md`. `PLAN.md` doesn't exist yet on purpose — it gets written after the POC runs and
proves (or disproves) what's below.

Everything in `2. Ci-Gate-Research.md` and `3. Ci-Gate-Recommendation.md` was written by an agent with
**no access to this repo** — it's general industry research, tagged with confidence levels. Everything
below was checked directly against `databonder-enterprise-search` on branch `DATA-338-CI-Gate`
(branched from `main` @ `51f16494`). Where repo fact contradicts the earlier doc's guess, repo fact wins.

## 1. Answers to the brief's open questions

### "Do the 33 existing tests pass today?"

**Yes — all of them, and there are more than 33.** Ran every test script that exists in the repo today:

| Workspace | Tests | Result |
|---|---|---|
| `databonder-mcp` | 31 | all pass |
| `Temporal` | 21 | all pass |
| `packages/amendment-resolver` | 40 | all pass |
| `packages/calculated-fields` | 99 | all pass |
| `packages/conflict-types` | 6 | all pass |
| `packages/mcp-auth-core` | 26 | all pass |
| `Strapi/src/utils/*.test.ts` (no `test` script wired in `package.json`, ran directly) | 12 | all pass |

Total: 235 tests, 0 failures. The brief's "33 existing tests" undercounts — that was probably a file
count, not a test count, and it missed Strapi's 5 test files entirely (they exist but nothing runs
them — no `test` script in `Strapi/package.json`). **Action item for POC: add a `test` script to
Strapi's `package.json`, same pattern as the other workspaces (`node --import tsx --test ...`).**

This clears the brief's hard-cut blocker: nothing is red on main today. No "fix broken tests before the
gate can land" work needed.

### "What is the real failure rate across the 25 open PRs?"

Not yet measured — needs the actual POC workflow to exist and run against each open branch via
`workflow_dispatch` with a ref input (can't retroactively trigger `pull_request` on old PRs — GitHub
only runs a workflow added mid-PR on new events on that PR, confirmed in the earlier research doc §10.1).
This happens after the POC workflow exists, not before. Currently 49 open PRs (brief said 25; backlog
grew since the brief was written — `DATA-340`, `DATA-261` series, a few others landed since).

### "Who owns CI?"

Not something I can answer — that's a team/people decision, needs your senior or lead to name someone.
Flagging as a sign-off item, not researching further.

## 2. Open questions from the no-repo-access doc, now answered

| # | Question | Answer |
|---|---|---|
| 1 | Is Turborepo or Nx used anywhere? | No. No `turbo.json`, no `nx.json` anywhere in the repo. Confirms the "don't adopt a build system just for this" recommendation — nothing to build on top of. |
| 2 | Is `UI/Dockerfile` single- or multi-stage? | Multi-stage (`builder` → `production`). Confirms `mode=max` GHA cache is the right call (caches all stages, not just the final image). |
| 3 | Is GHCR in use, or would a registry cache be new infra? | Neither — deploy pushes to **AWS ECR** (`aws-actions/amazon-ecr-login`), not GHCR. No GHCR presence at all. Registry-backed cache would mean standing up new infra (GHCR or ECR); `type=gha` avoids that entirely, and avoids AWS creds in PR runs per A3. |
| 4 | What Yarn version, and is `changesetBaseRefs` set? | Yarn **4.6.0** (`"packageManager"` / `.yarnrc.yml` confirms). No `changesetBaseRefs` configured — default base ref behavior applies, confirm against `main` explicitly in the workflow rather than relying on defaults. |
| 5 | How does the private npm token reach the Docker build? | `UI/Dockerfile` takes `GITHUB_TOKEN` as a build **arg**, which the deploy workflow maps from the `PACKAGE_REGISTRY_TOKEN` secret (`--build-arg GITHUB_TOKEN=${{ secrets.PACKAGE_REGISTRY_TOKEN }}`). `.yarnrc.yml` reads `${GITHUB_TOKEN}` for the `bharat-tech-labs` npm scope. The gate's Docker build has to pass this same build-arg or any PR touching UI fails to install private deps — not a Dockerfile bug, a gate bug. **This is a real gotcha to get right in the POC.** |
| 6 | Will fork PRs stay absent? | Confirmed via `gh pr list` — every open PR's `headRefName` is a plain branch name, consistent with "everyone pushes branches directly," matching assumption A3. Not verified for 100% of history, but no contrary evidence. |
| 7 | Is touching the deploy workflow acceptable? | Not yet asked — still a sign-off item (see §5). POC does **not** touch any `build-*.yml` or `deploy-web.yml`. |
| 8 | Rollout shape? | Still open — brief says hard cut, the no-access research doc recommended observe-first. Repeating as a sign-off item since the "33 tests are red" risk (the main reason to hesitate on hard-cut) turned out to be false. That weakens the case for observe-first specifically on test-redness grounds, but doesn't resolve the "what's the real PR failure rate" question, which still needs the POC to exist first. |

## 3. Repo facts that change the design (vs. the no-access doc's assumptions)

### The PR #297 fixture doesn't reproduce as stated

The brief says PR #297 "ships `@remixicon/react` imports without the dependency." Checked: PR #297
is **already merged** into `main` (it's the tip commit, `51f16494`), and `@remixicon/react` **is** in
`UI/package.json` on main right now. Either it was fixed during review before merge, or the brief's
snapshot was taken mid-review. Either way, this specific PR can't be used as the fixture anymore —
need a synthetic fixture branch instead (see §4).

Verified the *mechanism* still works, though, two ways:

1. Removed `@remixicon/react` from `UI/package.json` and ran `yarn install --immutable` — it failed
   immediately (`"The lockfile would have been modified by this install, which is explicitly
   forbidden"`). Any package.json/lockfile drift is caught before a single line of app code runs.
2. The deeper case — someone writes `import ... from "new-pkg"` and never touches `package.json` or
   `yarn.lock` at all (so nothing's "inconsistent," the package is just never installed) — isn't
   caught by `--immutable`, it's caught downstream: `tsc`/`next build` fails to resolve the import
   because node-modules-linked Yarn won't have the package on disk. **No separate dependency-check
   tool needed** (no `depcheck`, no custom script) — `yarn install --immutable` + the existing
   `build`/`lint` scripts already cover both failure shapes for free. This simplifies the "fast tier"
   design from the no-access doc, which left dependency-checking as an open design question.

### Workspace inventory (ground truth, for the matrix)

19 Node/TS workspaces with their own `package.json`, all buildable via `yarn workspace <name> run
build`:

`UI`, `Strapi`, `BullBoard`, `databonder-mcp`, `Temporal`, `StrapiMCP`, `workers/{gmail,google-drive,
one-drive,outlook,slack,smartsheet,teams,upload-file,vector-store,workflow-engine}`,
`packages/{amendment-resolver,calculated-fields,conflict-types,document-preview,mcp-auth-core}`,
`ruleengine/{rules-service,rules-studio}`.

Of those, `test` scripts exist only on: `databonder-mcp`, `Temporal`, `packages/amendment-resolver`,
`packages/calculated-fields`, `packages/conflict-types`, `packages/mcp-auth-core`. The rest (`UI`,
`Strapi`, workers, `ruleengine/*`, `packages/document-preview`) have no unit tests wired — fast tier
for those is build + lint only, which is correct and not a gap to fix as part of this task (brief
scopes unit-test gating to what exists, doesn't ask us to write new tests).

`lint` scripts exist on: `UI`, `BullBoard` (via `lint-staged`, needs care — that's meant for staged
files in a pre-commit hook, not a clean CI checkout; **don't reuse it as-is in the gate**). Most
workspaces have no `lint` script at all — fast tier for those is build-only (tsc via `build`, since
`build` is `tsc` directly for most of them).

Dockerfiles exist for 19 services total, including two **Python** ones (`RAG/Dockerfile`,
`Parser/Dockerfile`) that are explicitly out of scope per the brief (no pytest/lint tooling exists
yet). POC Docker tier covers the 18 non-Python Dockerfiles; RAG/Parser stay untouched.

### ⚠️ Base images for most Dockerfiles live in a *private* ECR registry — breaks the "no AWS creds" assumption

This is the single biggest finding from the POC build and directly contradicts A3 ("gate builds images but
never pushes to ECR — no AWS creds in PR runs") and the no-access research doc's recommendation (§6.5,
"`type=gha` + `push: false` needs no AWS access").

Checked every Dockerfile's `FROM` line. Two different patterns exist:

| Base image source | Needs AWS creds to even `docker pull`? | Services |
|---|---|---|
| `759325905697.dkr.ecr.us-east-1.amazonaws.com/node:...` (private ECR, confirmed by a real anonymous `docker pull` attempt → `pull access denied ... no basic auth credentials`, and independently by a live POC run hitting `401 Unauthorized` on `BullBoard`'s build) | **Yes** | `UI`, `Strapi`, `BullBoard`, `databonder-mcp`, `workers/{gmail,google-drive,one-drive,outlook,slack,smartsheet,teams,upload-file,workflow-engine,vector-store}` — **14 of 18** Dockerfile-having services |
| Public Docker Hub (`node:20.12.2-slim`, `node:22-alpine3.20`) or `public.ecr.aws` | No | `Temporal`, `StrapiMCP`, `ruleengine/rules-service`, `ruleengine/rules-studio` — 4 services |

(`BullBoard` was first miscategorized as unblocked in an earlier pass of this doc — its `FROM` line *does* use the private ECR registry, just like 11 others. Caught when the POC's `docker-build` job actually hit a live `401 Unauthorized` pulling it. Worth noting as a reminder that this whole ECR-blocked list should be re-verified against the real Dockerfiles before anyone treats it as final, not just read off this table.)

**Why this matters:** `UI` — the service with the actual demonstrated failure history (12 Dockerfile
commits in 6 months, the entire reason this gate exists) — is in the "needs AWS creds" bucket. A Docker
build-validation tier that honors A3 as written literally cannot build the one Dockerfile the brief is
built around. `push: false` + `load: true` avoids needing creds to *push*, but says nothing about the
*pull* — and nobody flagged that distinction until it was checked against the real Dockerfiles.

**Options, not yet decided:**

1. **Scoped read-only pull credentials via OIDC** — a narrow IAM role with only `ecr:GetAuthorizationToken`
   + pull on that one base-image repo, assumed via GitHub's OIDC provider (no long-lived keys stored as
   secrets). Still "AWS creds in PR runs," technically violates A3's letter, but is a materially smaller
   exposure than the deploy workflow's full push-capable credentials.
2. **Mirror the base images to GHCR** (or make the ECR repo's relevant tags public) — a one-time infra
   task outside this gate's scope, after which the gate pulls from GHCR/public ECR with zero AWS creds,
   preserving A3 exactly. Costs someone's time to set up and keep in sync when the base image updates.
3. **Skip Docker-build validation for the 11 affected services** — not viable, defeats the stated purpose
   for the majority of the fleet, and specifically for `UI`.

This needs a decision from whoever owns AWS/ECR access before the gate's Docker tier can be real for
more than 4 of 18 services. Flagging in `PLAN.md` as a blocking question, not picking an answer here.

### `ruleengine/*`, `StrapiMCP`, `packages/document-preview` are not real Yarn workspaces

`yarn workspaces list` only returns 19 entries. `ruleengine/rules-service`, `ruleengine/rules-studio`,
`StrapiMCP`, and `packages/document-preview` are missing from it entirely — confirmed directly, not
assumed. They have their own `package.json` and live inside the monorepo folder tree, but the root
`workspaces` array in `package.json` never lists `ruleengine/*`, and the `"StrapiMCP/*"` glob matches
*subdirectories of* `StrapiMCP`, not `StrapiMCP` itself. Their Dockerfiles confirm this independently —
`ruleengine/rules-service/Dockerfile` and `ruleengine/rules-studio/Dockerfile` both use `npm ci`/`npm
install`, not `yarn`, and carry a comment `# pattern from Web-App/apps/rules-service/Dockerfile`,
suggesting these were ported in from a different repo and never fully integrated into this one's Yarn
workspace graph.

**This breaks `yarn workspace <name> run build`** as a universal fast-checks mechanism — it only works
for the 19 registered workspaces, and errors with `Usage Error: Workspace 'X' not found` for the other
four. Found this the hard way: the POC's `rules-service` fixture PR failed `fast-checks` with exactly
that error before the fix. **Fixed** by giving each service a `standalone` flag in
`detect-services.py` — standalone services get their own `cd <path> && yarn/npm install` step (using
whichever lockfile is actually there — `rules-studio` has no lockfile at all, matching its own
Dockerfile's `npm install`, not `npm ci`) before running their scripts directly, instead of going
through `yarn workspace`.

This also means **`yarn workspaces foreach --since`**, the no-access doc's recommended day-one change
detection mechanism (§3 of `2. Ci-Gate-Research.md`), **would silently never detect changes to these
four directories at all** — they're invisible to the Yarn workspace graph. Good thing the POC used
plain path-prefix matching instead of `--since` for exactly this reason (it only knows about
directories, not Yarn's internal workspace registry) — but worth being explicit that `--since` alone,
as recommended, would have missed a real slice of the repo.

### Dockerfile build targets differ per service (matrix needs a `target` field, not just a path)

- `UI`, `Strapi`, `databonder-mcp`: multi-stage, final stage named **`production`**.
- `ruleengine/rules-service`, `ruleengine/rules-studio`: multi-stage, final stage named **`runner`** (not
  `production` — would silently break a gate that hardcoded `--target production` for everything).
- `BullBoard`, `Temporal`, `StrapiMCP`, all `workers/*`: single-stage, no `--target` flag needed/possible.

### Deploy's real service list (for matrix naming, from `deploy-web.yml`)

`nextjs(UI), strapi, rag, vector-store, google-drive, one-drive, outlook, slack, teams, gmail,
smartsheet, parser, bullboard, postgres, redis, upload-file, workflow-engine, strapi-mcp, temporal,
temporal-ui, temporal-worker, rules-service, rules-studio, databonder-mcp, langfuse-web,
langfuse-worker, clickhouse`

`postgres`, `redis`, `temporal`, `temporal-ui`, `langfuse-web/worker`, `clickhouse` are vendored images
with no Dockerfile in this repo (third-party images deployed as-is) — correctly excluded from any
build-validation tier, nothing to build.

## 4. POC plan (what gets built next, on this branch)

Single new workflow, `.github/workflows/pr-gate.yml`, trigger `pull_request` (no path filters at the
trigger level — per the researched GitHub sharp edge, a workflow skipped by a path filter leaves its
check "Pending" forever; detection has to happen *inside* a workflow that always runs).

1. **`detect` job** — diffs `git diff --name-only $(git merge-base origin/main HEAD)...HEAD`, maps
   changed paths to the 19 service dirs above via a small bash/jq script (not `dorny/paths-filter` —
   skipping a new dependency for ~15 lines of bash; also the earlier research flagged `paths-filter`'s
   supply-chain score as weak). Emits two JSON arrays: node workspaces touched, Dockerfiles touched.
2. **`fast-checks` job** — dynamic matrix over the node-workspaces array. Per leg: `corepack enable`,
   `yarn install --immutable`, then build, then lint/test only if the script exists in that
   workspace's `package.json` (checked with `jq`, not assumed).
3. **`docker-build` job** — dynamic matrix over the Dockerfiles array. Per leg: `docker/build-push-action`
   with `push: false`, `load: true`, `cache-from/to: type=gha,scope=<service>`, passing
   `PACKAGE_REGISTRY_TOKEN` as the `GITHUB_TOKEN` build-arg where the Dockerfile asks for it (not all
   of them do — check per-Dockerfile).
4. **`gate` job** — `needs: [detect, fast-checks, docker-build]`, `if: always()`, inspects `needs.*.result`,
   fails on any `failure`/`cancelled`, passes on `success`/`skipped`. **Only this job becomes the
   required check** — matches the researched aggregating-gate pattern, avoids the skipped-matrix-leaves-
   check-pending trap.

### Fixtures to prove it works (draft PRs on this branch, or local simulation first)

- Missing dependency: re-run the `@remixicon/react` removal as an actual fixture commit (since PR #297
  no longer demonstrates it) — confirm `fast-checks` fails for `UI`.
- Broken Dockerfile: introduce a bad `COPY` path in `UI/Dockerfile` — confirm `docker-build` fails for
  `UI` only, nothing else runs.
- Docs-only change: touch only `docs/`/`README.md` — confirm both matrices come back empty, `gate`
  still reports success (not stuck Pending).
- Multi-service change: touch `UI` and one `workers/*` — confirm only those two run.

## 5. Needs sign-off before this becomes the real `PLAN.md`

- **Rollout: hard cut vs. observe-first.** Test-redness risk is gone (§1), but PR-failure-rate risk
  isn't measured yet. Leaning toward: run the gate unrequired for ~1 week against real PR traffic,
  then flip to required — want your senior's call on whether that week is acceptable or whether hard
  cut from day one (as the brief states) is non-negotiable.
- **Strapi test script.** Adding one is a one-line `package.json` change but it's still an edit to a
  workspace file outside the gate's own files — flagging so it isn't a surprise.
- **GHA cache size.** Shared 10GB/repo budget across 18 Docker builds plus Yarn's own dependency
  cache. Likely fine at current scale (~20 PRs/month per A2) but worth a 2-week watch once live.
- **Who owns this gate long-term (A5).** Still unanswered, not a research question.

## 6. What's explicitly NOT in the POC (matches brief's out-of-scope list)

Playwright/e2e, Python CI for RAG/Parser, any edit to `build-*.yml` or `deploy-web.yml`, merge queue /
auto-merge.
