# v2 Research: Testing Only What Changed, Per Directory

Status: research only, no decisions made, no code changed.

## The question this answers

Your senior's second point, as explained: we are **not** building build-then-promote right now (that's
deferred — see `PLAN.md`'s "Phase 2" section). What's needed instead is confirmation that a branch gets
checked for build errors and tested **only against what changed** between it and its reference branch.
Different directories have different test setups (`UI` has Playwright + Wiremock, other services have
their own tooling), so a single global "test everything" script in the root `package.json` isn't
feasible — it would mean running every suite for every small change. E2E stays out of scope for actual
implementation, but should be documented here, per your explicit instruction.

## Where the current POC already gets this right

The existing `pr-gate.yml` / `detect-services.py` (see `PLAN.md`) already does the core of this
correctly: it diffs changed files against the merge-base, maps them to the specific services touched,
and only runs `build`/`lint`/`test` for those services — never a global script. This was proven live in
the POC (touching `BullBoard` + `packages/mcp-auth-core` only ran those two services' checks, nothing
else). That part doesn't need rework.

## Where it has a real, demonstrated gap: it doesn't know about dependents

Industry research converges hard on one warning, repeated across every source checked: **naive
path-matching ("a file under this folder changed, so test this folder") is blind to anyone who *depends
on* that folder.** The fix requires either a dependency-graph-aware tool (Nx, Turborepo) or a repo that
tracks the graph another way — plain path-matching alone cannot know this.

> "Folder-only selection misses dependents: change `packages/pricing`, skip `apps/checkout` tests, break
> production totals." — paraphrased from current monorepo-testing guidance

**This isn't theoretical for this repo. Checked directly, right now:**

```
grep -rl '"@databonder/conflict-types"' --include=package.json .
→ Temporal/package.json, UI/package.json, ruleengine/rules-service/package.json, Strapi/package.json
```

If a PR only touches `packages/conflict-types/`, the current `detect-services.py` selects **only**
`conflict-types` for testing. It does **not** select `Temporal`, `UI`, `ruleengine/rules-service`, or
`Strapi` — all four of which import this package directly and would silently miss a breaking change to
it. Same gap confirmed for `@databonder/calculated-fields` (consumed by `Temporal`, `UI`,
`packages/amendment-resolver`, `Strapi`) and `@databonder/mcp-auth-core` (consumed by `UI`,
`databonder-mcp`).

This is the single most important finding in this doc: **the existing POC's change detection is real and
working, but it is not dependency-graph-aware, and this repo has at least three shared packages
(`conflict-types`, `calculated-fields`, `mcp-auth-core`) where that gap would currently let a breaking
change through untested.**

## What closes this gap

Two paths, not mutually exclusive:

1. **Adopt a graph-aware tool (Nx or Turborepo `--affected`).** Both build a dependency graph from
   imports/package.json and automatically include dependents when a shared package changes.
   `turbo run test --affected` or `nx affected --target=test --base=main` would, for example, correctly
   also select `Temporal`/`UI`/`rules-service`/`Strapi` when `conflict-types` changes. Cost: adopting a
   build system this repo doesn't currently use (confirmed — no `turbo.json`, no `nx.json` anywhere),
   which earlier research (`2. Ci-Gate-Research.md`) already flagged as "not worth the lift just for this
   gate" — that calculus may need revisiting now that the dependents-gap is concretely demonstrated,
   rather than hypothetical.
