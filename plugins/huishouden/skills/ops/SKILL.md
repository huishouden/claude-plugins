---
name: ops
description: >
  Operating the Huishouden suite: the Firebase projects (production huishouden-piekstra, staging
  huishouden-staging) and their Spark limits, Hosting bandwidth, OAuth origins, the Cloudflare
  Workers (notify, connector, calendar), New Relic monitoring, the org profile, and handling
  secrets. Use when changing infrastructure, provisioning or checking a project, rotating or
  setting a secret, diagnosing production, or when Firebase/Cloudflare/New Relic/GitHub settings
  come up.
---

# Operating Huishouden

## Firebase

| Project | Use | Plan |
|---|---|---|
| `huishouden-piekstra` | Production: the suite at `https://huishouden-piekstra.web.app` (one site, every app under its path), Firestore, Auth | Spark, no billing |
| `huishouden-staging` | Staging: per-app sites `huishouden-staging-<app>.web.app`, the staging suite `huishouden-staging.web.app`, invented data | Spark |

Limits that bite (Spark, per project):
- **Hosting: 10 GB transferred a month**; at the limit Hosting stops serving until the 1st. Budget
  (kit docs/one-site.md "Bandwidth"): households well under 1 GB; each deploy's HTTP smoke about
  4 KB; New Relic pings about 0.1 GB. A browser visit costs an app's precache, 0.4 to 0.5 MB.
  **Never open production in a browser** (tests, screenshots, manual checks): `curl -sI` instead.
  Usage: Cloud Monitoring `firebasehosting.googleapis.com/network/monthly_sent` (one query; do not
  poll anything that can prompt for credentials).
- Firestore: 50,000 reads, 20,000 writes, 20,000 deletes a day. Staging's are shared by every
  app's test runs (`pwa-staging quota`). Reads are the limit that bites: the Workers' daily shares
  (`FIRESTORE_*_READS` in each `wrangler.toml`) follow kit STANDARD.md "Read budgets" (Workers 28,000 a
  day, the apps the rest; the Workers' sum stays at or under 30,000), and Cloud Monitoring emails "Huishouden budget alerts" at 5,000 billed reads in an
  hour and 30,000 in 24 hours. The policies are code (kit `infra/read-alert.sh`); `hh ops
  read-alerts` applies them (`--print` shows them; a policy that matches the live one is left alone,
  since an edit re-opens a firing policy's incident and emails again: don't re-apply them to "check"). When one fires: the Workers log each run's
  `reads` (calendar `work`/`round`/`tick`, notify `NotifyRun.reads`/`dayReads`, the connector's
  `tool` lines), and `firestore.googleapis.com/api/request_count` by method shows which kind of read.
- Hashed `/assets/*` are `immutable` for a year; HTML, `sw.js` and manifests `no-cache`; Hosting
  serves Brotli/gzip. `pwa.yml`'s smoke checks these on every deploy.

Deploys: only `main`'s `pwa.yml` deploys production (keyless, Workload Identity Federation).
`bun run bootstrap` in the portal grants deploy access and provisions repos; the user runs it.
Firestore rules deploy only from `huishouden/rules` main.

## Sign-in origins

Each project's OAuth client lists only its suite site and `<project>.firebaseapp.com`; Google allows
an unverified app 10 domains. `hh ops oauth-check` probes both projects' origins and the auth
handler redirect URI (`https://<project>.firebaseapp.com/__/auth/handler`); `pwa-oauth-origins`
does the origins in the smoke job. Adding one is a Cloud Console click for the user (Google has no
API). `hh ops auth-domains` compares Firebase Auth's authorized domains with what each project
should have; the kit bootstrap adds missing ones (`--prune-domains` removes extras).

## Workers (Cloudflare, free plan)

`notify` (Web Push every 5 minutes: product, keep its schedule), `connector` (MCP server for AI
assistants, acts as the person via the portal's /connect hand-off; also `hh login`'s
`/cli/hand-off` and `/cli/token`), `calendar` (feed and Google
sync, card alerts). Each deploys staging then production from `main` with `CLOUDFLARE_API_TOKEN`.
Worker secrets live in Cloudflare (`wrangler secret put`), never in the repo or Actions.
Every Worker reads Firestore through the kit's `ReadBudget`: a cap per run, its share of the day
(calendar's in D1 `meta`, notify's and the connector's in a SQLite Durable Object), and a 15-minute
pause for every run after Firestore answers 429.

## New Relic

Free tier: Browser apps, a ping monitor per app path (every 30 minutes, two locations), alerts and a
dashboard, provisioned by the kit's `infra/newrelic.ts` from the portal's `monitoring` workflow
(runs when `apps.json` changes, or by hand). Pings are HTTP only, no browser. `hh ops monitoring
[--dry-run]` dispatches it, waits and summarizes; a skipped provisioning step means the
`NEW_RELIC_API_KEY` or `ALERT_EMAIL` secret is missing.

## GitHub

- Org `huishouden`; every repo public, PolyForm Shield (`actions/license-check`).
- Hosted CI on `main` only; no schedules, no Renovate, no release-please (pr-lifecycle skill).
- Org profile `huishouden/.github` `profile/README.md`: a row per app and Worker, and every repo
  has a description. `hh ops profile-check` (`--fix` opens a PR) when apps or Workers change.
- The Renovate GitHub App, if still installed, is the user's to uninstall (org settings,
  GitHub Apps).

## Secrets

- **Never paste a secret in chat.** The user copies it, runs `hh ops secret store <name>`, then
  `hh ops secret push <name>`. **Agents never read keychain values** (no `security
  find-generic-password -w`, `secret-tool lookup`, `op read`, `pbpaste`), never print, log, or pass
  a secret in argv (`ps` shows argv). An agent asks the user to copy and store; hh does the rest
  in-process.
- `hh ops secret store <name>`: the clipboard, validated (cloudflare: `cfut_`, 40+ characters,
  accepted by Cloudflare; newrelic-user: `NRAK-`; other names non-empty), saved to the OS keychain
  (cloudflare: service `huishouden-cloudflare-api-token`, account `huishouden`), clipboard cleared.
- `hh ops secret push <name> [--repos a,b] [--env production]`: keychain to GitHub secrets on
  stdin, with companions (cloudflare also sets `CLOUDFLARE_ACCOUNT_ID`). Defaults: cloudflare to
  environment `production` (main only) of portal, connector, notify, calendar. No app repo holds it.
- `hh ops secret where`: which repos and environments hold each known secret (names and updatedAt
  only), flagging unexpected placements (a repo-level Cloudflare token, an app repo) and missing ones.
- `hh ops secret rotate <name>`: the user rolls it in the provider and copies it; hh stores,
  pushes to every default target and checks each was updated. Rolling is only for a leaked or
  expiring token: moving or re-adding one is `store` then `push`.
- `hh ops secret set <repo> <NAME> < file` still sets one GitHub secret from stdin (a name outside
  the registry). Read other credentials with the `secrets-access` skill. Worker secrets live in
  Cloudflare (`wrangler secret put NAME`, prompts on stdin).
- Household roles: `hh ops roles` lists, `hh ops roles set <email> <role>` sets as the signed-in
  admin (`hh login`), through the rules: never one's own role, only members.
- Firebase web config, OAuth client ids, VAPID public keys and New Relic browser keys are public
  by design: repo variables, not secrets.
- Never poll a command that can raise a credential or permission prompt; ask once, then wait.
