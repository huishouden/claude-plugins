---
name: developing-an-app
description: >
  Playbook for working on any Huishouden household app or repo (portal, Tasks, Spending, Pet, Home,
  Car, Bills, Baby, Health, Groceries, the kit, rules, the Workers) or creating a new one: names,
  design language, logos, the shared kit, the Firebase projects and household model, sign-in,
  Hosting bandwidth, where personal data may live, deploys, staging, the new-app checklist. Use in
  any session that touches a repo in the huishouden org, a *.web.app site of the suite, or a
  household PWA, and before creating a new household app.
---

# Huishouden: how every app is built

Huishouden is a suite of small installable web apps (PWAs) for one household, used on a shared
wall tablet and on phones. The apps must feel like one product. The sources of truth live in the
public kit — read them, don't restate them from memory:

- Design language, names and logos: https://github.com/huishouden/pwa-kit/blob/main/DESIGN.md
- Engineering standard: https://github.com/huishouden/pwa-kit/blob/main/STANDARD.md
- Household specifics (project, sites, who deploys rules): https://github.com/huishouden/portal/blob/main/STANDARDS.md
- Pull requests: the `pr-lifecycle` skill in this plugin (draft → `hh dev review` → `hh dev verify`/`evidence` → `hh dev ready` → merge). The `hh` CLI: the `hh` skill. Operations and secrets: the `ops` skill.

## The suite

Every app is served from **one site**, `https://huishouden-piekstra.web.app/`, under its own path,
so the installed portal opens every app with no browser bar and one sign-in covers them all
(pwa-kit `docs/one-site.md`). The old `huishouden-<app>.web.app` addresses 301 to the path.
The address is set once, `SUITE_SITE` in pwa-kit `src/site.ts`: code never writes the host out.
Use `SUITE_ORIGIN` / `suiteUrl(base, path)` from `@huishouden/pwa-kit/site` where there is no page
(tests, Playwright defaults, fallbacks), `appUrl(import.meta.env.BASE_URL, …)` in the browser;
`pwaApp({ base })` and `pwa.yml` derive link previews and the smoke URL themselves (no `url`, no
`site-url`). Moving the suite: kit docs/one-site.md "Moving the suite".

| App | Repo | Path |
|---|---|---|
| Huishouden (portal: Today, Calendar, Contacts, app tiles, household, sign-in) | `huishouden/portal` | `/` |
| Tasks, Home, Pet, Car, Bills, Spending, Baby | `huishouden/<app>` | `/<app>/` |
| Shared kit | `huishouden/pwa-kit` (`@huishouden/pwa-kit`) | — |
| Push sender (Cloudflare Worker) | `huishouden/notify` | — |
| AI assistant connector, calendar feed (Cloudflare Workers) | `huishouden/connector`, `huishouden/calendar` | — |
| The `hh` CLI | `huishouden/cli` | — |
| Code reviewers | `huishouden/cr-reviewers` | — |
| This plugin | `huishouden/claude-plugins` | — |
| Firestore rules and indexes | `huishouden/rules` | — |

The list of apps, with each path, is the portal's `apps.json`.

All repos are public (free CI). One Firebase project, `huishouden-piekstra`, on the free plan with
no billing account; keep it that way unless the user approves spending.

**Hosting bandwidth is the scarcest resource.** Spark serves 10 GB a month for the whole suite
and stops serving at the limit (October 2026 reached 9.2 GB by day 4, from CI and agents loading
production in browsers). Never open production in a browser: not in Playwright, not by hand to
"check it works". Browser tests and screenshots run against a local preview or a staging site
(`BASE_URL`; the apps' Playwright configs default to the staging suite). To check production,
`curl -sI` one URL. A browser visit costs the app's whole precache (0.4 to 0.5 MB compressed)
per fresh context. Kit docs/one-site.md "Bandwidth".

## What Huishouden is

A product for **any household**, free, self-serve; the user's own household dogfoods it. Never
propose "works on my machine / my account" solutions: no dependency on the user's Mac, personal
CLIs, GitHub secrets or Apps Script tied to their Sheet. External connectivity runs as each
household's own Google account in the browser (or in shared infrastructure that serves every
household alike). Free tiers only; anything needing billing is raised as a decision with its cost.
Full text: pwa-kit STANDARD.md, "Who it's for".