2. **Hand-maintain a small dependency map for the handful of shared `packages/*`.** Since this repo only
   has 5 shared packages total (`conflict-types`, `calculated-fields`, `mcp-auth-core`,
   `amendment-resolver`, `document-preview`), it's plausible to hardcode "if `packages/X` changes, also
   select every consumer of `@databonder/X`" directly in `detect-services.py`, without adopting a whole
   new build system. Cheaper, but manual — has to be kept in sync by hand if a new package gains a new
   consumer, with no tooling to catch a missed update. (The general industry guidance's point #7 — "Validate
   the graph itself, don't just assume inference found everything" — applies doubly hard to a hand-maintained
   map, since there's no tool to double-check it against reality.)

Both are implementation decisions, not made here — this doc only establishes that the gap is real and
names the two ways to close it.

## Per-directory test framework differences — the actual ask

Your senior's point: UI uses Playwright + Wiremock, other services use other test setups, and this needs
to be respected rather than forced into one global script. Checked what actually exists today:

| Directory | Test tooling | Scope |
|---|---|---|
| `databonder-mcp`, `Temporal`, `packages/{amendment-resolver,calculated-fields,conflict-types,mcp-auth-core}` | `node --import tsx --test` (Node's built-in test runner) | Unit-level, already wired to a `test` script, already run by the POC's fast-checks tier |
| `UI/tests/` | Playwright (`.spec.ts` files, `tests/dynamic-table-conflict-indicators/`, `tests/demo/`, etc.) | **E2E** — drives a real browser against a running app |
| Wiremock (repo-wide: `docker-compose-wiremock.yaml`) | Stubs real HTTP dependencies so services can be tested without live external APIs | Supports integration-level testing for whichever service needs an external dependency faked |
| `Strapi/src/utils/*.test.ts` | Same Node test runner as above, but **no `test` script wired in `package.json`** — these tests exist and pass but nothing invokes them (already flagged in `PLAN.md`) | Unit-level, currently orphaned |

This confirms the shape of what "respecting each directory's own tooling" already means in practice: the
POC's existing approach — check if a `test` script exists in that service's own `package.json`, run it if
present, skip if not — already *is* the mechanism for "don't force one global test command." No directory
is forced into running another directory's framework. The real remaining question is specifically about
UI's Playwright suite, which is deliberately excluded (see below), not about the unit-level dispatch,
which already works per-directory today.

### Industry patterns for scoping e2e/Playwright to only what changed (documented, not adopted)

Per your instruction, documenting this for the record even though it's out of scope for actual
implementation right now:

- **`--only-changed` flag.** Playwright has a built-in flag: `npx playwright test --only-changed=origin/main`
  runs only the spec files whose corresponding source likely changed, using git history (needs
  `fetch-depth: 0`, same requirement as every other graph-aware tool in this research).
  ([Playwright Docs — Continuous Integration](https://playwright.dev/docs/ci))
- **Tag-based tiering.** Mark a small subset of e2e tests `@smoke` and run only those on every PR;
  reserve the full suite for merge-to-main or nightly — the same tiered-CI principle from the PR-stack
  research applies here too.
- **Path-based mapping.** Diff changed files against `main`, map changed source directories to their
  corresponding spec files by convention (e.g. `UI/src/components/Foo` → `UI/tests/foo.spec.ts`) — this
  is a weaker, convention-dependent version of the dependency-graph approach above.
- **Wiremock, scoped per service.** Current industry pattern for polyglot repos: one Wiremock container
  per mocked external dependency (not one shared container for everything), with a
  `./wiremock/{ServiceName}:/home/wiremock` volume per service — meaning Wiremock stubs can, in
  principle, be spun up selectively for only the service(s) a PR actually touches, rather than booting
  every stub for every PR. This repo's `docker-compose-wiremock.yaml` already exists; whether it's
  already organized this way, or would need restructuring to support "only boot stubs for touched
  services," wasn't checked in this pass — flagging as something to verify before any future e2e-scoping
  work, not something resolved here.

None of this is being implemented now. It's here so that when e2e scoping is revisited, there's a
starting point instead of a blank page.

## Summary: what this means for the gate, concretely (not yet implemented)

- The "don't run a global test script" ask is **already satisfied** by the existing per-service
  `test`-script-if-present dispatch in `detect-services.py` / `pr-gate.yml`.
- The real, demonstrated gap is **dependents of shared packages aren't selected** — confirmed with real
  `grep` evidence against `conflict-types`, `calculated-fields`, and `mcp-auth-core`. This needs a
  decision (adopt Nx/Turborepo, or hand-maintain a small dependency map) before the gate can be trusted
  for PRs that touch shared packages.
- E2E/Playwright/Wiremock scoping is documented above for later, explicitly not part of what runs today.

## Sources

- [Nx Docs — Run only tasks affected by a PR](https://nx.dev/docs/features/ci-features/affected)
- [Turborepo Docs — `run` reference (`--affected`)](https://turborepo.dev/docs/reference/run)
- [Playwright Docs — Continuous Integration](https://playwright.dev/docs/ci)
- [WireMock](https://wiremock.org/)
- This repo's own `grep` output against `package.json` files (see "Where it has a real, demonstrated gap"
  above) — reproducible, not a third-party claim.
