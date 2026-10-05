---
name: pr-lifecycle
description: >
  The pull request lifecycle for every huishouden org repo: open as a draft, review with
  `hh dev review`, verify with `hh dev verify` or `hh dev evidence` (staging), set the version and
  CHANGELOG.md with `hh dev release`, then `hh dev ready` (mandatory) before marking ready or
  merging. Use whenever you open, update, review, mark ready or merge a PR in a huishouden repo, and
  whenever main's build fails with "Version not bumped".
---

# Pull requests in the huishouden org

Pull requests run **no hosted CI**. The author (you) owns everything before `main`; `main` builds,
tests, deploys, checks the live site over HTTP and tags the version. The `hh` CLI does each step
(install: `bun add -g github:huishouden/cli#v1`; the `hh` skill). Source of truth: pwa-kit STANDARD.md "Pull requests" and "Versions".

## The steps, in order

1. **Branch and commit** with Conventional Commit messages (`feat(scope): …`, `fix: …`): they set
   the version bump and the CHANGELOG.md lines. The pre-commit hook runs the leak scan
   (`git config core.hooksPath .githooks` once per clone).
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
   it after the last push (`hh dev release --commit` moves the head).
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
5. **Version and changelog**: `hh dev release --commit`. It bumps package.json from main's version
   (feat → minor, fix → patch, `!`/BREAKING CHANGE → major; 0.x: minor) using the commits since the
   last tag, and writes the CHANGELOG.md section. Re-run after rebasing onto a main that released
   meanwhile (it replaces its own section). Docs-only PRs need no bump. Push.
6. **Ready**: `hh dev ready`. **Mandatory before `gh pr ready` and before merging.** It refuses
   unless: the reviewer account reviewed the head commit with no Blocking/Major thread open (when it
   did not review, the `hh-review` marker for the full head SHA with 0 Blocking and 0 Major,
   authored by the PR author or the reviewer, stands in; a marker for an older head or by anyone
   else is ignored); the
   evidence comment is for the head commit and passed; the version is above main's with its
   CHANGELOG.md section (or the change is docs only). On success it runs `gh pr ready`. Never mark a
   PR ready or merge it by hand around this check.
7. **Merge** (squash). `main` refuses to deploy if the version wasn't bumped; on success it tags
   `v<version>` and publishes the section as the GitHub release.

## When main says "Version not bumped"

Someone merged code without `hh dev release`. Nothing deploys until a PR bumps it: on a branch from
main, `hh dev release --commit`, push, draft PR, `hh dev review`, `hh dev evidence`, `hh dev ready`,
merge. The section covers everything since the last tag.

## Keep the kit current

In any repo you touch, `hh dev bump-kit` (moves `@huishouden/pwa-kit` to the latest tag, installs,
lints, tests) and include it in the PR. Nothing bumps dependencies on a schedule.

## What not to do

- No `pull_request` triggers, schedules, bots or comment-posting jobs in workflows: process work is
  the author's, done with `hh`.
- No release-please, no Renovate, no README screenshot commits from CI.
- No browser against production, ever (the kit's `pwa-bandwidth-check` fails workflows that try).
