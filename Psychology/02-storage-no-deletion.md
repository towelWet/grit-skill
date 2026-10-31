# Corrections to file 01 — Storage promise, no deletion

> **Status (2026-10-31):** design vision. Shipped commands are only `init` `status` `diff`
> `snapshot` `log` `show` `restore` `undo` `checkout` `branch` `tag` `verify` `gc` `doctor`.
> Every other `grit …` command in this file (`backup`, `clean`) is **planned** or rejected.

File 01 was written with sections on garbage collection, `grit clean`,
and unreachable-object cleanup that contradict Grit's core promise.

## The promise (corrected)

Grit keeps every recoverable state, deduplicates identical content
automatically, and never makes you decide what history is safe to delete.

## How storage stays controlled — without deletion

| Mechanism | What it does |
|---|---|
| Deduplication | Identical content stored once regardless of how many workspaces or snapshots reference it |
| Compression | zstd on every object — stored size is always less than logical size |
| Rebuildable-file exclusion | `target/`, `dist/`, `.cache/` are tracked by metadata only — never stored as snapshot content |
| Shared workspace data | Workspaces are overlays on a base snapshot — changed objects only, no copy of unchanged bulk |
| Optional archival | `grit backup` moves cold data to external disk — the local repo stays lean without deleting anything |

## What Grit does NOT do

| Never | Because |
|---|---|
| `grit clean` | There is nothing to clean. Storage is controlled by exclusion, not deletion. |
| Destructive pruning | Every snapshot is recoverable by design. |
| "Unreachable objects" | Nothing becomes unreachable — all objects are referenced by the journal. |
| Expiry timers | Your history is yours forever. No auto-delete. |
| Quarantine periods | Deleted files are journaled. Their content stays in the object store. |
| Garbage collection | Not needed. Storage grows only by unique new content. |

## Revised storage architecture

```
~/.grit/
  repos/       ← snapshot manifests + workspace overlays
  objects/     ← immutable, deduplicated, content-addressed, append-only
  manifests/   ← backup manifests, archive indices
  journals/    ← operation log (append-only, never truncated)
  cache/       ← rebuildable working data (not snapshotted)
```

No object is ever removed from `objects/`. The store only grows.
Control is through what is *stored* (exclusion rules), not what is
*deleted* (garbage collection).

## What `rebuildable` means

Rebuildable files (`target/`, `dist/`, `.cache/`) have their paths
and metadata recorded in the snapshot manifest so Grit knows they
existed, but their content is never added to the object store.

A restore from snapshot recreates the rebuildable paths as empty
directories. `cargo build` or `npm install` refills them.

This is the same principle as never deleting — we simply never stored
them in the first place.

## Remote backup behavior

```
grit backup add /Volumes/<backup-drive>/GritBackup
grit backup now
```

Backup copies the full object store incrementally. Objects already
present on the destination are skipped (same BLAKE3 hash). No object
is ever removed from the source repo just because it was backed up.
