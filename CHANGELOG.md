# Changelog

## 2.4.0 (2026-10-11)

### Documentation

* developing-an-app, pr-lifecycle, hh: household tools ship with the data. A change that adds or changes user-editable data, or a household task an agent was asked to do without a tool, carries the kit tool, the connector bump and the `hh data` command in the same change set. The kit's `COVERAGE` lists every writable collection, and huishouden/rules' test enforces it.
* hh: the `baby`, `pet outing`, `pet routines` and `health conditions` commands are in the `hh data` table.

## 2.3.2 (2026-10-11)

### Documentation

* developing-an-app: never create a `.env.local` with production values; `bun run env:pull` writes staging values, and `hh dev verify` and `hh dev evidence` refuse a production Firebase env.

## 2.3.1 (2026-10-11)

### Documentation

* developing-an-app, pr-lifecycle, hh: kit workflows and actions are pinned by commit SHA with the version in a trailing comment (`@<sha> # vX.Y.Z`); `hh dev bump-kit` maintains it.

## 2.3.0 (2026-10-10)

### Documentation

* pr-lifecycle, hh: versions come from CI on merge (tag, release, tarball); no version or CHANGELOG in a PR, `hh dev release` is a no-op; hh updates itself before a command; verify/evidence/review/ready bump a behind kit as a `chore: kit` commit. Install by release tarball.

## 2.2.0 (2026-10-10)

### Features

* ops, hh: the secret rule (never paste a secret in chat: copy it, `hh ops secret store`, then `push`; agents never read keychain values) and `hh ops secret store`, `push`, `where`, `rotate`.

## 2.1.3 (2026-10-10)

### Documentation

- `hh ops deploy-site` and the pr-lifecycle step that runs it after an app merge (apps hold no Cloudflare token).

## 2.1.2 (2026-10-05)

### Documentation

* pr-lifecycle, hh: `hh dev review` records a clean result for the head commit (hh-review marker, hh 1.3.2); `hh dev ready` accepts it from the PR author or the reviewer.

## 2.1.1 (2026-10-05)

### Documentation

* hh: `hh data health appointments [person]` (Health visits, kit 0.98.0).

## 2.1.0 (2026-10-05)

### Features

* hh: `hh login` (loopback + PKCE through the portal; the person's own step), `hh data` (the AI connector's tools as the signed-in person: commands, flags, `--json`, confirming guards), and the new `hh ops` commands (auth-domains, oauth-check, secret set, monitoring, roles).
* ops: `hh ops oauth-check` and `auth-domains` for sign-in lists, `hh ops secret set` for GitHub secrets from stdin, `hh ops monitoring`, `hh ops roles`; the connector's `/cli/*` endpoints.

## 2.0.2 (2026-10-05)

### Documentation

* hh: updating needs `bun remove -g` first (bun caches `#v1`); `hh ops staging-cleanup`.

## 2.0.1 (2026-10-05)

### Documentation

* hh: install with `bun add -g github:huishouden/cli#v1` (bunx caches moving tags); pr-lifecycle: body-only findings and a review per head.

## 2.0.0 (2026-10-05)

### Features

* The `huishouden` plugin moves here from piekstra/claude-plugins (1.9.0) and splits into
  `developing-an-app`, `pr-lifecycle`, `hh` and `ops`; the developer-owned PR flow with the `hh`
  CLI, the Hosting bandwidth rule and the new-app checklist (org profile, repo description).