## Rules the user set (non-negotiable)

1. **Names:** suite "Huishouden"; apps named for what they do in plain English ("Spending",
   "Tasks"); full name "Huishouden <App>"; repos `huishouden/<app>` in the huishouden org, served at `/<app>/` on the one site. No per-app brand names.
2. **One design language** (DESIGN.md): cream page, white cards, forest green primary, terracotta
   for attention, Inter, no gradients/glass/emoji/off-palette colours. `bunx pwa-design-check`
   enforces the mechanical part in CI; the `huishouden/design-language` cr reviewer judges the rest.
3. **Logos:** `bunx pwa-logo <glyph>` (forest tile, cream house, terracotta roof, one glyph), then
   `bunx pwa-icons --background=#1b4332`.
4. **Shared building blocks over copies:** if two apps need it, it goes in huishouden-pwa-kit
   (Vite preset, firebase/auth/household/e2e/logo modules, reusable `pwa.yml` CI, leak scan,
   bootstrap). Gaps in tools become CLIs/kit features, not one-off scripts. **Every time an app is
   added or grows a feature, compare it with the other apps and extract anything duplicated or
   generic into the kit** (proposals in the kit's `docs/proposals/`).
5. **Dynamic, not hard-coded:** household facts (people, cards, category rules, alert labels,
   app list) are data — Firestore or the household Sheet's tabs — never code or fixtures.
6. **Public by default, so no personal data in any repo**: every Huishouden repo is built to be
   public from its first commit. No personal data including fixtures, sample data, comments and commit
   messages: invent data from scratch; never transform real rows. Well-known national chains and
   services are fine in sample data (recognisable, not sensitive); keep people, contact details,
   account numbers, local businesses and store numbers invented. Before publishing a repo: fresh
   single-commit repo (never force-push to clean), cross-check every real description/amount, and
   an independent auditor agent — see the `repo-to-public` skill.
11. **Enforce with reviewers:** when a rule matters, add a check — CI for what code can decide, and
    a specialized `cr` reviewer in the org's own repo `huishouden/cr-reviewers`
    (`.codereview/agents/huishouden/`, cloned outside app checkouts to `~/Dev/huishouden-cr-reviewers`
    and added with `cr config agent-source add <clone>/.codereview/agents --profile <profile>`) for
    judgment: `huishouden:docs-sync` (README, screenshots, kit docs, `apps.json`, privacy
    page, org profile and these skills move with the change), `huishouden:design-language`,
    `huishouden:i18n`, `huishouden:roles-privacy`, `huishouden:public-data`, `huishouden:shared-kit`.
    A rule that changes updates the reviewer that cites it.
7. **Leak protection everywhere:** the kit leak scan in CI, the pre-commit hook
   (`git config core.hooksPath .githooks`), secret scanning + push protection on. Rules describe
   patterns; never list real values to block.
8. **Every PR follows `pr-lifecycle`:** a draft first, `hh dev review` to the bar, evidence
   (`hh dev verify` or `hh dev evidence`), `hh dev ready` (mandatory), then merge. Conventional
   title: CI versions from it on merge; a PR carries no version or CHANGELOG.
9. **Verify in a real browser yourself** (Playwright headless, against a local preview or staging,
   never production) before telling the user it works; extend the tests when a bug is found by hand.
10. **General guidance lives in general places:** when the user gives a rule that applies beyond one
    app, put it in these skills (huishouden/claude-plugins), DESIGN.md/STANDARD.md — not just in one repo or chat.
12. **Quick, and use location:** adding takes one field and one tap; extra fields only where they fit
    the item and folded otherwise (DESIGN.md principle 5). Use location wherever it saves a step;
    asking for precise or approximate location is fine, in context (DESIGN.md principle 6).

## Shared platform facts

- **Household:** `households/{id}` in Firestore with `members` (lowercase emails) and `joined`;
  each app keeps data in subcollections (`households/{id}/spendingTransactions`, tasks' lists...).
  Members are managed in the portal's Household panel. Use `@huishouden/pwa-kit/household`.
- **Firestore rules:** the project has one rules file, in its own repo `huishouden/rules` (moved
  from Tasks on 2026-10-02). Every app sends its `match /households/{householdId}/<collection>`
  block there as a PR, with emulator tests next to the others (`bun run test`; needs Java 21:
  `JAVA_HOME=/opt/homebrew/opt/openjdk@21`). Merging to main deploys rules and indexes.
- **Sign-in:** Firebase Auth with Google; `authDomain` is `huishouden-piekstra.firebaseapp.com`
  (the only redirect the auto-created OAuth client allows). Silent sign-in via
  `@huishouden/pwa-kit/auth` (needs the site origin, `https://huishouden-piekstra.web.app`, on that OAuth client).
  Each project's OAuth client lists only its suite site and `<project>.firebaseapp.com` (staging:
  `https://huishouden-staging.web.app`), never per-app sites: Google allows an unverified app 10
  authorized domains and each `*.web.app` counts (kit docs/one-site.md "Sign-in origins"). The service
  worker must not answer `/__/` paths (the kit preset handles it).
- **New app checklist:**
  1. Create `huishouden/<app>` (public, the `huishouden` org) from the kit templates
     (`templates/ci.yml`, `templates/playwright.config.ts`, `templates/firebase.json`, githooks),
     with a repo description "Huishouden <App>: <what it does>".
  2. Add it to the portal's `apps.json` with `"path": "/<app>/"`; `bun run bootstrap` in the portal
     (the user runs it: IAM bindings).
  3. Build with `pwaApp({ base: '/<app>/' })` and `pwa.yml` `base: /<app>/`; generate its logo;
     DESIGN.md from the first commit. No package version or CHANGELOG: CI tags releases.
  4. Its Firestore block in `huishouden/rules`, with emulator tests.
  5. **Update the org profile and repo description**: a row in huishouden/.github
     `profile/README.md`; check with `hh ops profile-check` (`--fix` opens the PR).
- **One site, in code:** links between apps are same-origin paths (`/`, `/pet/`), never absolute
  addresses; anything stored or sent (agenda and reminder `url`, invitations) uses
  `appUrl(import.meta.env.BASE_URL, ...)` from `@huishouden/pwa-kit/site`. Specs navigate with
  relative paths (`./`, `?tab=x`): `BASE_URL` is the app's path, a leading `/` is the portal.
  Every app shares localStorage and IndexedDB: prefix new localStorage keys with the app's name
  unless sharing is the point.
- **Deploys:** only `main` runs hosted CI (pull requests run nothing). `pwa.yml` (keyless) refuses
  tags and releases the merge first (next semver from the Conventional Commit titles; nothing
  committed back; chore/docs/test/ci alone release nothing), builds with that `vX.Y.Z` and the
  commit SHA embedded, publishes the build as the release's `hosting` asset, assembles the whole
  site from every app's latest asset, deploys it and checks the live path over HTTP (no browser). No schedules: no reconcile, Renovate or digest. README
  screenshots come from the author's evidence run, never production.
- **Staging:** a second Firebase project, `huishouden-staging` (free, invented data only, its own
  10 GB of Hosting). Each app has a staging site, `huishouden-staging-<app>.web.app`, holding the
  whole suite with the app's build under `/<app>/` (the portal's is `huishouden-staging.web.app`).
  `hh dev evidence --staging` builds the branch against staging, deploys it there with your
  `firebase` login, runs `e2e` and `e2e:signed-in` (households of the run's own,
  `useTestHousehold`; `pwa-staging run` gets the staging token through gcloud impersonation) and
  takes the screenshots. Signed-in tests use `hh.signIn(page, 'helper')` / `signInTestUser`, never a
  Google account. Gmail, Calendar and Contacts consent stays manual; by hand with Google use
  `https://huishouden-staging.web.app/<app>/` (One Tap fails on per-app staging sites). Kit
  STANDARD.md "Staging"; provisioning: `bootstrap.sh --staging`.
