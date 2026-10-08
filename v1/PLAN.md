# CI Gate — Plan & POC Results

Author: Uzair Saiyed. Status: POC done, ready for review.
Branch: `DATA-338-CI-Gate`. Draft PR: [#327](https://github.com/Bharat-Tech-Labs/Enterprise-Search/pull/327).

Don't merge PR #327. It's a test rig. I pushed broken code to it on purpose, watched the checks catch
the breakage, then undid the broken code. The PR stays open only so you can click through and see the
real runs for yourself.

A full list of copy-paste commands to verify every claim in this doc yourself is at the bottom.

## The problem, in two sentences

Right now nobody checks a pull request before it merges into `main`. People just merge on review, by
eye. The history shows this bites us specifically around Docker: 12 broken-Dockerfile commits in six
months, because the thing we actually deploy is a Docker image, not just the raw code.

## What I built

A GitHub Actions workflow that runs automatically on every pull request. It does two things:

1. **Fast checks** — build, lint, and run tests, but only for the parts of the code the PR actually
   touched.
2. **Docker build checks** — build a real Docker image for any service the PR touched, to catch the
   exact kind of breakage that review-by-eye keeps missing.

Both run in parallel, and a final job looks at both results and decides pass or fail. Only that final
job needs to be the "required" check in GitHub — this matters because of a known GitHub quirk: if you
mark individual pieces of a dynamic checklist as required, and some of them get skipped, GitHub can get
stuck waiting on a check that will never run. One final job that watches everything else avoids that
trap.

## What I actually proved, not just claimed

I didn't just write the workflow and assume it works. I pushed real broken code to the draft PR, on
purpose, and watched it fail for real in GitHub's own UI. Then I fixed it and watched it pass. Every one
of these is a real run you can click on:

- A PR opens → the check runs by itself. [See it here](https://github.com/Bharat-Tech-Labs/Enterprise-Search/actions/runs/37325754996).
- A PR imports a package that was never added to `package.json` → the check fails. [See it here](https://github.com/Bharat-Tech-Labs/Enterprise-Search/actions/runs/37327305904).
- A PR breaks a Dockerfile → the check fails, and only for that one service. [See it here](https://github.com/Bharat-Tech-Labs/Enterprise-Search/actions/runs/37332593446).
- Nothing gets touched by a PR → the check still passes cleanly, it doesn't hang. [See it here](https://github.com/Bharat-Tech-Labs/Enterprise-Search/actions/runs/37335852964).
- A PR touches two unrelated services → only those two get checked, nothing else runs. [See it here](https://github.com/Bharat-Tech-Labs/Enterprise-Search/actions/runs/37335138891).

After each test I removed the broken code and pushed the fix, so the branch right now only contains the
two real files that make up the gate. No leftover test junk.

## Along the way, I found five real bugs in my own workflow — and fixed all five

I'm listing these because they're exactly the kind of thing that looks fine on paper and then breaks
two weeks into real use. Each one got caught by actually running the thing, not by reading the code
twice.

1. I hardcoded one setting (which part of a Dockerfile to build) the same way for every service. Two
   services needed a different setting. Would have silently broken the gate for them.
2. Four folders in this repo look like normal workspaces but aren't actually registered as one in Yarn's
   setup. My first version assumed they all were, so it failed with "workspace not found." Fixed by
   handling those four differently.
3. I guessed one service (`BullBoard`) didn't need special cloud credentials to build. Wrong — it does,
   same as thirteen others. Found out because the real test run failed with an authorization error.
4. One service's "lint" command is actually a tool meant to run before a commit on your own machine, not
   in a clean CI checkout. It failed loudly the first time I ran it for real.
5. Two services expect to be built from their own folder, not from the root of the repo, unlike every
   other service. Docker build hit "file not found" the first time, which is how I caught it. (While
   checking this, I also noticed the ten worker services have a different, separate problem: their
   Docker setup expects a build to have already happened before Docker even starts. Didn't fix this one
   — all ten are blocked by the cloud-credential issue below anyway, so it doesn't matter yet. Leaving a
   note for whoever unblocks them next.)

While fixing #5, I also found something that isn't my bug at all — it's a real, pre-existing problem in
the repo. One service's lockfile doesn't match its own package file. Nobody noticed, because this
service has simply never been checked by any automation before. I left it alone rather than quietly
fixing someone else's file as a side effect.

## Four more things I checked, with real numbers

- **Does the system actually cancel old runs when you push a new commit fast?** Yes — caught it
  happening by accident during testing. [Real example here](https://github.com/Bharat-Tech-Labs/Enterprise-Search/actions/runs/37326793628).
- **Does anyone open PRs from an outside fork, which would break secrets access?** Checked all 49
  currently open PRs through GitHub's own API. Zero are from forks.
- **Does the Docker build cache actually save time, or is it just there for show?** Built the same
  Dockerfile twice. First time: 57 seconds. Second time, cache warm: 28 seconds. Cut the time in half.
- **If someone changes a file at the root of the repo that everything depends on, does the check
  correctly test everything instead of guessing wrong?** Yes — pushed a harmless one-line change to the
  root `package.json` and watched it schedule checks for all 23 affected parts of the code plus all 4
  unblocked Docker builds, exactly as it should.

## Things the original brief asked about, now answered

- **Do the existing tests even pass today?** Yes. I ran every single one — 235 tests across 7 parts of
  the codebase. All green. This removes the main reason anyone had for being nervous about turning this
  check on right away.
- **The brief pointed at PR #297 as an example of a missing-dependency bug.** That PR already merged,
  and the dependency is already correctly in place. I used a safer stand-in test instead, in a small
  library package that nothing depends on, so nothing real could break if something went wrong.
- **What fraction of the 49 currently open PRs would actually fail this check?** Can't know yet. That
  only becomes answerable once this gate is live and can be pointed at those PRs directly.
- **Who's going to own this long-term?** Not something I can answer. Needs a name from the team.

## The one real blocker, and it's not mine to fix

Most of the Docker images in this repo (14 out of 18, including the main app) pull their starting point
from a private, locked-down image storage location. Building them at all needs cloud credentials — which
goes against what the original plan assumed (no cloud credentials during pull request checks).

I checked this two separate ways and both agree: it's real. I did not use or test with the live
credentials I found sitting in a `.env` file, since that file was flagged as off-limits partway through
this work.

There are three ways to handle it, and the choice isn't mine to make:

1. Give the pull-request checks a narrow, read-only credential that can only fetch that one image. Still
   technically "cloud credentials during a PR," just a much smaller risk than what deploy already uses.
2. Copy those starting images somewhere public once, so pulling them never needs credentials again. Costs
   someone time up front, and again whenever the image updates.
3. Skip Docker checks for those 14 services for now. Not really an option — the main app is one of the
   14, and catching exactly this kind of break is the whole reason this project exists.

Right now, the Docker build check works fully and for real on the 4 services that don't have this
problem. The mechanism is proven. It just can't reach the other 14 until someone picks one of the three
options above.

## Still open, and it's your call

- **Turn this check on for everyone immediately, or run it quietly first and see how many PRs would
  actually fail?** The original plan said turn it on immediately. The main reason to hesitate — maybe the
  tests are already broken — turned out to be false. But we still don't know how many of today's 49 open
  PRs would fail. Both choices are reasonable. I'm not picking one for you.
- One service (`Strapi`) has test files sitting there that nothing ever runs. A one-line fix, but it
  touches a file outside the gate itself, so I'm flagging it instead of just doing it.
- GitHub gives a shared, limited amount of storage for caching across all these builds. Probably fine at
  our current pace, worth watching once this is live.
- Still nobody named to own this long-term.

## What I deliberately did not touch

End-to-end browser tests, the Python services (no test tooling exists for them yet, that's a separate
job), anything in the actual deploy workflow, and turning the check into a hard requirement on `main`
(that's an instant, repo-wide switch — not something to flip without you saying so first).

## An idea for later: build the image once, reuse it, don't rebuild it

Right now, the check builds a Docker image just to test it, then throws it away. Separately, whenever
someone actually deploys, a brand new image gets built from scratch at that moment — which could be
weeks after the PR merged. Same instructions, two different build events. In theory they could still
come out slightly different.

The stronger way to do this: build the image exactly once, label it with the commit it came from, and
have deploy simply reuse that exact image instead of building a new one. Nothing can drift, because
there's only ever one image.

I didn't build this part, on purpose. It needs the check to actually push the image somewhere permanent
(today it doesn't push anywhere at all, by design), and it needs a change to the actual deploy workflow,
which the brief specifically said to leave alone. Writing the idea down now so it isn't lost, but it
needs its own separate approval before anyone builds it.

## What happens next, if this gets approved

1. Decide how to handle the cloud-credential blocker above.
2. Decide: turn the check on as required right away, or watch it run quietly for a bit first.
3. Once the credential question is settled, switch on Docker checks for the remaining 14 services — the
   code to do it already exists, it's one setting flipped per service.
4. Add the missing test command to `Strapi`.
5. Run the check against today's real open PRs, see how many would actually fail, then make the check
   required on `main`.

## Want to double-check any of this yourself? Here's exactly how

Everything above links to a real GitHub Actions run or a real commit. Here are the same checks as
copy-paste commands, for anyone who'd rather verify from a terminal than click around. Run these from
inside the repo.

See the actual diff this POC produces (should be exactly 2 files):
```
git diff main...DATA-338-CI-Gate --stat
```

See every fixture that was pushed and reverted (fixtures say `poc fixture:`, real fixes say `fix:`):
```
git log --oneline main..DATA-338-CI-Gate
```

Check the current state of PR #327 (should show the check passing, nothing left broken):
```
gh pr checks 327
```

Look at any run mentioned above in detail (swap in any run ID from this doc):
```
gh run view 37332593446 --json jobs --jq '.jobs[] | {name, conclusion}'
```

Check the cancel-on-new-push proof specifically:
```
gh run view 37326793628 --json jobs --jq '.jobs[] | {name, conclusion}'
```

Check the cache-speed numbers, down to the second:
```
gh run view 37423716902 --json jobs --jq '.jobs[] | select(.name | startswith("docker-build")) | {name, startedAt, completedAt}'
gh run view 37424071732 --json jobs --jq '.jobs[] | select(.name | startswith("docker-build")) | {name, startedAt, completedAt}'
```

Check the "root file change selects everything" proof:
```
gh run view 37424581138 --json jobs --jq '[.jobs[].name]'
```

Check there are zero fork PRs right now:
```
gh pr list --limit 50 --state open --json number,isCrossRepository --jq '[.[] | select(.isCrossRepository)] | length'
```

Check the cloud-credential blocker is real:
```
docker pull 759325905697.dkr.ecr.us-east-1.amazonaws.com/node:20.12.2-alpine3.18
```
(should say access denied / no credentials)

Check that 4 folders are missing from the real Yarn workspace list:
```
yarn workspaces list
```
(won't show ruleengine/rules-service, ruleengine/rules-studio, StrapiMCP, or packages/document-preview)

Check PR #297's real state:
```
gh pr view 297 --json state,mergedAt
grep '"@remixicon/react"' UI/package.json
```
(should show it's already merged, and the dependency is already there)

Re-run every test in the repo:
```
corepack enable && yarn install --immutable
for ws in databonder-mcp Temporal packages/amendment-resolver packages/calculated-fields packages/conflict-types packages/mcp-auth-core; do
  echo "== $ws =="; (cd "$ws" && yarn test)
done
(cd Strapi && node --import tsx --test src/utils/*.test.ts)
```
