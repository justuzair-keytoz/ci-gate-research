# CI Gate Research

Research, decisions, and build history behind the CI gate work for `databonder-enterprise-search`
(branch `DATA-338-CI-Gate`, PR [#327](https://github.com/Bharat-Tech-Labs/Enterprise-Search/pull/327)).

This repo is maintained standalone for now, separate from the main codebase, while the design is still
being reviewed and iterated on. Once the approach is finalized, the relevant parts of this will be
written up properly and added to `docs/` in the main repo.

## Start here

- **[`v1/PLAN.md`](v1/PLAN.md)** — round one: the gate itself (change detection, fast checks, Docker
  validation, the aggregating gate job), built and proven on real CI runs.
- **[`v2/PLAN-V2.md`](v2/PLAN-V2.md)** — round two: stacked-PR safety, dependents-aware detection, build
  caching, and the bugs found and fixed along the way. Every decision cites the exact commit and the
  exact Actions run that proves it.

Everything else in `v1/` and `v2/` is the research and reasoning that led to those two documents, kept
for anyone who wants to see the full trail rather than just the outcome.

## Layout

```
v1/   round one — original research (no repo access), repo-specific findings, the final plan
v2/   round two — PR-stack research, per-directory test-dispatch research, 4 real OSS case studies,
      a caching deep-dive, and the final plan
```

Each round's own scratch/working notes (`TEMP-PLAN.md`, `TEMP-PLAN-V2.md`) live inside that round's
folder, not at the repo root — they're the pre-build decision record, kept for context, not the
deliverable itself.
