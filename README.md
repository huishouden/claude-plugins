# Huishouden plugins for Claude Code

A plugin marketplace with one plugin, `huishouden`: the skills an agent needs to work on the
[Huishouden](https://github.com/huishouden) apps.

| Skill | For |
|---|---|
| `developing-an-app` | Standards, design, the kit, Firebase, Hosting bandwidth, staging, the new-app checklist |
| `pr-lifecycle` | Draft → `hh dev review` → `hh dev verify`/`evidence` → `hh dev ready` → merge (CI versions on merge) |
| `hh` | Installing and using the [`hh` CLI](https://github.com/huishouden/cli) |
| `ops` | Firebase projects and limits, sign-in origins, Workers, New Relic, GitHub, secrets |

## Install

```sh
claude plugin marketplace add huishouden/claude-plugins
claude plugin install huishouden@huishouden
```

Update: `claude plugin marketplace update huishouden && claude plugin update huishouden@huishouden`.

## Changing a skill

Edit the SKILL.md. This repo is the one exception to the suite's CI versioning (the marketplace has no release step): bump `version` in `plugins/huishouden/.claude-plugin/plugin.json` (installed
copies only refresh on a version change) and add the CHANGELOG.md entry, in the same PR. Otherwise follow
the pr-lifecycle skill.

## License

[PolyForm Shield 1.0.0](LICENSE).
