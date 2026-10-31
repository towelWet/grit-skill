# grit-skill

**Claude Code skill for [Grit](https://github.com/towelWet/grit) — local-first, content-addressed version control.**

This is the AI skill / design spec that drives Claude Code when working on Grit.
It holds settled design decisions, command surface, storage model, AI verb layer,
and study notes so context never has to be re-derived session to session.

## What this skill contains

| File | Purpose |
|---|---|
| `SKILL.md` | canonical design spec — identity, storage, safety model, sync, study set |
| `ai-commands.md` | AI verb layer — `save` auto-message, `ask`, `why`, `combine` |
| `Psychology/` | design psychology docs — why Grit exists, storage-no-deletion philosophy |

## Relationship to Grit project

This skill is the **design document and AI integration layer**.
The implementation (Rust engine + CLI) lives in the project repo:

**Project repo:** [towelWet/grit](https://github.com/towelWet/grit)

| Concern | Where |
|---|---|
| Design spec, settled decisions | this repo (`grit-skill`) |
| Rust source, build, tests | [towelWet/grit](https://github.com/towelWet/grit) |
| Roadmap / future features | `FUTURE.md` in project repo |

## Install

```
git clone https://github.com/towelWet/grit-skill ~/.claude/skills/grit
```

Then invoke `/grit` in Claude Code. Build the `grit` binary from
[towelWet/grit](https://github.com/towelWet/grit) to run the commands it describes.

## License

MIT.
