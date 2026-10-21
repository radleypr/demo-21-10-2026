# 08 — Operations

This document specifies how Ledgerline is run and maintained on a single machine: commands, ports, environment
variables, database location and lifecycle, the maintenance CLIs, backup and restore, and troubleshooting. Toolchain
and configuration values are in [02-Environment-and-Stack.md](02-Environment-and-Stack.md); the HTTP contracts of the
system endpoints are in [05-Backend-and-API.md](05-Backend-and-API.md).

## 1. Commands

All commands run from the project root on Node 24.

| ID | Command | Behaviour |
| --- | --- | --- |
| OPS-01 | `pnpm install` | Installs dependencies and builds the `better-sqlite3` native binding (requires `allowBuilds` in `pnpm-workspace.yaml`) |
| OPS-02 | `pnpm dev` | Development server on `http://localhost:3000`. The database is created and migrated on the first request that needs it |
| OPS-03 | `pnpm build` | Production build. Must not create, open or migrate the database (no `data/` directory appears) |
| OPS-04 | `pnpm start` | Serves the build on port 3000 (`PORT=xxxx` changes it). Database is created and migrated on the first request |
| OPS-05 | `pnpm db:migrate` | Creates the database if missing and applies pending migrations |
| OPS-06 | `pnpm db:seed [--force]` | Loads demo data; refuses on a database that already has data unless `--force` is given |
| OPS-07 | `pnpm db:demo:remove` | Deletes demo rows only; real records are kept |
| OPS-08 | `pnpm db:backup [target-dir]` | Online backup to `data/backups/` (or `target-dir`) |
| OPS-09 | `pnpm db:restore <backup-file>` | Verified restore with a safety copy; the app must be stopped first |
| OPS-10 | `pnpm db:generate --name <name>` | Generates a new SQL migration in `drizzle/` after editing `src/db/schema.ts` |
| OPS-11 | `pnpm typecheck` | `next typegen && tsc --noEmit` |
| OPS-12 | `pnpm lint` | ESLint over the whole project |
| OPS-13 | `pnpm format` / `pnpm format:check` | Prettier write / check |
| OPS-14 | `pnpm test` / `pnpm test:watch` | Vitest unit and integration tests (temporary databases only) |
| OPS-15 | `pnpm test:e2e:install` | Downloads Playwright Chromium (once per machine) |
| OPS-16 | `pnpm test:e2e` | Playwright; starts its own dev server on port 3200 with a throwaway database |
| OPS-17 | `pnpm check` | typecheck, lint, format check and Vitest in sequence; stops at the first failure |

## 2. Ports and network exposure

| Port | Used by | Change it with |
| --- | --- | --- |
| 3000 | `pnpm dev`, `pnpm start` | `pnpm dev -p 3001`, `PORT=3001 pnpm start` (or `pnpm start -p 3001`) |
| 3200 | Playwright web server (`pnpm test:e2e`) | Fixed in `playwright.config.ts`; must be free when the e2e suite starts (`reuseExistingServer: false`) |

**Security:** the app has no authentication and is designed for a single local user. Do not expose it to a network.
To guarantee loopback-only binding, start it with `-H 127.0.0.1` (for example `pnpm start -H 127.0.0.1`). Never
put it behind a public reverse proxy or tunnel.

## 3. Environment variables and path resolution

All variables are optional. Next.js loads `.env` and `.env.local` automatically for `dev`, `build` and
`start`; the `tsx` scripts read only the process environment, so pass variables on the command line for CLIs
(for example `SALES_DB_PATH=/tmp/x.db pnpm db:migrate`).

