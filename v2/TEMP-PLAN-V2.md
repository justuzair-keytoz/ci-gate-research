# CI Gate V2 — Scratchpad (decisions + research at hand, pre-implementation)

Status: draft, pre-POC. Same role `v1/TEMP-PLAN.md` played before `v1/PLAN.md` was written — this is
where the agreed-upon decisions and the research backing them get pinned down *before* touching code.
Once the POC below is built and verified against real GitHub Actions runs, the actual `PLAN-V2.md`
(the handoff doc) gets written from this, same as before. This doc is allowed to be messier/more detailed
than that one; it's the working record, not the deliverable.

Branch: `DATA-338-CI-Gate`. Same PR #327 used as the live test harness as in V1 — same discipline: push a
real fixture, watch a real Actions run, revert, keep the fix.

All V1 research (the original no-repo-access docs, the repo-specific `TEMP-PLAN.md`/`PLAN.md`, and the
hybrid gate they produced) now lives under `v1/`. All the follow-up research that led to this build
(PR-stacking, per-directory dispatch, five OSS case studies, the reconsidered synthesis, the stacked-CI
strategy survey) lives under `v2/`. This file is the next layer: turning that research into actual code.

## What V2 is building, and why each piece earns its place

Three changes, chosen specifically because each closes a **demonstrated** gap (not a hypothetical one)
found during research, and none of them cost anything beyond engineering time — no new paid tool, no new
required infrastructure:

1. **Dependents-aware change detection** — `packages/conflict-types`, `calculated-fields`, and
   `mcp-auth-core` each have real consumers (`UI`, `Temporal`, `Strapi`, `rules-service`, etc.) that the
   V1 gate would not retest if only the shared package itself changed. Confirmed with a real `grep`
   against this repo's own `package.json` files (`v2/2-per-dir-test-dispatch-research.md`).
2. **Service-registration completeness check** — V1's `detect-services.py` only knows about a service if
   it's in the hardcoded `SERVICES` table. A new service directory gets zero gate coverage until someone
   remembers to add it. Found by contrast with `calcom/cal.com`'s opposite default
   (`v2/oss/calcom-analysis.md`), then re-designed in `v2/4-reconsidered-synthesis.md` to fit this repo's
   enumerated-matrix architecture instead of literally copying cal.com's pattern (which wouldn't have
   closed the gap here — see that doc for why).
3. **Stack-foundation safety check** — this repo stacks PRs heavily already (27 of 49 open, up to 8 deep,
   `v2/1-pr-stack-research.md`), and nothing today checks whether a stacked PR's parent branch is itself
   sound before this PR's own gate can go green. Modeled on real merge-safety logic found in `spr` and
   `git-spice`'s own source (`v2/5-stacked-ci-strategy-survey.md`) — built as a **safety** check (is the
   foundation proven, yes/no), not a cost-optimization scheme (no polling, no delay logic, two cheap API
   calls maximum).

Explicitly **not** building (per direct instruction, after reconsideration): any CI-cost-reduction
mechanism for stacks (predecessor-polling, leaf-only, bottom+top hybrid). That family of ideas solves a
different problem (wasted CI minutes) this repo doesn't have yet at its current scale, and costs real
engineering/maintenance surface to build. If redundant-CI cost across deep stacks becomes a measured
problem later, `v2/5-stacked-ci-strategy-survey.md` has the comparison ready to revisit.

---

## Build log

(Filled in as each piece lands — implementation details, what was tested, what broke, what the real
Actions runs showed.)
