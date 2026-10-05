---
name: hh
description: >
  Using the `hh` command line (huishouden/cli): install, the `dev` group (verify, evidence,
  release, ready, review, bump-kit), `ops` (profile-check), the account commands (login, whoami,
  logout), --json output, and how to add a command. Use whenever a Huishouden task runs hh, when hh
  is missing or fails, or when adding an hh command.
---

# hh, the Huishouden command line

Repo: https://github.com/huishouden/cli. Bun and TypeScript, PolyForm Shield.

## Install

```sh
bunx github:huishouden/cli#v1 <group> <command>     # no install; v1 follows the latest 1.x
bun add -g github:huishouden/cli#v1                 # or install `hh` on PATH
```

Needs `bun`, `gh` (signed in), and per command: `cr` (review), Java 21 (emulator tests),
`firebase` signed in with access to `huishouden-staging` and `gcloud` (staging evidence), the OS
keychain (login).

Every command takes `--json`: exactly one JSON document on stdout (`ok`, `command`, data), progress
on stderr. Exit code 0 means `ok: true`. Prefer `--json` when an agent reads the result.

## Commands

| Command | Does |
|---|---|
| `hh dev verify` | Install, lint, kit checks (design, writes, headers, i18n, bandwidth), unit tests, build, screenshots (phone/tablet, light/dark) on a local preview, emulator tests. Writes `.hh/evidence/<sha>/` |
| `hh dev evidence [--staging\|--local] [--no-post]` | Verify locally or on the app's staging site (chosen from changed paths), post/update the PR's evidence comment with images |
| `hh dev release [--level=…] [--commit] [--dry-run]` | package.json version + CHANGELOG.md section from Conventional Commits since the last tag |
| `hh dev review [--pr=N]` | `cr review` with the org reviewers, one at a time per machine; findings and unresolved threads |
| `hh dev ready [--dry-run]` | Review bar, evidence for head, version bump; then `gh pr ready`. Mandatory before ready/merge |
| `hh dev bump-kit [--to=vX.Y.Z]` | `@huishouden/pwa-kit` to the latest tag, install, lint, test |
| `hh ops profile-check [--fix]` | Org profile and repo descriptions vs apps.json and Worker repos; `--fix` opens a PR |
| `hh login [--staging]`, `hh whoami`, `hh logout` | One sign-in via the portal's /connect hand-off, kept in the OS keychain, for `hh data` |

`hh data …` (household data as the signed-in person) and more `hh ops …` commands are planned;
`hh --help` lists what exists.

`.hh/` in a repo is hh's scratch space (results, screenshots, wrapper configs); hh adds it to
`.git/info/exclude`.

## Troubleshooting

- "ports 8080/9099 are in use": another emulator run on the machine; hh waits up to 15 minutes.
- "another review is running": `cr` runs one review per machine; hh waits.
- "No PR for this branch": push and `gh pr create --draft` first, or `--no-post`.
- Staging deploy fails on permissions: `firebase login` with an account that can deploy
  `huishouden-staging`; signed-in tests need `gcloud auth login` and Token Creator on the staging
  deploy account (kit STANDARD.md "Staging").

## Adding a command

One file in `src/commands/<group>/` calling `register({ group, name, summary, usage, run })`,
imported from the group's `index.ts`; `run` returns `{ ok, data, text }`. Groups: `dev`, `ops`,
`data`, `account`. Household-data commands act as the signed-in person (`idToken()` in
`src/commands/account`) with the kit's server-safe cores (todo-core, agenda-core, contact-core,
role-core, firestore-rest, firebase-auth-rest), the same code the connector's tools use. Follow
the PR lifecycle; `main` tags the version and moves `v1`.
