# Proposal

## Why

On 2026-10-01, Reflect Dev ran on Omarchy (Linux) with `~/vault` open (3,387 files, 867 folders) and nothing else happening. Within 25 minutes it was using about 18 cores, 16 GB of RAM, 48 GB of swap and 399 threads. systemd-oomd then killed `herdr-server.service`, 736 processes in all, because Reflect had been started from a herdr pane.

The log shows the loop. `watch_start installed` ran 349 times, and every time it was followed by an `index_reconcile_scan` with `candidates=0`. Nothing had changed, yet the scan never stopped.

The cause: `notify` 8.2 uses inotify on Linux and watches with `WatchMask::OPEN` (`notify-8.2.0/src/inotify.rs:427`). Opening a file or folder therefore produces `Access(Open)` events, and closing it after a read produces `Access(Close(Read))`. The handler in `watch_start` passes the paths of every event to `collect_changes`. That function flips `reconcile` for any event on a visible folder. The reconcile walk opens every folder, which creates new open events, which flip `reconcile` again. FSEvents on macOS does not report reads, so the bug cannot appear on the platforms that ship.

The same events also make any outside read of a note, such as Obsidian, an indexer, or `cat`, look like an upsert.

A second loop sits under the first one, inside the watcher thread itself. It came to light during the first live check: with only the first fix in place, the reconcile storm was gone, but the `notify-rs inotify` thread still used a full core, and the vault saw 2.4 million folder opens in 4 seconds. For every event on a path it has not cached, `notify-debouncer-full` 0.7 calls the file-ID cache's `add_path` (`lib.rs:335`). `PrunedFileIdMap` never caches hidden paths such as `.git/objects/…`, so those always miss. `add_path` then walked the path, and walkdir opens a directory before `filter_entry` can reject it. That open is a new event on the same uncached path. In a test, one open of `.git/objects` fed 108,755 more opens in one second.

## What Changes

- The `watch_start` handler drops events that only record a read (`is_read_access`) before it collects paths. It drops `Access(Open)`, `Access(Close(Read))` and `Access(Read)`. It keeps `Access(Close(Write))`, which means a file was written.
- `PrunedFileIdMap::add_path` checks `cache_keeps` before it walks, so a hidden or pruned path costs no open.
- Three tests in `watcher.rs`. One is a unit test of the classifier. The other is a Linux-only test: it starts a real inotify watcher, walks and reads a graph, and checks that every event the walk produces is a read access. The third is also Linux only: it runs a real debouncer with `PrunedFileIdMap`, opens `.git/objects` once, and counts the raw opens that follow with a second watcher. Without the `add_path` fix it fails with 108,755 opens.

## Capabilities

### New Capabilities

- `file-watcher`: the requirement "Reads are not changes".

### Modified Capabilities

None.

## Impact

- `apps/desktop/src-tauri/src/watcher.rs` only. No change to the frontend or to IPC.
- macOS and iOS behave the same as before, because FSEvents sends no read events.
- The two fixes are in their own commits (`310e6aea`, `da6a6c9f`), so it can go upstream to team-reflect separately from the `openspec/` files.
