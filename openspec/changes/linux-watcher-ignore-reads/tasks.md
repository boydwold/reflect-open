# Tasks

## 1. Tests

- [x] 1.1 Add `read_access_is_not_a_change` (a unit test of the classifier) and `listing_the_graph_produces_no_change_events` (Linux only, with a real inotify watcher) to `watcher.rs`. Verify: both pass, and the Linux test also confirms that inotify does report the walk's opens (`read_events > 0`). (Written together with the fix, not before it.)

## 2. The fix

- [x] 2.1 Add `is_read_access` and filter events with it in the `watch_start` handler before `collect_changes`. Verify: `cargo test -p reflect-open --lib watcher` gives 19 passed. Commit `310e6aea`.
- [x] 2.3 The first live check (3.1) failed: no reconcile storm, but the `notify-rs inotify` thread used one full core, and `inotifywait` counted 2.4 million folder opens in 4 seconds, mostly in `.git/objects`. Add `opening_a_hidden_folder_does_not_feed_itself`. Verify: it fails without the fix (`one open fed 108755 more opens`).
- [x] 2.4 Make `PrunedFileIdMap::add_path` check `cache_keeps` before walking. Verify: the test from 2.3 passes, and `cargo test -p reflect-open --lib watcher` gives 20 passed. Commit `da6a6c9f`.
- [x] 2.2 Run `cargo clippy -p reflect-open --all-targets -- -D warnings` and `cargo fmt --all -- --check`. Verify: both are clean. (fmt is clean. clippy is clean except for a deprecation of `fetch_update` in `web_fetch.rs:205`, which was there before this change and fails `-D warnings` on Rust 1.99. It is out of scope here, so the check ran with `-A deprecated`.)

## 3. Live checks

- [x] 3.1 Rebuild Reflect Dev and run it in its own memory-limited scope (`systemd-run --user --scope -p MemoryMax=4G`), not inside herdr, with `~/vault` open and idle for 5 minutes. Verify: the log has at most a few `watch_start installed` lines, against 349 in 25 minutes before; RSS and thread count stay flat; the app is near 0% CPU when idle; and `inotifywait` counts only a few vault opens in 4 seconds. The first run failed (see 2.3). After `da6a6c9f`: 1 watch_start, 0% CPU, 229 MB, 4 opens in 4 seconds.
- [x] 3.2 With the app open, edit a note in another program. Verify: the change shows in Reflect, and the log has no reconcile storm. Checked against the index rather than the UI: the note entered and left `notes`.
- [x] 3.3 Write `evidence.md` with the commit, the test counts and the live results. Verify: `openspec validate linux-watcher-ignore-reads --strict` passes.
