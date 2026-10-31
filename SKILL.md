---
name: grit
category: coding
tags:
  - rust
  - cli
  - vcs
  - version-control
  - local-first
  - content-addressed
  - split-files
trigger: /grit
description: >
  grit — a local-first, content-addressed version control system of the user's own
  design. A git alternative built around sub-file (grain) tracking, BLAKE3 atoms,
  speed, and idiot-proof safety. Use when the user says /grit, or wants to design,
  spec, or build grit: storage format, splitmap grammar, sync, or command surface.
---

# /grit

`grit` is the user's own version control system — a from-scratch git alternative,
designed around his priorities: **assembly / low-level, split-files, speed, idiot-proof,
local-first**. This skill holds the canonical design so it never has to be re-derived.

Status: **implementation in progress**. The Rust engine (`grit-engine`) and CLI (`grit-cli`)
live in the grit project repo (local path: `local.toml` beside this file — private, never published). Current
storage: raw blobs (`GRB1` marker, BLAKE3 domain-tagged ids), a stat cache, streamed and
parallel hashing, empty snapshots create no commit, and nested per-directory trees
(commit format 2). Legacy JSON-blob and flat-tree repos still read. The `--format toml` key
contract is in `docs/output-schema.toml` of that repo. This skill remains the design spec;
see the repo's `FUTURE.md` for the roadmap.

## The identity (settled decisions — do not re-litigate)

| Decision | Value |
|---|---|
| Name | `grit` (CLI + repo format) |
| Unit of tracking | a **grain** = one function / struct / config block |
| The store | the **grit** (content-addressed grain pile) |
| The history | the **grind** |
| Hash | BLAKE3 (SIMD, parallel) |
| Compression | zstd |
| Access | mmap, zero-copy |
| Core language | Rust, with asm/SIMD hot paths |
| Distribution | single static binary, zero deps |
| Network model | **local-first, peer-to-peer — no GitHub, no cloud, no account** |

## The one idea that makes it grit, not git

Git's atom is the **file**. grit's atom is the **grain** (function / section).

A repo carries a **splitmap** — a rule that decomposes each file into grains.
grit versions the grains, not the whole blob. Payoffs:

**Merges stop conflicting** — two people editing different functions in the same
file touch different grains → zero conflict.

**Diffs are semantic** — "you changed `parse_header`", not "lines 340–520 changed".

**It bakes /split-files into the tool** — the repo rewards small units.

## Storage — content-addressed grain store

Flat CAS keyed by BLAKE3. Grains stored zstd-compressed, mmap'd. No zlib, no
git-style packfile repack. The index is a mmap'd binary table, O(1) lookup.

The hot loop (hashing / diffing grains) is exactly the kind of thing to hand-tune
with SIMD or asm — that's a feature, not an afterthought.

## Idiot-proof — the safety model (this is where it diverges hardest)

**Append-only, always** — nothing on disk is ever overwritten or deleted.

**`grit undo` is first-class** — any operation reverses with one word. Not a
buried reflog trick; the primary interface.

**No footguns** — no detached HEAD, no `--force` that eats history, no silent
staging surprises.

**Delete = trash generation** — never gone (mirrors the user's mv-to-Trash rule).

**Destructive-looking ops preview by default** — dry-run is default, `--do` commits.

**Errors tell you the fix** — plain English, with the exact next command.

## Command surface — small, obvious verbs

| Verb | Does |
|---|---|
| `grit save` | snapshot current state |
| `grit undo` | reverse the last op |
| `grit log` | history, grain-level |
| `grit branch` | fork a line |
| `grit sync` | pull/push between peers |
| `grit serve` | expose this repo over ssh/LAN (the whole "server") |
| `grit bundle` | pack repo into one carryable file |
| `grit why <grain>` | who/when/what changed a function |

**AI verbs** (`grit save` auto-message, `grit ask`, `grit combine` proposals):
see [ai-commands.md](ai-commands.md) — LLM proposes, deterministic core disposes.

## Sync — peer-to-peer, no central node

Your machine is the source of truth. Sync is an optional convenience between peers
you control. Because every grain is content-addressed, two repos just trade the
hashes they're missing — set difference over BLAKE3 keys, no merge server.

| Transport | How |
|---|---|
| LAN | mDNS discovery — `grit sync` finds peers on the network |
| SSH | `grit sync user@box:path` — plain pipe over ssh |
| USB / file | `grit bundle` → one file you carry anywhere |
| Offline | fully functional; sync is never required |

A "remote" is just another grit repo on a box you own — a bare directory, not a
service. No line of code assumes GitHub exists.

## Study set — ranked git alternatives (design inspiration)

We are studying the field to steal the best ideas for grit. Ranked best→worst
fit for grit's goals. Cloned codebases live in:
`Reference/` inside the grit project repo (excluded from publish)

| # | Tool | Lang | Best for | Cloneable |
|---|---|---|---|---|
| 1 | Jujutsu (jj) | Rust | best dev UX, git-compatible backend | yes |
| 2 | Fossil | C | independent, local-first, self-hosted | yes |
| 3 | Pijul | Rust | patch-based, closest to grain/atom idea | yes |
| 4 | Mercurial (hg) | Python/C | stable, simpler traditional DVCS | yes |
| 5 | Sapling (sl) | Rust/Python | huge repos, stacked changes | yes |
| 6 | Darcs | Haskell | fine-grained interactive patch selection | yes |
| 7 | Unity/Plastic SCM | — | games, large binaries, GUI | no (closed) |
| 8 | Perforce P4 | — | AAA games, huge binary depots | no (closed) |
| 9 | Subversion | C | simple centralized, file locking | yes |

**Highest-value targets for grit's model:** Pijul (patch/atom theory → grain
merges), Fossil (single-binary, self-contained, local-first), Jujutsu (undo-first
UX, safety). Perforce & Plastic are closed — study only via docs, not source.

**Analysis done** — full write-up: `Reference/_ANALYSIS.md`. Key results:
- pijul's `libpijul/src/pristine/` is a vertex+edge **graph** mutated by patches =
  grit's grain model, already shipping. Study first.
- `sanakirja` (~3.5k LOC) is a COW transactional mmap B-tree = candidate grit store.
- jujutsu's `op_store`/`operation` = the undo-log design for `grit undo`.
- fossil = single static binary + delta blobs in SQLite (deployment lesson).
- All 4 self-hosted tools (fossil/pijul/darcs/hg) can't be git-cloned — pulled as
  tarballs/crates. Lesson: a real VCS dogfoods itself, doesn't need GitHub.

## Reference — what git is (for contrast)

Git core is **C**; porcelain historically shell + Perl; `gitk`/`git-gui` in Tcl/Tk.
grit rejects the file-atom model, the central-remote default, and the footgun surface.

## Next steps (when the user says go)

1. Nail the on-disk format (grain header, CAS layout, index table).
2. Define the splitmap grammar (how files decompose into grains, per language).
3. Spec command semantics (save/undo/sync state transitions).
4. Only then scaffold the Rust crate — CLI-first per /cli-first.
