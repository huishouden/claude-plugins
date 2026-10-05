---
name: hh
description: >
  Using the `hh` command line (huishouden/cli): install, signing in (`hh login`, loopback + PKCE
  through the portal), the household's data as the signed-in person (`hh data`: today, groceries,
  tasks, pets, Health…, the AI connector's own tools), `ops` (auth domains, OAuth origins and
  redirect URIs, secrets from stdin, monitoring, household roles, profile-check, staging-cleanup),
  the `dev` group (verify, evidence, ready, review, bump-kit), --json output, and how to
  add a command. Use whenever a Huishouden task runs hh, reads or changes household data from a
  terminal, when hh is missing or fails, or when adding an hh command.
---

# hh, the Huishouden command line

Repo: https://github.com/huishouden/cli. Bun and TypeScript, PolyForm Shield.

## Install

```sh
bun add -g https://github.com/huishouden/cli/releases/download/vX.Y.Z/cli-X.Y.Z.tgz   # a release's tarball (public)
hh --version
```

hh updates itself: before a command runs it installs the latest release (asked of GitHub at most
every 6 hours, cached in `~/.cache/hh`) and runs the same command on it. Skipped in CI, with
`HH_NO_AUTO_UPDATE=1`, offline, or from a source checkout; a failure warns and the command runs.
`hh self-update` does it by hand. To install by hand over an existing install, `bun remove -g
@huishouden/cli` first (bun add -g over a tarball URL fails with DependencyLoop).

`hh login` and `hh data` need only `bun` (and a browser once). The `dev` and `ops` commands also
need `gh` (signed in), and per command: `cr` (review), Java 21 (emulator tests), `firebase` signed
in with access to `huishouden-staging` and `gcloud` (staging evidence, `ops auth-domains`).

Every command takes `--json`: exactly one JSON document on stdout (`ok`, `command`, data), progress
on stderr. Exit code 0 means `ok: true`. Prefer `--json` when an agent reads the result.

## Signing in

`hh login [--staging]` is the person's own step: it opens their browser, and only they can sign in
and press Allow. An agent never runs it for them, never reads the stored sign-in, and never
completes the portal page.

1. hh listens on `127.0.0.1:<random port>` and opens the portal's
   `/connect?service=hh&redirect=http://127.0.0.1:<port>/callback&state=…&code_challenge=…`.
2. The portal accepts only that exact loopback address, says "Sign in to the hh command-line tool
   on this computer", and on Allow hands the sign-in to the connector Worker (`/cli/hand-off`),
   which keeps it two minutes under a one-time code bound to the state and the PKCE challenge.
3. The browser returns to hh with only the code; hh trades it with its verifier at `/cli/token`.
4. Kept in the macOS Keychain or libsecret; otherwise an encrypted file in `~/.config/hh` with a
   warning (`HH_PASSPHRASE` for a passphrase key).

`hh whoami [--staging]` checks the sign-in (exit 1 and "hh login" when there is none or it ended).
`hh logout` removes it from this computer; Firebase can't revoke one sign-in from a client.

## Household data: `hh data`

