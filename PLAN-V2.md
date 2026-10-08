# CI Gate — Plan V2

Author: Uzair Saiyed. Status: built, tested, ready for review.
Branch: `DATA-338-CI-Gate`. Same PR as before: [#327](https://github.com/Bharat-Tech-Labs/Enterprise-Search/pull/327).
Two more PRs, left open on purpose so you can see the real run history: [#346](https://github.com/Bharat-Tech-Labs/Enterprise-Search/pull/346) and [#347](https://github.com/Bharat-Tech-Labs/Enterprise-Search/pull/347).

This is the second round of work on the CI gate. The first round (`v1/`) built the gate itself and proved
it catches real breaks. This round answers two things your senior asked after reviewing that: how the
gate behaves on stacked PRs, and whether it tests everything that actually needs testing when a shared
piece of code changes. Along the way we also found and fixed some real bugs that had nothing to do with
either question, and chased down a slow build.

Every section below names the exact commit and the exact test run behind it, so none of this has to be
taken on faith.

## How to read this

- `v1/` — the first round: the gate itself, how it was built, what it catches.
- `v2/` — all the research behind this round. Each doc is small and answers one question.
- This file — what we actually built from that research, in order, with the commits and runs that prove
  each piece works.

## 1. A shared package change now tests everyone who actually uses it

**The problem:** the gate only re-tested a service if you touched that service's own folder. If you
changed a shared package like `packages/conflict-types`, the gate had no way of knowing that `UI`,
`Strapi`, `Temporal`, and `rules-service` all import it. We checked this directly — grepped every
`package.json` in the repo — and found four real consumers the gate would silently skip.

**What we did:** `detect-services.py` now reads every workspace's real `package.json` at the moment the
gate runs, and builds a map of "who depends on what" from that. The same idea langchain uses for its own
Python packages, just applied to ours. No separate list to keep updated by hand — it reads the same file
you'd already edit if you added a new dependency.

**Commit:** `dec4b3e3`.

**Proof:** we pushed a one-line comment into `conflict-types` and watched the gate pick it up — [run
37639158295](https://github.com/Bharat-Tech-Labs/Enterprise-Search/actions/runs/37639158295). It
correctly selected all four real consumers, not just the package itself.

## 2. A new service can't quietly skip the gate anymore

**The problem:** the gate only knows about a service if someone adds it to a list in
`detect-services.py`. Add a new folder and forget that step, and it gets zero coverage, forever, and
nobody notices.

**What we considered first, and why we didn't do it:** `cal.com` handles this by flipping the default —
their gate checks everything unless a file is obviously safe to skip (like a markdown file). We looked
closely at copying that, and it wouldn't actually work here. Our gate is built around a specific list of
known services, each with details a generic rule can't guess (which Docker stage to build, which
services need their own separate install). Flipping the default the way cal.com does would mean running
every check on almost every PR, which is exactly the slow, noisy outcome we were trying to avoid — and it
still wouldn't catch a truly new, unlisted service, because there'd be nothing telling the gate how to
build it.

**What we did instead:** a much smaller, cheaper check. Before anything else runs, the gate compares two
things it already has — the real list of registered workspaces (`yarn workspaces list`) and a quick scan
of the actual folders on disk — against that service list. If something exists that isn't on the list, the
gate stops immediately and names the exact folder. No guessing, no broad slowdown, just a loud, specific
failure the moment it's actually needed.

**Commit:** `dec4b3e3` (same commit as above).

**Proof:** tested twice. Once locally, by creating a fake unregistered folder and watching the script
reject it immediately. Once for real, as part of the stack test below — a real PR with a genuinely
unregistered folder failed the gate exactly as designed.

## 3. A stacked PR now checks whether its parent actually passed

This is the one your senior specifically asked about. This team stacks PRs constantly — we checked, and
27 of 49 open PRs at the time of this research were stacked on another branch instead of `main`, with two
chains going 7 and 8 layers deep. Nothing in the gate knew or cared about that.

**What we looked into first:** a lot of research here went into whether the gate should somehow test the
*whole* stack at once, or skip re-testing layers to save time (this is what tools like Graphite do).
We built a full comparison of four different approaches real projects use for this — it's in
`v2/5-stacked-ci-strategy-survey.md` if you want the detail. In the end we didn't build any of them. They
all solve a cost problem (too much CI time spent on a deep stack), and that's not the problem we actually
have. Our real gap was a safety one: nothing checks whether the branch a PR is built on is itself sound.

**What we did:** a new check that runs alongside everything else. If a PR's target branch is itself
another open PR (not `main`), the gate looks up that other PR's own result. If it already failed, this
PR's gate fails too, and says exactly which PR and why. If the other PR hasn't finished yet, this PR
isn't blocked — it just gets a warning. Two quick lookups, nothing more. No waiting, no polling, no new
service running in the background. The idea itself comes from how real stacking tools (`spr`,
`git-spice`) refuse to merge a PR whose foundation hasn't actually passed.

**Commit:** `72a809ea`.

**Proof — this is the one worth clicking into yourself.** We built a real two-PR stack and left it open:
[PR #346](https://github.com/Bharat-Tech-Labs/Enterprise-Search/pull/346) (the base) and
[PR #347](https://github.com/Bharat-Tech-Labs/Enterprise-Search/pull/347) (stacked on top of it). We ran
it through all three outcomes for real:
1. Healthy parent → child passes, citing the parent's result directly —
   [run 37735990089](https://github.com/Bharat-Tech-Labs/Enterprise-Search/actions/runs/37735990089).
2. We broke the parent on purpose (pushed a deliberately unregistered folder to it) → the child correctly
   failed, naming the parent and the reason —
   [run 37736295618](https://github.com/Bharat-Tech-Labs/Enterprise-Search/actions/runs/37736295618).
3. We fixed the parent → the child flipped back to passing —
   [run 37736610384](https://github.com/Bharat-Tech-Labs/Enterprise-Search/actions/runs/37736610384).

Both PRs are still open. You can read the real run history on them yourself.

## 4. A real bug we found by accident: shared packages were never built before the things that use them

This wasn't something we planned to fix — we found it the moment #1 above actually worked for the first
time. The moment the gate correctly selected `UI`, `Strapi`, and `Temporal` for a `conflict-types`
change, all three failed to build. Not because of anything wrong with the change — because the gate never
built `conflict-types` itself first. The real Docker build does this correctly (it builds shared packages
before the app that needs them); the gate's fast checks never did.

**First attempt was wrong.** We built the shared packages in alphabetical order, which broke, because one
of them (`amendment-resolver`) depends on another one (`calculated-fields`) that comes later
alphabetically. Fixed properly by asking Yarn to figure out the correct order itself, instead of guessing
an order by hand.

**Commits:** `dbc5a6ff` (first attempt, wrong order), `c2852cca` (fixed, correct order).

**Proof:** [run 37642875051](https://github.com/Bharat-Tech-Labs/Enterprise-Search/actions/runs/37642875051) — `UI`, `Strapi`, and `Temporal` all built cleanly afterward.

## 5. A separate, pre-existing bug we found and left alone on purpose

While all this was happening, `conflict-types`'s own test command failed — but for a completely
unrelated reason: it depends on exact behavior from a newer Node.js version than what the gate uses. It
passes fine on a newer Node version locally, and fails on the older, pinned CI version. This isn't
something the gate caused, and we didn't touch it — fixing someone else's test file as a side effect of
unrelated work isn't the move. Flagging it here so it's on record, same as we did with a similar lockfile
bug we found in round one.

## 6. GitHub's own deprecation warning, fixed

Noticed while watching these runs: GitHub started warning that several of the actions we use
(`actions/checkout`, `actions/setup-node`) are running on a version of Node that's being phased out.
This is about the tool that runs the action itself, not the Node version our own app builds against,
which is unaffected. Checked each action's current release directly rather than guessing, confirmed they
all support the newer runtime, and bumped all four pinned versions.

**Commit:** `73889189`.

**Proof:** [run 37737221510](https://github.com/Bharat-Tech-Labs/Enterprise-Search/actions/runs/37737221510) — warning gone, and every job still passed with the new versions.

## 7. Caching — one layer kept, one layer tried and honestly reverted

You pointed out `fast-checks` was slow and asked if we'd set up any build caching. We hadn't. Full detail
in `v2/6-ci-caching-research.md` — short version here.

**Layer one, kept.** Docker builds already had caching from round one. The fast checks had none — every
run re-downloaded every package from scratch, and Next.js's own build cache was never saved either. Added
both, adapted from Next.js's own official caching guide (that guide is written for a different package
manager; we adjusted it for the one this repo actually uses).

**Commit:** `a99fc459`.

**Measured, not assumed:** ran the exact same build twice in a row. First time (cold): 7 minutes 19
seconds. Second time (warm): 6 minutes 15 seconds — about 14.5% faster, confirmed by checking the actual
cache hit in the logs, not just the clock. [Cold run](https://github.com/Bharat-Tech-Labs/Enterprise-Search/actions/runs/37738278237), [warm run](https://github.com/Bharat-Tech-Labs/Enterprise-Search/actions/runs/37739081356).

**Layer two, tried and reverted.** Went looking at how `cal.com` handles this same problem, since
they're a similar-sized project on the same package manager. Their setup caches two more things beyond
what we'd already added. We built the same two layers and tested them the same honest way — twice, cold
and warm.

**Commits:** `df0381aa` (added), `d64e1029` (reverted).

**What we measured:** it made things slower, not faster. Cold run: 9 minutes 31 seconds (worse than
before — saving one of the new caches alone took two and a half minutes). Warm run: 6 minutes 52 seconds
(also worse than layer one alone, at 6:15). [Cold run](https://github.com/Bharat-Tech-Labs/Enterprise-Search/actions/runs/37741565732), [warm run](https://github.com/Bharat-Tech-Labs/Enterprise-Search/actions/runs/37742752046).

**Why it didn't work for us, specifically:** cal.com's setup can skip the install step entirely when
their cache hits. Ours can't — the gate always runs a strict install check, on purpose, because that
exact check is what caught a real bug in round one (a lockfile that didn't match its own package file).
Keeping that check means we don't get to skip the expensive part the way cal.com does, so the extra
caching cost more than it saved. We kept it reverted rather than keep something that measured worse just
because someone else's project uses it. One small, free piece from the same research did stay — a Yarn
setting that makes install files slightly smaller and faster to write, with no downside either way.

## What's still open, and whose call it is

- The 14-of-18-Dockerfiles-need-cloud-credentials blocker from round one is unchanged — still needs a
  decision from whoever owns that access.
- Hard-cut vs. observe-first rollout — still an open call, same as before.
- Whether the redundant-CI-cost problem on deep stacks (not the safety problem, the cost one) is ever
  worth solving — `v2/5-stacked-ci-strategy-survey.md` has four real options compared side by side if
  that's revisited later.

## What this round deliberately did not touch

Anything to do with reducing CI cost on stacks (that's a different problem than the safety check we
built — see above). Adopting a paid caching or stacking service. The ECR credential question. Any change
to the actual deploy process.

## Verify any of this yourself

Same approach as round one — everything above links to a real run. If you'd rather check from a
terminal:

```
git log --oneline main..DATA-338-CI-Gate
git show dec4b3e3        # dependents graph + registration check
git show 72a809ea        # stack-foundation safety check
git show a99fc459        # caching, layer one (kept)
git show d64e1029        # caching, layer two (reverted) + why
gh pr checks 327
gh pr checks 346
gh pr checks 347
gh run view <run-id> --json jobs --jq '.jobs[] | {name, conclusion}'
```

## One more thing before this gets reviewed

Plan is to push the whole `ci-gate-research` folder into its own repository so your senior can browse all
of this directly, instead of reading it secondhand. Not done yet — flagging it here since it's the
obvious next step once this doc is approved.
