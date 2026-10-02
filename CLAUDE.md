# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

[AGENTS.md](AGENTS.md) is the main guide: product principles, agent workflow, commands, repo layout, and code conventions. Read ALL of it before starting ANY work. This file only adds the architecture that you otherwise must piece together from several files.

## Architecture: Rust owns capabilities, TypeScript owns policy

Rust (`apps/desktop/src-tauri/`) gives primitives: graph file IO, SQLite, file watching, keychain, git, and embeddings. All rules over those primitives go in `@reflect/core` (`packages/core/src/`). Before you add a `#[tauri::command]`, check whether the feature is a new rule over commands that already exist. If it is, write it in TypeScript. See `docs/plans/architecture-conventions.md`.

A call goes through these layers. Do not call `invoke` directly from UI code.

```text
React component / hook
  └─ packages/core/src/<domain>/commands.ts   typed binding
       └─ packages/core/src/ipc/invoke.ts     call() + zod validation of the response
            └─ packages/core/src/ipc/bridge.ts  injected transport
                 └─ apps/desktop/src/lib/tauri-bridge.ts
                      └─ #[tauri::command] in src-tauri/src/<module>.rs, registered in lib.rs
```

- Errors: Rust returns `AppError` (`error.rs`), which serializes as `{ kind, message }`. The frontend branches on `kind` through the zod union in `packages/core/src/errors.ts`. Never return bare strings.
- Paths: resolve every graph-relative path through `fs/resolve.rs`. It rejects traversal and symlink escapes.
- Full walkthrough: `docs/contributing/adding-a-command.md`. Settings: `docs/contributing/adding-a-setting.md`.

## The index database

- `<graph>/.reflect/index.sqlite` is a projection of the markdown files, so it can be rebuilt. The exception is the `chat_*` tables, which hold AI chat history. Index wipes and rebuilds must keep them.
- The migrations are in `crates/index-schema/`. The desktop app (writer) and the `reflect` CLI (reader) share them.
- Reads: the frontend builds typed Kysely queries with `@reflect/db`'s `IpcDialect`. These queries run in Rust through the `db_query` command. Writes go through the `index_*` command bindings in `@reflect/core`.
- `packages/db/src/schema.gen.ts` is generated from the Rust migrations. After a migration changes, run `pnpm --filter @reflect/db db:codegen`. Do not edit the file by hand.

## Editor

The editor is Meowdown (`@meowdown/core`, `@meowdown/react`), a separate first-party repo. Before you change editor behavior, read `docs/contributing/editor-architecture.md`. If the root cause is in Meowdown, fix it there.

## Platform notes

- The shipped targets are macOS and iOS. The Rust code also compiles for Windows and Linux, and CI runs clippy and `cargo test --workspace` on Ubuntu. Apple-only features are behind `cfg(target_os = ...)` and do nothing on other platforms: iCloud, Contacts, Calendar, LinkPresentation, and browser native-messaging manifests. On Linux, the keychain is Secret Service (`dbus-secret-service-keyring-store`).
- When you change platform-gated Rust, check the `cfg(not(...))` stub as well. The non-Apple path must still compile and behave.
