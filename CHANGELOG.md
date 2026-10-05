# Changelog

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