Acts as the signed-in person, so the household's rules decide everything. The commands are the AI
connector's tools (`@huishouden/pwa-kit/household-tools`), the same code: answers match the
assistant's. Tables for lists, a sentence for changes; `--json` returns `{ ok, tool, household,
lang, text, data }` (the tool's own data). Exit 1 when the rules or the tool refused (the text says
why). Writes are not marked `via: 'assistant'`.

| Command | Tool |
|---|---|
| `households`, `household home` | who and which households, the home's address |
| `today`, `calendar [from] [to]`, `todos [--filter --app --sort --limit]` | agenda and to-dos |
| `todo done <id>`, `todo cancel <id>` | a to-do's own Done/Cancel (ids from `todos`) |
| `groceries list [list]`, `groceries add <name> [--quantity --category --list --urgency]`, `groceries check <item>` | shopping lists |
| `tasks add <name> [--due YYYY-MM-DD[THH:MM]]` | tasks |
| `bills due` | bills (admins and members) |
| `pet today [pet]`, `pet feeding <pet>`, `pet dose <pet> <medicine>` | Pet |
| `home upkeep`, `home event <title>` | Home |
| `appointment add <app> <title> <start>` | appointments |
| `contacts search <query>`, `contacts add <name>` | contacts |
| `health people`, `health medicines <person>`, `health history <person>`, `health due [person]`, `health appointments [person]`, `health dose <person> <medicine>`, `health add <person> <name>`, `health update <person> <medicine>`, `health doctor-list <person>` | Health (carers and admins) |

- Every other argument is a `--kebab-case` flag named after the tool's: `hh data <command> --help`
  lists them with their accepted values. Lists are comma-separated (`--times 08:00,20:00`),
  `--no-x` sets false, `null` clears a nullable field.
- `--household <id or name>` when the person has several (`hh data households`).
- Writes take `--idempotency-key <k>`: reuse it when retrying the same change.
- Health and Pet doses: a guard warning (`needs_confirmation: true`, nothing written) means ask the
  person; only then repeat with `--confirm`.
- Answers are the household's records, not medical advice.

## Commands

| Command | Does |
|---|---|
| `hh dev verify` | Install, lint, kit checks (design, writes, headers, i18n, bandwidth), unit tests, build, screenshots (phone/tablet, light/dark) on a local preview, emulator tests. Writes `.hh/evidence/<sha>/` |
| `hh dev evidence [--staging\|--local] [--no-post]` | Verify locally or on the app's staging site (chosen from changed paths), post/update the PR's evidence comment with images |
| `hh dev review [--pr=N]` | `cr review` with the org reviewers, one at a time per machine; findings and unresolved threads; records the result for the head commit (PR comment + `~/.cache/hh/reviews/`) |
| `hh dev ready [--dry-run] [--no-bump-kit]` | Review bar (reviewer review or `hh-review` marker for head) and evidence for head; then `gh pr ready`. Bumps a behind kit first (and redoes review and evidence). Mandatory before ready/merge |
| `hh dev bump-kit [--to=vX.Y.Z]` | `@huishouden/pwa-kit` (the release tarball, else the git tag) and the workflow refs to the latest tag, install, lint, test. verify, evidence, review and ready do this as a `chore: kit vX.Y.Z` commit when behind (`--no-bump-kit` skips) |
| `hh ops auth-domains [--production\|--staging]` | Firebase Auth authorized domains vs the expected set (gcloud token, read-only); the kit bootstrap fixes |
| `hh ops oauth-check [--production\|--staging]` | OAuth client JavaScript origins + the auth handler redirect URI (public probes); missing ones are a console click for the user |
| `hh ops secret set <repo> <name> [--env <e>]` | GitHub Actions secret, value on stdin only (see the ops skill) |
| `hh ops monitoring [--dry-run]` | Dispatch the portal's monitoring workflow (New Relic), wait, summarize |
| `hh ops roles`, `hh ops roles set <email> <role>` | Household roles; set as the signed-in admin through the rules |
| `hh ops staging-cleanup` | Staging leftovers over a day old (kit's sweep from its latest release, gcloud-impersonated token); evidence --staging runs it |
| `hh ops profile-check [--fix]` | Org profile and repo descriptions vs apps.json and Worker repos; `--fix` opens a PR |
| `hh login [--staging]`, `hh whoami`, `hh logout` | The sign-in for `hh data` and `hh ops roles` (above) |

`.hh/` in a repo is hh's scratch space (results, screenshots, wrapper configs); hh adds it to
`.git/info/exclude`.

## Troubleshooting

- "ports 8080/9099 are in use": another emulator run on the machine; hh waits up to 15 minutes.
- "another review is running": `cr` runs one review per machine; hh waits.
- "No PR for this branch": push and `gh pr create --draft` first, or `--no-post`.
- "not signed in" / "the sign-in has ended": the person runs `hh login` (add `--staging` for
  staging). An agent asks them to; it doesn't retry in a loop.
- `hh login` printed a URL and nothing opened: the person opens it in their browser; it waits five
  minutes.
- Staging deploy fails on permissions: `firebase login` with an account that can deploy
  `huishouden-staging`; signed-in tests need `gcloud auth login` and Token Creator on the staging
  deploy account (kit STANDARD.md "Staging").

## Adding a command

One file in `src/commands/<group>/` calling `register({ group, name, summary, usage, run })`,
imported from the group's `index.ts`; `run` returns `{ ok, data, text }`. Two-word names
(`roles set`) work; `valued` lists the flags that take a value. Groups: `dev`, `ops`, `data`,
`account`. A new household tool goes in the kit's `household-tools` (so the connector gets it too)
and is one row in `src/commands/data/index.ts`; anything else acting as the person uses
`openSession(site)` from `src/lib/household.ts`. Follow the PR lifecycle; CI tags the version and
attaches the tarball on merge (no version in the PR).
