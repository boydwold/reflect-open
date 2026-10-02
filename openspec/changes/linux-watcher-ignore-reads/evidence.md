# Evidence

## Commits

- `310e6aea`: fix: stop the file watcher looping on Linux when files are only read
- `da6a6c9f`: fix: stop the watcher thread re-opening hidden folders on Linux

Branch `fix/linux-watcher-ignore-reads` on `boydwold/reflect-open`.

## Before (2026-10-01, Omarchy, `~/vault`: 3,387 files, 867 folders)

| Measure | Value |
|---|---|
| CPU | about 1,826% (about 18 cores) |
| RAM / swap | 16 GB / 48 GB |
| Threads | 399 |
| `watch_start installed` in 25 minutes | 349 |
| Outcome | systemd-oomd killed `herdr-server.service` (736 processes) at 20:16 |

## Between the two fixes (only `310e6aea`)

| Measure | Value |
|---|---|
| `watch_start` / reconcile in 2.5 minutes | 1 / 1 |
| RSS / threads | 270 MB / 41, flat |
| CPU | about 100%, all in the `notify-rs inotify` thread |
| Vault folder opens in 4 seconds (`inotifywait -r -e open`) | 2,399,130 (1,056,124 in `.git/objects`) |

## Tests

- `cargo test -p reflect-open --lib watcher`: 20 passed.
- `opening_a_hidden_folder_does_not_feed_itself` without `da6a6c9f`: failed, `one open fed 108755 more opens`. With it: passes.
- `cargo fmt --all -- --check`: clean.
- `cargo clippy -p reflect-open --all-targets -- -D warnings -A deprecated`: clean. Without `-A deprecated` it fails only on `web_fetch.rs:205` (`fetch_update` is deprecated in Rust 1.99). That code was already there; it is not part of this change.

## Live check after both fixes (task 3.1)

Release build, run in `systemd-run --user --scope -p MemoryMax=4G -p MemorySwapMax=0`, `~/vault` open, idle, sampled every 30 seconds from 20:41:48 to 20:46:36 (MDT).

| Measure | Value |
|---|---|
| CPU (sampled over 2 seconds) | 0% in all 10 samples |
| RSS | 229 MB in all 10 samples |
| Threads | 41–42 |
| `watch_start installed` / `index_reconcile_scan` | 1 / 1 |
| Vault opens in 4 seconds | 4 |
| Scope memory peak | 1.13 GB (startup index pass included) |

## Outside edit (task 3.2)

A note `zz-reflect-watch-test.md` written from the shell appeared in `notes` in the index, and left it after `rm`. The counts of `watch_start` and reconcile stayed at 1.
