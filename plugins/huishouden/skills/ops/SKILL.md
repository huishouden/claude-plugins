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
  app's test runs (`pwa-staging quota`).
- Hashed `/assets/*` are `immutable` for a year; HTML, `sw.js` and manifests `no-cache`; Hosting
  serves Brotli/gzip. `pwa.yml`'s smoke checks these on every deploy.

Deploys: only `main`'s `pwa.yml` deploys production (keyless, Workload Identity Federation).
`bun run bootstrap` in the portal grants deploy access and provisions repos; the user runs it.
Firestore rules deploy only from `huishouden/rules` main.

## Sign-in origins

Each project's OAuth client lists only its suite site and `<project>.firebaseapp.com`; Google allows
an unverified app 10 domains. `pwa-oauth-origins` checks them (the smoke job runs it). Adding an
origin is a Cloud Console click for the user.

## Workers (Cloudflare, free plan)

`notify` (Web Push every 5 minutes: product, keep its schedule), `connector` (MCP server for AI
assistants, acts as the person via the portal's /connect hand-off), `calendar` (feed and Google
sync, card alerts). Each deploys staging then production from `main` with `CLOUDFLARE_API_TOKEN`.
Worker secrets live in Cloudflare (`wrangler secret put`), never in the repo or Actions.

## New Relic

Free tier: Browser apps, a ping monitor per app path (every 30 minutes, two locations), alerts and a
dashboard, provisioned by the kit's `infra/newrelic.ts` from the portal's `monitoring` workflow
(runs when `apps.json` changes, or by hand). Pings are HTTP only, no browser.

## GitHub

- Org `huishouden`; every repo public, PolyForm Shield (`actions/license-check`).
- Hosted CI on `main` only; no schedules, no Renovate, no release-please (pr-lifecycle skill).
- Org profile `huishouden/.github` `profile/README.md`: a row per app and Worker, and every repo
  has a description. `hh ops profile-check` (`--fix` opens a PR) when apps or Workers change.
- The Renovate GitHub App, if still installed, is the user's to uninstall (org settings,
  GitHub Apps).

## Secrets

- Read with the `secrets-access` skill; never print, log, or pass a secret in argv (`ps` shows
  argv). Pipe on stdin: `gh secret set NAME -R huishouden/<repo> < file` or `printf %s "$v" | gh
  secret set NAME`, `wrangler secret put NAME` (prompts on stdin).
- Firebase web config, OAuth client ids, VAPID public keys and New Relic browser keys are public
  by design: repo variables, not secrets.
- Never poll a command that can raise a credential or permission prompt; ask once, then wait.
