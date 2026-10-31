# Grit, redefined from scratch

> **Status (2026-10-31):** design vision. Shipped commands are only `init` `status` `diff`
> `snapshot` `log` `show` `restore` `undo` `checkout` `branch` `tag` `verify` `gc` `doctor`.
> Every other `grit …` command in this file is **planned**, not in the binary yet.

Grit is not Git with nicer commands.
It is a local, automatic, content-deduplicated project backup system for humans and AI agents.

No GitHub assumptions. No branches. No staging. No rebases. No detached HEAD. No force push. No duplicate worktree forests.

---

## Core model

```
Project folder
    ↓
automatic journal
    ↓
deduplicated local object store
    ↓
recoverable project states
```

Grit watches a project and records meaningful changes continuously.

The user mainly does three things:

```
grit start
grit undo
grit restore
```

Everything else is secondary.

---

## Replace Git concepts entirely

| Git | grit | Status |
|---|---|---|
| `git init` | `grit init` | shipped |
| `git add` + `git commit` | `grit snapshot -m "…"` | shipped |
| `git status` | `grit status` | shipped |
| `git diff` | `grit diff` | shipped |
| `git log` | `grit log` | shipped |
| `git show` | `grit show` | shipped |
| `git reset --hard` / `git checkout -- .` | `grit restore` | shipped |
| `git reflog` + reset | `grit undo` | shipped |
| `git branch` / `git tag` / `git checkout` | same names (transitional) | shipped |
| `git fsck` | `grit verify` | shipped |
| `git gc` | `grit gc` | shipped |
| staging area / index | none — snapshot takes whole tree | by design |
| `git merge` | `grit combine` | planned |
| `git cherry-pick` | none yet | planned |
| `git worktree` | `grit workspace` | planned |
| `git push` / `git pull` | `grit backup` / `grit sync` | planned |

All grit commands are lowercase verbs. "Shipped" = in the `grit` binary today.

Use normal words. Do not make users learn version-control theology.

---

## The idiot-proof rules

### 1. Every change is recoverable

Grit journals changes before and after writes.

```
AI rewrites 40 files
build fails
user runs:
grit undo
```

No commit needed. No staged state. No reflog archaeology.

---

### 2. Saving is automatic

Grit creates snapshots based on events:

* before an AI task;
* after an AI task;
* before mass edits;
* after tests pass;
* before file deletion;
* before rename/move;
* every configurable time interval;
* when the editor becomes idle;
* when the user explicitly runs `grit save`.

Example timeline:

```
10:12  Before Claude task
10:14  Added computer backend modules
10:18  Build passed
10:21  Before dependency migration
10:24  Current
```

---

### 3. No staging area

There is no equivalent of:

```
git add
git reset
git commit
```

Instead:

```
grit save "computer backend compiles"
```

Grit records the actual project state.

---

### 4. No destructive history rewriting

Snapshots are immutable.

You can:

* restore an older state;
* start a new path from it;
* hide snapshots;
* compact unreachable temporary data later.

You cannot silently destroy history with rebase or force push.

---

## Fixing the doppelganger filesize monster

The filesystem may contain ten apparent project copies, but Grit stores duplicate content once.

### Storage architecture

```
~/.grit/
  repos/
  objects/
  manifests/
  journals/
  cache/
```

Files are split into content-defined chunks:

```
large-model.bin
├── chunk A
├── chunk B
├── chunk C
└── chunk D
```

Another copy containing A, B, C and a changed D stores only the new D.

Use:

* BLAKE3 for identity;
* content-defined chunking for large files;
* zstd compression;
* immutable object storage;
* reference-counted garbage collection.

---

## Example

Five 20 GB project workspaces mostly contain identical files.

| | Size |
|---|---|
| Normal copies | 5 × 20 GB = 100 GB |
| Grit storage — shared content | 20 GB |
| Grit storage — unique modifications | 2 GB |
| Grit storage — metadata | small |
| **Grit total** | **~22 GB** |

Not five physical repository clones.

---

## Workspaces without worktree duplication

A Grit Workspace is an overlay, not a copied project tree.

```
Base snapshot
    +
Workspace overlay
```

Example:

* main workspace
* agent-ui workspace
* agent-backend workspace
* experiment workspace

Each workspace stores only changed paths or changed grains.

```
grit workspace new agent-backend
grit workspace enter agent-backend
grit workspace discard agent-backend
grit workspace combine agent-backend
```

Internally:

```
workspace/
  base_snapshot = abc123
  changed_objects = [...]
  deleted_paths = [...]
  renamed_paths = [...]
```

