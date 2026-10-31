# grit AI commands — verbs that press into an LLM

Captured 2026-07-07 from the git↔grit teaching session (god readiness-hang fix session).
Delegate of `SKILL.md` — read only when designing/building the AI verb layer.

## The idea

Most grit verbs are deterministic Rust (`save`, `undo`, `restore`). Some verbs,
when invoked, **assemble context and press into an LLM** instead of computing an
answer in code. git has zero of this — its porcelain is dumb deterministic
scripts. grit is AI-native by design.

## The mechanism (three steps)

**1. Bundle** — grit gathers a context bundle: the relevant grains, a journal
slice, a diff, as structured text.

**2. Press** — it sends that bundle to an LLM endpoint. Reuse god's local
mlx-magic server (`:8765`) so it stays offline, no account — matching grit's
local-first ethos.

**3. Render** — it gets back natural language or a proposal, and renders/uses it.

## Concrete AI verbs

| Verb | AI behavior |
|---|---|
| `grit save` (no message) | reads the grain diff, writes the save note itself |
| `grit why <grain>` | synthesizes that function's history into plain English |
| `grit ask "…"` | natural-language query over the grind |
| `grit combine` | on a real overlap, LLM **proposes** a merge of two grains |

## The one hard safety rule

**AI proposes, the deterministic core disposes.** An AI verb never writes the
store directly. It returns a *proposal*; the append-only deterministic engine
applies it. `grit save`'s AI-written note is just text on a save that happened
deterministically — the LLM can hallucinate a *message* but can never corrupt
*history*. Preserves the idiot-proof promise while adding intelligence.

## Why git can't do this

git's atom is an opaque file blob — nothing semantic to hand an LLM. grit's
grains are already function-level semantic units, so they're *natively*
LLM-ready context. The grain model isn't just for merges — it's what makes AI
commands possible at all.

## Architecture note

AI verbs are hot-loadable — drop `verbs/ai/<name>.rs` — same as grit's
split-files ethos. No central registry edit to add one.
