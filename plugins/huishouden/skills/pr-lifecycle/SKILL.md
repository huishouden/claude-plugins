---
name: pr-lifecycle
description: >
  The pull request lifecycle for every huishouden org repo: open as a draft, review with
  `hh dev review`, verify with `hh dev verify` or `hh dev evidence` (staging), then `hh dev ready`
  (mandatory) before marking ready or merging. No version or CHANGELOG in a PR: CI tags and
  releases on merge. Use whenever you open, update, review, mark ready or merge a PR in a
  huishouden repo, and when asking why a version or release appeared (or didn't).
---

# Pull requests in the huishouden org

Pull requests run **no hosted CI**. The author (you) owns everything before `main`; `main` builds,
tests, deploys and checks the live site over HTTP, and CI versions it. The `hh` CLI does each step
and keeps itself current (the `hh` skill). The mental model: **start a draft, run `hh dev ready`,
merge.** Source of truth: pwa-kit STANDARD.md "Pull requests" and "Versions".

## The steps, in order

1. **Branch and commit.** The PR title (the squash-merge commit) is a Conventional Commit
   (`feat(scope): …`, `fix: …`, `feat!:`): it sets the release CI makes (feat minor,
   fix/perf/refactor patch, `!` or `BREAKING CHANGE:` major; chore/docs/test/ci alone release
   nothing). The pre-commit hook runs the leak scan
   (`git config core.hooksPath .githooks` once per clone).
   **A change to user-editable data carries its household tool** (developing-an-app rule 13):
   the kit tool, the connector bump and the `hh data` command are PRs in the same change set,
   merged in this order: kit, rules (when they change), connector, cli (developing-an-app rule 13). A household task an agent was asked to do
   with no tool for it is the same: build the tool, then do the task with it.
2. **Open a draft**: `gh pr create --draft`, conventional title, body with what changed and why.
3. **Review while draft**: `hh dev review`. It pulls `huishouden/cr-reviewers` (clone at
   `~/Dev/huishouden-cr-reviewers`), registers it on the `reviewer` profile, and runs
   `CR_CLAUDE_FOREGROUND=1 cr review <PR> --profile reviewer --max-agents 8 --json` (with
   `--fresh-session` on a PR's first review after the reviewers change), one review at a time on
   the machine. Address every finding: fix it, or answer it in the thread with evidence, then
   resolve the thread. Push and run `hh dev review` again. **Bar: no Blocking or Major open.** A
   finding posted only in the review body (no thread) stays open until a review of a later head
   no longer reports it; every push needs a review of the new head (hh asks cr with `--rerun` when
   cr would keep its older review). A clean review (0 findings) posts nothing as the reviewer, so
   `hh dev review` also records its result for the head commit: one PR comment by you, updated in
   place (`hh review: <SHA> — 0 Blocking, 0 Major (N Minor) · cr run <id>`, marker
   `<!-- hh-review sha=… -->`), and a file under `~/.cache/hh/reviews/`. It records nothing when cr
   wrote no rollup, and warns when your gh login is neither the PR author nor the reviewer. Run
   it after the last push.
4. **Verify**:
   - `hh dev evidence` picks the mode from the changed paths: **staging** for rules, Workers,
     sign-in, Google, notifications or Hosting config; **local** otherwise (`--staging`/`--local`
     override). Local: install, lint, the kit checks, unit tests, build, screenshots on a local
     preview, emulator tests. Staging: the same plus the branch deployed to the app's staging site
     with your `firebase` login, smoke and signed-in tests there.
   - Screenshots: phone and tablet, light and dark, from the app's `screenshots` script.
   - It posts or updates **one** PR comment (marker `<!-- hh-evidence sha=… result=… -->`) with the
     step table and the images, uploaded to GitHub's user-attachments. Re-run after every push:
     evidence is per commit.
   - `hh dev verify` runs the local checks without posting, while you work.
   - **Never production.** No Playwright, screenshots or manual browser checks against
     `huishouden-piekstra.web.app`: Hosting has 10 GB a month for the whole suite. `curl -sI` if you
     must look at production.
5. **No version step.** A PR carries no version bump, CHANGELOG.md section or built output;
   `hh dev release` is a no-op that says so. On merge to `main` CI computes the next semver from
   the commit titles since the last `v*` tag, creates the annotated tag and a GitHub release with
   generated notes and, for packages that ship code (the kit, hh), the tarball attached. Nothing
   is committed back to `main`. Apps get this from the kit's `pwa.yml`; the Workers and the rules
   repo call its `tag-release.yml` after their tests, before their deploy. Apps embed the tag in
   the build (About screen, New Relic); `package.json` stays `0.0.0`. The one exception is the
   huishouden/claude-plugins repo: installed plugins refresh only when `plugin.json`'s `version`
   changes, so its PRs bump it and add a CHANGELOG.md entry (its README).
6. **Ready**: `hh dev ready`. **Mandatory before `gh pr ready` and before merging.** It refuses
   unless: the reviewer account reviewed the head commit with no Blocking/Major thread open (when it
   did not review, the `hh-review` marker for the full head SHA with 0 Blocking and 0 Major,
   authored by the PR author or the reviewer, stands in; a marker for an older head or by anyone
   else is ignored); the
   evidence comment is for the head commit and passed. If the kit was behind it first bumps it
   (see "Keep the kit current") and redoes review and evidence for the new head. On success it
   runs `gh pr ready`. Never mark a
   PR ready or merge it by hand around this check.
7. **Merge** (squash, conventional title). `main` builds, tests, releases and deploys.
8. **Upload the app's assets (app repos).** Apps hold no Cloudflare token; once the app's own deploy has
   finished on main, run `hh ops deploy-site`. It dispatches the portal's `ci` with `reconcile: true`
   (which uploads the new build to the asset CDN, then redeploys the site) and watches it. A no-op
   when the site is current.

## Keep the kit current (automatic)

`hh dev verify|evidence|review|ready` in an app repo first checks the kit's latest release. When
the repo is behind, hh commits `chore: kit vX.Y.Z` on the current branch (the package pin,
`bun.lock`, and the kit workflow and action refs pinned by commit as `@<sha> # vX.Y.Z`), pushes it, and prints what changed, so the PR
carries it. `--no-bump-kit` skips it. It never runs on `main`, never over uncommitted edits to
`package.json`, `bun.lock` or the workflows, and a failure only warns. `hh dev bump-kit` does the
same by hand (plus lint and tests). Kit workflows and actions are pinned by commit SHA with the
version in a trailing comment, never by tag (tags are movable); a pin whose comment names the
target is current, and a pin is never turned back into a tag.

## hh updates itself

Before any command, hh installs a newer release (checked at most every 6 hours) and re-runs the
command on it. Not in CI, with `HH_NO_AUTO_UPDATE=1`, offline, or from a source checkout; a failed
update is a warning.

## What not to do

- No household data change without its tool: no new collection, field or setting that `hh data` and
  the connector can't reach, unless the kit's `COVERAGE` lists it as `internal` or `gap` with the reason.
  Never hand-write production Firestore in place of a missing tool.

- No `pull_request` triggers, schedules, bots or comment-posting jobs in workflows: process work is
  the author's, done with `hh`.
- No release-please, no Renovate, no version or CHANGELOG commits (CI tags), no README screenshot commits from CI.
- No browser against production, ever (the kit's `pwa-bandwidth-check` fails workflows that try).