No duplicated `target/`, `node_modules/`, model files, references, caches, or unchanged source.

---

## Separate source from disposable bulk

Grit classifies files into storage classes.

```
[storage.source]
patterns = ["src/**", "Cargo.toml", "docs/**"]

[storage.generated]
patterns = ["target/**", "dist/**", ".cache/**"]
policy = "rebuildable"

[storage.large]
patterns = ["models/**", "Reference/**"]
policy = "chunked"

[storage.secret]
patterns = [".env", "keys/**"]
policy = "local-encrypted"
```

### Storage policies

* **tracked** — fully snapshotted
* **chunked** — deduplicated large-file storage
* **rebuildable** — metadata only, normally not backed up
* **ignored** — invisible to Grit
* **encrypted** — locally encrypted before storage
* **pinned** — never garbage-collected

This prevents build artifacts from multiplying across workspaces.

---

## AI-native safety

Every agent task gets a transaction.

```
Task begins
    ↓
automatic snapshot
    ↓
writes occur
    ↓
format/build/test
    ↓
success snapshot   or   automatic rollback
```

Task metadata:

```
task = "Port Hermes computer-use package to Rust"
agent = "Fable"
started_at = "2026-07-07T02:01:00-04:00"
base_snapshot = "b3:..."
result = "build_failed"
```

Commands:

```
grit task begin "Port computer-use"
grit task finish
grit task abort
```

Normally the coding tool calls these automatically.

---

## Semantic change tracking

Text files can optionally be decomposed into grains:

```
Rust file
├── imports
├── struct ComputerBackend
├── impl ComputerBackend
├── fn capture
└── tests
```

This gives:

* function-level history;
* function-level restore;
* conflicts only when the same grain changed;
* reformatting noise reduction;
* better AI summaries.

But semantic parsing is an enhancement, not a requirement for safety.
Raw file and chunk storage remain authoritative.

---

## Conflict model

Grit should avoid the word "merge conflict" until necessary.

It first attempts:

1. identical-content deduplication;
2. independent-path combination;
3. independent-grain combination;
4. structural merge;
5. human choice only for actual overlap.

Example:

```
Both workspaces changed src/backend.rs.
agent-a changed:  fn capture
agent-b changed:  fn list_apps
Result: combined automatically
```

Only two changes to the same function require intervention.

---

## Local backup model

A grit repo can have replicas, but the primary model is local.

```
Internal SSD
    ↓
External backup drive
    ↓
optional second disk
```

Commands:

```
grit backup add /Volumes/<backup-drive>/GritBackup
grit backup now
grit backup verify
grit backup status
```

No account. No cloud. No server.

Backup behavior:

* incremental;
* deduplicated;
* checksummed;
* resumable;
* encrypted optionally;
* periodically verified;
* resistant to partial-copy corruption.

---

## Safe deletion and garbage collection

Deleting a file does not immediately erase stored content.

```
grit clean
```

Shows:

```
2.4 GB  temporary workspace objects
8.1 GB  rebuildable cache
14 MB   unreachable source history
```

Then:

```
grit clean --safe
```

Rules:

* recent snapshots remain protected;
* pinned snapshots remain forever;
* backup-only objects are never deleted accidentally;
* deletion uses a quarantine period;
* every cleanup produces a recovery manifest.

---

## Minimal CLI

### Everyday commands (planned set — today `grit snapshot` = save, `grit log` = timeline)

```
grit start
grit status
grit save
grit undo
grit timeline
grit restore
grit workspace
grit backup
```

### Examples

```
grit start
grit save "before Rust migration"
grit undo
grit restore yesterday
grit restore "last build passed"
grit workspace new experiment
grit backup now
```

Natural selectors should work:

```
grit restore "before Fable changed computer backend"
grit undo last-task
grit show "what changed today"
```

---

## Status should answer human questions

Bad:

```
HEAD detached at 7f91bc2
Changes not staged for commit
```

Good:

```
Project protected
Last snapshot: 3 minutes ago
Current changes: 14 files
Recoverable: yes
Backup disk: current
Workspace: main
Build state: failed
```

---

## Grit's actual promise

You work normally. Grit continuously preserves recoverable project states,
stores duplicate content once, isolates experiments without copying the project,
and makes accidental destruction difficult.

**That is the product.**

Not distributed source control.
Not Git compatibility.
Not GitHub workflow management.
Not branch theory.

A local project time machine with deduplicated workspaces and AI-safe transactions.
