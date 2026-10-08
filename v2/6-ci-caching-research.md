# v2 Research: CI Build Caching for `fast-checks`

Status: research + implementation, done and measured. Unlike the other `v2/*.md` docs, this one covers
work that's already built, tested, and partly reverted based on real measurements — not just a
recommendation. Triggered by a direct observation: `fast-checks (ui)` was taking 6-7 minutes, and the
person running this project pointed at Next.js's own official CI caching guide
(https://nextjs.org/docs/pages/guides/ci-build-caching#github-actions) and asked whether we'd done
anything like it. We hadn't — this is the full story of what got added, what was measured, and why one
layer got kept and another got removed.

## Starting point: zero caching in `fast-checks`

Docker builds already cached (`type=gha`, proven in V1 with a real cold/warm test: 57s → 28s). But
`fast-checks` had nothing — every matrix leg re-downloaded every package from scratch, every single run,
and `UI`'s `.next/cache` (Next.js's own incremental build cache) was never persisted either.

## Layer 1: Yarn download cache + `.next/cache` — kept, measurably faster

Adapted Next.js's own guide for Yarn (the guide is npm-specific):

- `actions/cache` on `~/.yarn/berry/cache` (confirmed via `yarn config get cacheFolder` — Yarn Berry's
  real global cache location, not npm's `~/.npm`), keyed on every `yarn.lock` in the repo, OS-only
  restore-keys fallback.
- `actions/cache` on `UI/.next/cache`, gated to the `ui` matrix leg only (nothing else has a `.next`
  dir), keyed on `yarn.lock` + UI source file hashes, with a yarn.lock-only restore-keys fallback — same
  two-tier key shape as the Next.js doc's own example.

**Measured, real cold/warm comparison** (same methodology as V1's Docker cache proof):
- Cold: 7m19s (439s) — both caches miss, nothing to restore yet.
- Warm: 6m15s (375s) — **~14.5% faster**. Yarn cache hit exactly (480MB restored in ~12s instead of a
  fresh download). `.next/cache` hit via the restore-keys fallback specifically (source hash changed
  between the two fixture pushes, yarn.lock hash didn't — exactly the "rebuild from the closest prior
  cache" behavior the Next.js doc describes).

Honest framing: the improvement is real but modest, not dramatic. `next build`'s type-checking and
page-analysis phases still run regardless of a warm `.next/cache` — the cache mainly helps
webpack/SWC module-compilation reuse, not full build wall-time.

## Layer 2: deeper caching (`node_modules` + Yarn's "install state") — tried, measured, reverted

Went looking for more improvements specifically in the OSS repos already cloned from earlier research.
Found `calcom/cal.com`'s own `.github/actions/yarn-install/action.yml` — a hand-rolled Yarn Berry caching
setup (their own comment cites https://github.com/actions/setup-node/issues/325 as the reason they don't
use `setup-node`'s built-in `cache: yarn` input, same reason this repo's caching is hand-rolled too). It
caches three things, not just one:

1. The downloaded archive (what Layer 1 above already does).
2. `**/node_modules` itself, keyed on the lockfile — skips the whole *link* step on a hit, not just
   re-downloading.
3. Yarn's own "install state" file (`YARN_INSTALL_STATE_PATH`) — skips re-resolving the dependency tree
   entirely when the lockfile hasn't changed.

Also found, separately, in Next.js's own actual CI config (not their public guide) — a one-line comment
on their pnpm-store cache step: **"Do not use restore-keys since it leads to indefinite growth of the
cache."** Relevant because GHA's cache budget is a shared 10GB/repo pool (established in V1 research) —
noted as the reason the two new caches below were built exact-key-only, no restore-keys.

Implemented both of cal.com's extra layers, plus their `YARN_NM_MODE: hardlinks-local` setting (smaller,
faster `node_modules` writes via hardlinks instead of copies — a separate, independent optimization with
no claimed downside).

**Measured, real cold/warm comparison:**
- Cold: 9m31s (571s) — **slower than Layer 1's cold run (439s)**, because saving the new `node_modules`
  cache alone took ~2.5 minutes (it's a ~487MB blob).
- Warm: 6m52s (412s) — **slower than Layer 1's warm run (375s)**, even though both new caches hit
  correctly. The `Install (immutable)` step itself was genuinely fast this time (~33s) — but restoring
  the ~487MB `node_modules` cache cost ~50s on its own, and that overhead wasn't offset by what the
  install step saved.

**Why it didn't pay off here, when it works for cal.com:** this gate always runs `yarn install
--immutable` in full, every time — that's the deliberate dependency-drift check that caught the real
`StrapiMCP` `yarn.lock`/`package.json` bug earlier, and it isn't something to skip for speed. cal.com's
own action, by contrast, can skip `yarn install` entirely on a cache hit for some jobs, and doesn't use
`--immutable` at all (their own comment: disabled "so it doesn't try to remove our private submodule
deps"). Their install step on a hit does much less real work than ours does, so caching `node_modules`
saves them more than it saves us. Given our constraint (keep `--immutable` active, always), the
download-cache alone already captured most of the available speed-up; the extra `node_modules`/
install-state layers cost more (cache restore + save time for a large blob) than they returned.

**Decision: reverted both.** Kept `hardlinks-local` (independent, no measured downside) and the
`actions/cache@v4` → `v6` version bump (routine currency, same reasoning as the Node 24 runtime fix —
unrelated to whether the extra cache layers stayed).

## Other things checked, not adopted

- **Turborepo Remote Cache** (`TURBO_TOKEN`/`TURBO_TEAM`, used throughout cal.com's workflows) — a
  hosted, paid-beyond-free-tier third-party service (Vercel). Noted as the other big caching lever real
  projects use, not recommended here — conflicts directly with staying cost-efficient and not depending
  on paid third-party infrastructure.
- **`cache-checkout`** (cal.com caches the entire checked-out working tree per-commit, so multiple jobs
  in the same run skip re-cloning) — a same-run, cross-job optimization. Not adopted: at this repo's
  current size, a fresh `actions/checkout` is a small fraction of total run time; the complexity isn't
  justified yet.
- **`cache-build`** (cal.com caches the finished build *output* itself and skips running `yarn build`
  entirely on an exact content match, rather than just speeding the build up) — a more aggressive
  variant than what's built here. Not adopted now: this repo's own change-detection (`detect-services.py`)
  already scopes builds tightly to what actually changed, so the marginal value of also skipping a build
  on an exact-content match is smaller than it would be for a repo with coarser change detection (which
  is part of why cal.com leans harder on this). Worth revisiting if `UI`'s build time becomes a
  bottleneck on its own merits later.

## Net result

`fast-checks` now has working, measured, kept caching (Yarn download cache + `.next/cache`,
~14.5% faster warm vs. cold) plus one free independent optimization (`hardlinks-local`) plus a routine
`actions/cache` version bump. A deeper caching attempt was tried, measured honestly, found to be a net
loss given this repo's correctness constraints, and reverted rather than kept for its own sake.

## Sources

- Next.js's public CI build-caching guide: https://nextjs.org/docs/pages/guides/ci-build-caching#github-actions
- `tmp/cal.com/.github/actions/yarn-install/action.yml` (full file read)
- `tmp/cal.com/.github/actions/cache-checkout/action.yml`, `cache-build/action.yml`,
  `cache-build-key/action.yml` (full files read)
- `tmp/next.js/.github/workflows/build_and_test.yml` lines ~200-215 (pnpm-store cache step, the
  "do not use restore-keys" comment)
- `https://github.com/actions/setup-node/issues/325` (cited by cal.com's own action as the reason for
  hand-rolling Yarn Berry caching instead of `setup-node`'s built-in `cache: yarn` input)
- Real GitHub Actions runs on this repo's own `pr-gate.yml` (commits `a99fc459`, `df0381aa`, `d64e1029`
  and the fixture/revert commits around them) — all timing numbers above are measured, not estimated.
