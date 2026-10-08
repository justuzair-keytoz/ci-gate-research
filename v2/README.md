# CI Gate v2 — Research Index

This covers the two follow-up points from the senior review of round one, plus an OSS research pass, plus
the caching work that came out of watching the gate run. What actually got built from all of this, with
commits and real run links, is in `../PLAN-V2.md` — read that first if you want the outcome. Everything
in this folder is the research and reasoning behind it.

## Start here if short on time

- **`SUMMARY.md`** — key findings from every doc below, one section each, in under 10 minutes.
- **`3-recommended-adoptions.md`** — the first pass at "what would we pick from all this," as tables.
- **`4-reconsidered-synthesis.md`** — a second, independent pass that revisits pick #1 above (the
  cal.com-derived "match everything, exclude a short list" default). Found that pick #1 as written
  wouldn't actually have closed the gap it targeted in this repo's specific architecture (our matrix is
  an *enumerated* list of services, not a generic repo-wide command like cal.com's — flipping the default
  would pay cal.com's "heavy/slow" cost for zero extra coverage). Replaces it with a narrower, cheaper
  "registration check" instead — this is the version that actually got built, see `../PLAN-V2.md`.

## Full docs, in reading order

1. **`1-pr-stack-research.md`** — PR stacking: does the gate test the cumulative stack diff or just the
   immediate parent, and how does end-to-end stack correctness get verified. Real finding: this repo
   already stacks PRs heavily today (27 of 49 open PRs, some 8 layers deep).
2. **`2-per-dir-test-dispatch-research.md`** — testing only what changed, respecting each directory's own
   test tooling, without a global "test everything" script. Real, demonstrated gap: the current POC
   doesn't know that `UI`, `Temporal`, `Strapi`, and `rules-service` all depend on `packages/conflict-types`.
3. **`oss/langchain-analysis.md`**, **`oss/nextjs-analysis.md`**, **`oss/vault-analysis.md`**,
   **`oss/calcom-analysis.md`** — four real, working OSS CI setups analyzed for how they handle both
   questions above, with exact file/line citations.
4. **`oss/awesome-actions-curated-list.md`** — checked a curated Actions list for coverage of either
   question; found none. Documented so nobody else has to re-check it.
5. **`5-stacked-ci-strategy-survey.md`** — a wider follow-up specifically on PR-stack CI cost (not the
   safety question — that's answered and built, see `../PLAN-V2.md`). Compares four real strategies found
   as actual shipped code (`ghstack`, `spr`, `git-spice`, plus two real PRs from `pomerium` and `metabase`)
   rather than designing around a single example.
6. **`6-ci-caching-research.md`** — caching for `fast-checks`, triggered by a direct observation that it
   was slow. Covers what got built, measured, kept, and — just as importantly — what got tried and
   reverted once the numbers came back worse, not better.

## The short version of each finding

**PR stacks:** GitHub already scopes each PR's diff to its immediate parent by default — no fix needed
there. The real gap was a safety one, not a cost one: nothing checked whether a stacked PR's parent
branch had actually passed. Built and proven live on a real two-PR stack — see `../PLAN-V2.md` section 3.
A *separate* question (redundant CI cost on deep stacks) was researched in depth
(`5-stacked-ci-strategy-survey.md`) but deliberately not built — that's an efficiency problem, not the
safety gap that was actually open.

**Per-directory testing:** the "don't force a global test script" part already worked. The real gap was
that path-matching alone can't see who *depends on* a changed shared package — shown concretely against
this repo's own `conflict-types`, `calculated-fields`, and `mcp-auth-core`, and fixed using langchain's
own approach (parse real manifests, don't hand-maintain a list). Separately, cal.com's exclusion-based
detection default exposed a second gap (new services get zero coverage by default) — closed with a
narrower registration check instead of copying cal.com's pattern directly (see `4-reconsidered-synthesis.md`
for why).

## Status

Done. All of this fed into the actual build — see `../PLAN-V2.md` for what got implemented, which commits,
and which real test runs prove each piece.