| Variable | Default | Meaning |
| --- | --- | --- |
| `SALES_DB_PATH` | `data/sales.db` under the project root | Database file. Absolute, or relative to the project root (never to the shell's working directory) |
| `SALES_DASHBOARD_ROOT` | auto-detected | Forces the project root used by every path below |
| `PORT` | `3000` | Port for `next start` |
| `NEXT_TELEMETRY_DISABLED` | unset | Set to `1` by the Playwright web server; optional elsewhere |
| `CI` | unset | When set: Playwright uses 1 retry and the `github` reporter; shadcn CLI runs non-interactively |

Path resolution is implemented in `src/db/paths.ts` with this interface:

| Function | Returns |
| --- | --- |
| `projectRoot(): string` | If `SALES_DASHBOARD_ROOT` is set: its resolved absolute path. Otherwise, walk up from `process.cwd()`: the first directory containing a `package.json` whose `name` is `"sales-dashboard"` is the root. Unreadable or invalid `package.json` files are skipped. If the filesystem root is reached without a match, return `process.cwd()` |
| `dbPath(): string` | `SALES_DB_PATH` resolved against `projectRoot()` when set, else `<root>/data/sales.db` |
| `migrationsDir(): string` | `<root>/drizzle` |
| `backupsDir(): string` | `backups` next to the database file: `dirname(dbPath())/backups` |

Consequence: the app, the CLIs and `next start` launched from any sub-directory all use the same file. Renaming the
package breaks the search; set `SALES_DASHBOARD_ROOT` if the package must have another name.

## 4. Database lifecycle

### 4.1 Opening

`src/db/client.ts` exposes:

| Export | Signature | Behaviour |
| --- | --- | --- |
| `DB` | type | Drizzle better-sqlite3 database typed with the schema, with `$client` as the raw `better-sqlite3` handle |
| `DatabaseInitError` | `class extends Error`, constructor `(message: string, dbFile: string, cause?: unknown)`, `name = "DatabaseInitError"` | Raised for every failure to prepare the database |
| `openDatabase` | `(file: string, migrationsFolder = migrationsDir()) => DB` | Steps below |
| `getDb` | `() => DB` | Process-wide singleton stored on `globalThis` (survives dev hot reloads); re-opens when `dbPath()` changes |

`openDatabase` steps, in order:

1. Unless `file` is `:memory:`, create the parent directory recursively. On failure throw `DatabaseInitError`
   with message `Cannot create the data directory for <file>. Check folder permissions.`
2. Open the file with `better-sqlite3`. On failure throw `DatabaseInitError` with message
   ``Cannot open SQLite database at <file>: <driver message>. If better-sqlite3 failed to load, run `pnpm rebuild better-sqlite3` with the Node version you start the app with.``
3. Set pragmas: `journal_mode = WAL`, `foreign_keys = ON`, `busy_timeout = 5000`, `synchronous = NORMAL`.
4. Apply pending migrations from `migrationsFolder` with the Drizzle migrator (idempotent; applied migrations are
   tracked by Drizzle and never re-run). On failure close the handle and throw `DatabaseInitError` with message
   `Database migration failed for <file>: <message>`.
5. Insert the singleton settings row (`id = 1`, all defaults) if it does not exist.

### 4.2 Request-time access

Every route handler goes through `withDb` (`src/server/api.ts`), which first awaits `connection()` from
`next/server` so the handler is never executed during `next build`, then calls `getDb()`. A
`DatabaseInitError` is logged and returned as HTTP **503** with body `{ "error": <message>, "dbFile": <path> }`.
The UI shows that message in its error state.

### 4.3 Migrations

| Situation | Action |
| --- | --- |
| Normal start | Nothing; pending migrations apply automatically on first open |
| Schema change | Edit `src/db/schema.ts`, run `pnpm db:generate --name <name>`, commit the new SQL and `drizzle/meta` files |
| Restored older backup | `db:restore` migrates the restored file before finishing |

Migrations are forward-only. There is no down migration; recover from a bad migration by restoring a backup.

## 5. Data locations

| Path | Content | Created by |
| --- | --- | --- |
| `data/sales.db` | The database | First request, or any `db:*` script |
| `data/sales.db-wal`, `data/sales.db-shm` | WAL journal and shared memory, present while a connection is open | SQLite |
| `data/backups/sales-<stamp>.db` | Backups | `pnpm db:backup`, Settings → Back up now |
| `data/backups/pre-restore/sales-<stamp>.db` | Safety copy of the database taken before every restore | `pnpm db:restore` |
| `.tmp/e2e/sales.db` | Throwaway e2e database, deleted at the start of each e2e run | Playwright web server |
| `<os tmpdir>/sales-dashboard-test-*/` | Per-test databases, removed after each test | Vitest helpers |

`<stamp>` is the current UTC time in ISO 8601 with every `:` and `.` replaced by `-`, for example
`sales-2026-03-10T14-05-09-123Z.db`. When `SALES_DB_PATH` points elsewhere, `backups/` moves with it (section 3).
Everything under `data/` and every `*.db` file is git-ignored. Settings → Data shows the resolved database path and
backups directory.

## 6. Maintenance CLIs

Each script in `scripts/` opens the database with `openDatabase(dbPath())` (so it also migrates), closes the handle
before exiting, and on any error prints only the error message to stderr and exits with code 1. Exact console output:

| Script | Success output | Failure output |
| --- | --- | --- |
| `db-migrate.ts` | `Database ready and migrated: <file>` | error message, exit 1 |
| `db-seed.ts` | `Seeded demo data into <file>:` followed by the seed result object | When the database has data and `--force` is absent: `Refusing to seed: <file> already contains data. Re-run with --force to add demo rows anyway.`, exit 1 |
| `db-demo-remove.ts` | `Removed demo data:` followed by `{ salesRemoved, productsRemoved, customersRemoved, categoriesRemoved }` | error message, exit 1 |
| `db-backup.ts` | `Backup written: <absolute file>` | error message, exit 1 |
| `db-restore.ts` | `Current database saved to <safety file>` (only when a database exists), then `Restored <backup> -> <target>` | No argument: `Usage: pnpm db:restore <path-to-backup.db>`, exit 1. Invalid backup: verification message, exit 1 |

Demo seeding, demo removal and erase semantics (which rows, promotion of referenced demo rows, typed confirmation)
are defined in [05-Backend-and-API.md](05-Backend-and-API.md).

## 7. Backup

Backups use SQLite's online backup API (`better-sqlite3` `backup(file)`), so they are consistent and include
committed WAL content even while the app is running. Never copy `data/sales.db` with the file system while the app
runs: the copy can miss data still in the `-wal` file.

Required functions in `src/server/services/maintenance.ts`:

| Function | Behaviour |
| --- | --- |
| `backupTo(db: DB, dir: string): Promise<string>` | Creates `dir` recursively, writes `dir/sales-<stamp>.db` through the online backup API, returns the absolute file path |
| `listBackups(dir: string): { name: string; size: number; createdAt: number }[]` | Non-recursive: `*.db` files directly in `dir` (so `pre-restore/` is not listed); `createdAt` is the file mtime in ms; sorted newest first; `[]` when `dir` does not exist |

Ways to back up:

| Path | Procedure | Result |
| --- | --- | --- |
| CLI | `pnpm db:backup` or `pnpm db:backup /path/to/dir` | Prints the file path |
| UI | Settings → Back up now (`POST /api/system/backup`) | HTTP 201 `{ "file": "<basename>" }`; the Settings list shows the five newest backups |
| Off-machine | Copy any file from `data/backups/` to external storage | Backup files are complete standalone SQLite databases |

## 8. Restore

Restore exists only as a CLI; the UI explains it with the text: "To restore, stop the app and run
`pnpm db:restore <backup-file>`. The current database is saved to `backups/pre-restore` first."

Procedure for the operator (order matters):

1. Stop every running `pnpm dev` / `pnpm start` process. A running server keeps the old file open and would keep
   serving or overwriting it.
2. Run `pnpm db:restore <backup-file>` (path relative to the current directory or absolute).
3. Start the app again.

What `db:restore` does, in order:

1. Resolve the argument to an absolute path and verify it with `verifyBackupFile(file: string): void`:
   - open it read-only, requiring the file to exist;
   - `PRAGMA integrity_check` must return `ok`, otherwise throw `Integrity check failed: <result>`;
   - tables `sales`, `sale_items`, `products`, `customers`, `categories` and `settings` must all exist,
     otherwise throw `Not a sales dashboard database (missing table <name>)`;
   - always close the handle.
2. If the target database exists, open it and write a safety backup to `<backupsDir>/pre-restore/` with
   `backupTo`, then close it.
3. Create the target's parent directory, delete any stale `<target>-wal` and `<target>-shm` files, and copy the
   backup over the target.
4. Open the restored file once with `openDatabase` to apply newer migrations, then close it.

To undo a restore, restore the safety copy from `data/backups/pre-restore/` the same way.

## 9. Resetting data

| Goal | Procedure | Effect |
| --- | --- | --- |
| Remove sample data only | Settings → Demo data → Remove, or `pnpm db:demo:remove` | Demo rows deleted; real records kept |
| Delete all records, keep preferences | Settings → Erase all data, typing `DELETE ALL DATA` | All sales, sale lines, products, customers and categories deleted; settings row kept |
| Start from an empty file | Stop the app, back up, delete `data/sales.db`, `data/sales.db-wal`, `data/sales.db-shm`, start the app | New empty database with default settings |

**Warning:** erasing data and deleting database files are irreversible. Take a backup first (section 7); deleting
only `sales.db` while leaving its `-wal` file behind can corrupt the next database created at that path.

## 10. Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| `Could not locate the bindings file` | pnpm skipped the `better-sqlite3` build script | Check `allowBuilds.better-sqlite3: true` in `pnpm-workspace.yaml`, then `pnpm rebuild better-sqlite3` |
| `NODE_MODULE_VERSION` mismatch, or "was compiled against a different Node.js version" | Binding built for another Node major | Switch to Node 24 (`nvm use`), then `pnpm rebuild better-sqlite3`, or delete `node_modules` and reinstall |
| `gyp` / compiler errors during install | No prebuild for this platform and no native toolchain | Install the build tools from 02 section 1 (ENV-04) and reinstall |
| pnpm warns that build scripts were ignored | `pnpm-workspace.yaml` missing or incomplete | Restore it (02 section 4.11); approve only `better-sqlite3` and `esbuild` |
| API returns 503 with "Cannot open SQLite database at …" or "Cannot create the data directory …" | Path not writable, is a directory, or parent missing permissions | Fix permissions or point `SALES_DB_PATH` to a writable file path |
| API returns 503 with "Database migration failed for …" | Database from an incompatible schema or a damaged file | Restore a known-good backup (section 8) |
| `SQLITE_BUSY` / "database is locked" | Another process holds a write lock longer than the 5 s busy timeout (for example a restore while the app runs) | Stop the other process; never run `db:restore` while the app is running |
| App shows different data than a CLI | Different database files (different `SALES_DB_PATH` or package renamed) | Compare the path in Settings → Data with the CLI output; keep `name: "sales-dashboard"` or set `SALES_DASHBOARD_ROOT` |
| `data/sales.db` appears after `pnpm build` | A route handler touched the database without `await connection()` | Route every handler through `withDb` |
| `pnpm db:seed` exits with "Refusing to seed" | Database already has data | Intended. Use `--force` to add demo rows anyway, or load demo data from Settings |
| `db:restore` fails with "Integrity check failed" or "Not a sales dashboard database" | Damaged file or wrong file | Pick another backup; nothing was changed |
| Port already in use (`EADDRINUSE`) | Another server on 3000 or 3200 | `pnpm dev -p 3001` / `PORT=3001 pnpm start`; for e2e stop whatever holds 3200 |
| Playwright: `Executable doesn't exist` | Chromium not downloaded | `pnpm test:e2e:install` |
| e2e times out waiting for the web server | First compile slower than the timeout, or port 3200 busy | Free port 3200; re-run (timeout is 180 s) |
| Hydration warning about the `class` attribute on `<html>` | Theme class set before hydration | `<html>` must carry `suppressHydrationWarning` |
| `next dev` modifies `AGENTS.md` | Next.js maintains a managed block in that file | Expected; commit the file |
| `tsc` errors about `RouteContext` or `LayoutProps` | Route types not generated | Use `pnpm typecheck` (runs `next typegen` first) |

## 11. Limits of operation

| Limit | Detail |
| --- | --- |
| Users | One; no authentication or roles |
| Processes | One app server per database file; CLIs may run alongside except `db:restore` |
| Currency | One reporting currency at a time, 2-decimal currencies only, no FX conversion |
| Refunds | Full refunds only (status change) |
| Inventory | Not tracked |

See [../../BUILD_PLAN.md](../../BUILD_PLAN.md) for when each command is used during the build and [09-Gotchas.md](09-Gotchas.md)
for stack pitfalls.
