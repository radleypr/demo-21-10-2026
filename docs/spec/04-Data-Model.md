# 04 — Data Model

This document defines the persistent data of Ledgerline: the SQLite schema, Drizzle conventions, the migration
workflow, database path resolution, the connection lifecycle, and backup and restore. Domain formulas that produce the
stored amounts are in [03-Domain-Rules.md](03-Domain-Rules.md). Service behaviour and HTTP contracts are in
[05-Backend-and-API.md](05-Backend-and-API.md). Operator commands are in [08-Operations.md](08-Operations.md).

## 1. Required files

| File | Responsibility |
| --- | --- |
| `src/db/schema.ts` | Drizzle table definitions, indexes, checks and exported row types. The only place the schema is authored |
| `src/db/paths.ts` | `projectRoot()`, `dbPath()`, `migrationsDir()`, `backupsDir()` |
| `src/db/client.ts` | `openDatabase()`, `getDb()`, `DatabaseInitError`, `DB` type |
| `src/server/db.ts` | Re-exports `getDb` and starts with `import "server-only"`, so client code can never import the driver |
| `src/server/services/maintenance.ts` | `stats`, `removeDemoData`, `eraseAllData`, `backupTo`, `listBackups`, `verifyBackupFile` |
| `drizzle.config.ts` | drizzle-kit configuration (section 4) |
| `drizzle/0000_init.sql`, `drizzle/meta/_journal.json`, `drizzle/meta/0000_snapshot.json` | Generated migration, committed |
| `scripts/db-migrate.ts`, `scripts/db-backup.ts`, `scripts/db-restore.ts` | CLI entry points run with `tsx` |

Scripts in `scripts/` must import `src/db/*` and `src/server/services/maintenance.ts` directly, never
`src/server/db.ts`: the `server-only` marker throws outside the Next.js server.

## 2. Schema conventions

| ID | Rule |
| --- | --- |
| DM-01 | Every entity table except `settings` has a `text` primary key `id`, generated in the app with `crypto.randomUUID()` (Drizzle `$defaultFn`). No autoincrement integers |
| DM-02 | Timestamps are `integer` epoch milliseconds, never SQLite date strings. `created_at` and `updated_at` are `NOT NULL` and have no SQL default: Drizzle sets them with `$defaultFn(() => Date.now())`, and `updated_at` also with `$onUpdateFn(() => Date.now())`. Use plain integer columns holding numbers, not `mode: "timestamp"` |
| DM-03 | Booleans are `integer` columns in Drizzle `mode: "boolean"`, `NOT NULL DEFAULT false` (stored 0/1) |
| DM-04 | Money columns end in `_minor` and hold integer minor units. Tax rate is `tax_rate_bp` (basis points) |
| DM-05 | Enumerated text columns use Drizzle `text(..., { enum: [...] })` for typing **and** a SQL `CHECK ... IN (...)` for enforcement |
| DM-06 | Column names are snake_case in SQL and camelCase in TypeScript (`orderRef` ↔ `order_ref`) |
| DM-07 | Every row a user can create carries `is_demo` (default false). Demo rows are only ever written by the demo seeder |
| DM-08 | Foreign keys use `ON UPDATE no action`. `ON DELETE` is `restrict` unless listed otherwise; nothing that would erase sales history cascades |
| DM-09 | Sale lines **snapshot** product name, category name, unit price and unit cost. Renaming a product or category never rewrites past sales |
| DM-10 | Export the row types `Category`, `Product`, `Customer`, `Sale`, `SaleItem`, `Settings` as `typeof table.$inferSelect` |

Legend for the tables below: NN = `NOT NULL`; "app" default = set by Drizzle, not by SQL.

## 3. Tables

### 3.1 `categories`

| Column | SQLite type | Null | Default | Notes |
| --- | --- | --- | --- | --- |
| `id` | text | NN | app UUID | PK |
| `name` | text | NN | — | Trimmed, 1–60 chars (validated in the app) |
| `is_demo` | integer | NN | `false` | |
| `created_at` | integer | NN | app `Date.now()` | |
| `updated_at` | integer | NN | app `Date.now()`, on update | |

| Index | Columns | Kind |
| --- | --- | --- |
| `categories_name_nocase` | `name COLLATE NOCASE` | UNIQUE. "Coffee" and "coffee" conflict. Build it in Drizzle with `uniqueIndex("categories_name_nocase")` on a raw `sql` template expression `name COLLATE NOCASE` |

### 3.2 `products`

| Column | SQLite type | Null | Default | Constraint |
| --- | --- | --- | --- | --- |
| `id` | text | NN | app UUID | PK |
| `name` | text | NN | — | 1–120 chars |
| `sku` | text | null | — | max 64; unique when present |
| `category_id` | text | null | — | FK `categories.id` ON DELETE restrict |
| `description` | text | null | — | max 1000 |
| `default_price_minor` | integer | NN | — | CHECK `products_price_nonneg`: `>= 0` |
| `default_cost_minor` | integer | null | — | CHECK `products_cost_nonneg`: `IS NULL OR >= 0`. `null` = cost unknown |
| `archived` | integer (boolean) | NN | `false` | Archived products are hidden from pickers but kept for history |
| `is_demo` | integer (boolean) | NN | `false` | |
| `created_at`, `updated_at` | integer | NN | app | |

| Index | Columns | Kind |
| --- | --- | --- |
| `products_sku_unique` | `sku` | UNIQUE (SQLite allows many `NULL`s, so products without SKU never conflict). Case-sensitive |
| `products_category_idx` | `category_id` | plain |
| `products_name_idx` | `name` | plain |

### 3.3 `customers`

| Column | SQLite type | Null | Default | Constraint |
| --- | --- | --- | --- | --- |
| `id` | text | NN | app UUID | PK |
| `name` | text | NN | — | 1–120 chars |
| `email` | text | null | — | valid email when present; **not** unique |
| `phone` | text | null | — | max 40 |
| `notes` | text | null | — | max 2000 |
| `is_demo` | integer (boolean) | NN | `false` | |
| `created_at`, `updated_at` | integer | NN | app | `created_at` is "Customer since" |

| Index | Columns | Kind |
| --- | --- | --- |
| `customers_name_idx` | `name` | plain |

### 3.4 `sales`

| Column | SQLite type | Null | Default | Notes |
| --- | --- | --- | --- | --- |
| `id` | text | NN | app UUID | PK |
| `order_ref` | text | NN | — | Unique. User-entered (max 64) or generated |
| `occurred_at` | integer | NN | — | Epoch ms of the sale; all reporting filters on it |
| `customer_id` | text | null | — | FK `customers.id` ON DELETE restrict. `null` = walk-in |
| `customer_name` | text | null | — | Snapshot of the customer name, or the typed walk-in name; `null` for an anonymous walk-in |
| `payment_method` | text | NN | — | CHECK `sales_payment_method_valid`: `IN ('card','cash','bank_transfer','e_wallet','other')` |
| `status` | text | NN | — | CHECK `sales_status_valid`: `IN ('completed','pending','cancelled','refunded')` |
| `subtotal_minor` | integer | NN | — | Σ line subtotals |
| `discount_minor` | integer | NN | `0` | Sale-level discount before tax |
| `net_minor` | integer | NN | — | subtotal − discount; this is revenue |
| `tax_rate_bp` | integer | NN | `0` | 0–10000 |
| `tax_minor` | integer | NN | `0` | Round-half-up of net × rate |
| `total_minor` | integer | NN | — | net + tax |
| `cost_minor` | integer | null | — | Σ line costs; `null` when any line cost is unknown |
| `notes` | text | null | — | max 2000 |
| `is_demo` | integer (boolean) | NN | `false` | |
| `created_at`, `updated_at` | integer | NN | app | |

Table checks:

| Check | Expression |
| --- | --- |
| `sales_amounts_nonneg` | `subtotal_minor >= 0 AND discount_minor >= 0 AND net_minor >= 0 AND tax_minor >= 0` |
| `sales_discount_le_subtotal` | `discount_minor <= subtotal_minor` |
| `sales_status_valid` | as above |
| `sales_payment_method_valid` | as above |

| Index | Columns | Kind |
| --- | --- | --- |
| `sales_order_ref_unique` | `order_ref` | UNIQUE |
| `sales_occurred_at_idx` | `occurred_at` | plain |
| `sales_status_occurred_idx` | `status, occurred_at` | plain (serves every "completed in range" metric) |
| `sales_customer_idx` | `customer_id` | plain |

Rules:

- Amount columns are always written from `computeSaleTotals` on the server ([03-Domain-Rules.md](03-Domain-Rules.md)
  section 2.4); client-sent totals are never trusted or accepted.
- Generated order references are `ORD-` followed by 8 uppercase hex characters from 4 random bytes
  (`ORD-3FA9C01B`); regenerate until unused. Demo sales use `DEMO-` plus a 5-digit zero-padded sequence
  (`DEMO-00001`).
- A refund is a status change to `refunded` on the same sale row. No negative or offsetting sale is ever inserted.

### 3.5 `sale_items`

| Column | SQLite type | Null | Default | Notes |
| --- | --- | --- | --- | --- |
| `id` | text | NN | app UUID | PK |
| `sale_id` | text | NN | — | FK `sales.id` **ON DELETE cascade** (lines belong to their sale) |
| `product_id` | text | null | — | FK `products.id` ON DELETE restrict |
| `category_id` | text | null | — | FK `categories.id` **ON DELETE set null** |
| `product_name` | text | NN | — | Snapshot |
| `category_name` | text | null | — | Snapshot; `null` = uncategorized |
| `quantity` | integer | NN | — | CHECK `sale_items_qty_pos`: `> 0` |
| `unit_price_minor` | integer | NN | — | CHECK `sale_items_price_nonneg`: `>= 0`. Snapshot of the price actually charged |
| `unit_cost_minor` | integer | null | — | Snapshot; `null` = unknown |
| `line_subtotal_minor` | integer | NN | — | quantity × unit price |
| `line_net_minor` | integer | NN | — | Line subtotal minus its share of the sale discount |
| `line_cost_minor` | integer | null | — | quantity × unit cost, or `null` |
| `position` | integer | NN | `0` | 0-based order of the line in the form |

`sale_items` has no `is_demo` or timestamp columns; it inherits both from its sale.

| Index | Columns |
| --- | --- |
| `sale_items_sale_idx` | `sale_id` |
| `sale_items_product_idx` | `product_id` |
| `sale_items_category_idx` | `category_id` |

Updating a sale deletes all of its lines and inserts the new set, inside one transaction with the sale update. Lines
are always read ordered by `position`.

### 3.6 `settings`

| Column | SQLite type | Null | Default | Notes |
| --- | --- | --- | --- | --- |
| `id` | integer | NN | — | PK. CHECK `settings_singleton`: `id = 1` |
| `currency` | text | NN | `'USD'` | One of `CURRENCIES` |
| `locale` | text | NN | `'en-US'` | One of `LOCALES` |
| `date_format` | text | NN | `'MMM d, yyyy'` | One of `DATE_FORMATS` |
| `default_range` | text | NN | `'30d'` | Any range preset except `custom` |
| `updated_at` | integer | NN | app `Date.now()`, on update | |

Exactly one row exists, inserted by `openDatabase` (section 6). These defaults are SQL defaults, so inserting
`{ id: 1 }` alone produces a complete row. Changing the currency relabels amounts only; nothing is converted.

### 3.7 Relationships and delete rules

| Parent → child | ON DELETE | Service-level rule when deleting the parent (HTTP 409 message) |
| --- | --- | --- |
| `categories` → `products.category_id` | restrict | Blocked while products reference it: "This category is used by N product(s). Reassign them first." |
| `categories` → `sale_items.category_id` | set null | Snapshot `category_name` keeps reporting intact |
| `products` → `sale_items.product_id` | restrict | Blocked when the product appears in any sale: "This product appears in past sales. Archive it instead to keep history intact." |
| `customers` → `sales.customer_id` | restrict | Blocked when the customer has sales: "This customer has N sale(s). Customers with history cannot be deleted." |
| `sales` → `sale_items.sale_id` | cascade | Deleting a sale removes its lines |

Uniqueness messages (HTTP 409), checked by the service before writing, with the database constraint as the backstop:

| Constraint | Message |
| --- | --- |
| `categories_name_nocase` | `Category "<name>" already exists` |
| `products_sku_unique` | `SKU "<sku>" is already used by another product` |
| `sales_order_ref_unique` | `Order reference "<ref>" is already used` |

Creating a sale with a new product (or a new category name typed inside the sale form) creates those rows in the same
transaction; a typed category name matches an existing category case-insensitively before a new one is created.

## 4. Migration workflow

`drizzle.config.ts` values:

| Key | Value |
| --- | --- |
| `dialect` | `"sqlite"` |
| `schema` | `"./src/db/schema.ts"` |
| `out` | `"./drizzle"` |
| `dbCredentials.url` | `process.env.SALES_DB_PATH ?? "data/sales.db"` |

Package scripts: `"db:generate": "drizzle-kit generate"`, `"db:migrate": "tsx scripts/db-migrate.ts"`,
`"db:backup": "tsx scripts/db-backup.ts"`, `"db:restore": "tsx scripts/db-restore.ts"` (seed and demo-removal scripts
are in [08-Operations.md](08-Operations.md)). Versions: drizzle-orm `0.45.4`, drizzle-kit `0.31.11`.

Steps:

1. Write the complete schema in `src/db/schema.ts` (all tables, indexes and checks in section 3).
2. Run `pnpm drizzle-kit generate --name init` **once**. It must produce `drizzle/0000_init.sql`,
   `drizzle/meta/_journal.json` (one entry, tag `0000_init`) and `drizzle/meta/0000_snapshot.json`. Statements in
   the SQL file are separated by `--> statement-breakpoint`.
3. Inspect the SQL: six `CREATE TABLE` statements with every `CHECK` and `FOREIGN KEY ... ON DELETE` clause from
   section 3, and every index, including `CREATE UNIQUE INDEX categories_name_nocase ON categories (name COLLATE NOCASE)`.
4. Commit all three files. They are part of the source.
5. Never hand-edit generated SQL or snapshots. A later schema change means editing `schema.ts` and running
   `pnpm db:generate` to add `0001_*.sql`.
6. The app migrates automatically when it opens the database (section 6); `pnpm db:migrate` does the same from the
   CLI and prints `Database ready and migrated: <file>`.

Get every constraint right in `0000`. SQLite cannot add a `CHECK` with `ALTER TABLE`; drizzle-kit then copies the
table into a new one and drops the old one, and dropping `sales` while foreign keys are on cascade-deletes all `sale_items`. A later migration that must
replace a parent table this way has to wrap it in `PRAGMA foreign_keys=OFF` / `ON`.

## 5. Path resolution (`src/db/paths.ts`)

| Signature | Algorithm |
| --- | --- |
| `projectRoot(): string` | 1. If `SALES_DASHBOARD_ROOT` is set and non-empty, return `path.resolve(it)`. 2. Otherwise start at `process.cwd()` and walk up one directory at a time; at each level, if `package.json` exists, can be read and parsed, and its `name` is `"sales-dashboard"`, return that directory. An unreadable or invalid `package.json` is skipped and the walk continues. 3. When the filesystem root is reached without a match, return `process.cwd()` |
| `dbPath(): string` | If `SALES_DB_PATH` is set and non-empty: `path.resolve(projectRoot(), SALES_DB_PATH)` (absolute values are kept, relative values resolve against the project root, not the cwd). Otherwise `<projectRoot>/data/sales.db` |
| `migrationsDir(): string` | `<projectRoot>/drizzle` |
| `backupsDir(): string` | `<dirname(dbPath())>/backups`, so a temporary DB keeps its backups beside it |

The `package.json` `name` must therefore be exactly `sales-dashboard`. Read environment variables on every call (not at
import time) so tests can change them. Examples with root `/home/u/app`: no env → `/home/u/app/data/sales.db`;
`SALES_DB_PATH=.tmp/e2e/sales.db` → `/home/u/app/.tmp/e2e/sales.db` with backups in `/home/u/app/.tmp/e2e/backups`;
`SALES_DB_PATH=/var/x.db` → `/var/x.db`. Running a script from `/home/u/app/scripts` still resolves `/home/u/app`.

`.gitignore` must contain `/data/`, `*.db`, `*.db-wal` and `*.db-shm`.

## 6. Connection lifecycle (`src/db/client.ts`)

Types:

| Export | Shape |
| --- | --- |
| `DB` | `BetterSQLite3Database<typeof schema> & { $client: Database.Database }` |
| `DatabaseInitError` | `class extends Error`, constructor `(message: string, dbFile: string, cause?: unknown)`, `name = "DatabaseInitError"`, public `dbFile` |
| `openDatabase(file: string, migrationsFolder: string = migrationsDir()): DB` | Open sequence below |
| `getDb(): DB` | Process singleton below |

`openDatabase` steps, in order:

| Step | Action | On failure throw `DatabaseInitError` with message |
| --- | --- | --- |
| 1 | Unless `file === ":memory:"`, create `dirname(file)` recursively | `Cannot create the data directory for <file>. Check folder permissions.` |
| 2 | `new Database(file)` (better-sqlite3) | ``Cannot open SQLite database at <file>: <error message>. If better-sqlite3 failed to load, run `pnpm rebuild better-sqlite3` with the Node version you start the app with.`` (the inner backticks are part of the message) |
| 3 | Pragmas: `journal_mode = WAL`, `foreign_keys = ON`, `busy_timeout = 5000`, `synchronous = NORMAL` | — |
| 4 | `drizzle(sqlite, { schema })`, then the better-sqlite3 migrator `migrate(db, { migrationsFolder })` | Close the handle, then `Database migration failed for <file>: <error message>` |
| 5 | Insert `settings { id: 1 }` with `onConflictDoNothing()` | — |
| 6 | Return the Drizzle instance | — |

`getDb()`:

- Keep the instance and its file path on `globalThis` (keys `__salesDb` and `__salesDbFile`) so dev-server hot reloads
  do not open a new handle per module reload.
- On each call compute `dbPath()`; if it differs from the cached file (tests switching `SALES_DB_PATH`), close the old
  handle and open the new one.
- Open lazily, on the first request that needs data. Never open at module import or during `next build`; route
  handlers and pages that read data call `await connection()` first (see [09-Gotchas.md](09-Gotchas.md)).
- Route handlers map `DatabaseInitError` to HTTP 503 with body `{ "error": <message>, "dbFile": <path> }`; pages show
  the same message with the path (see [05-Backend-and-API.md](05-Backend-and-API.md)).

Tests call `openDatabase(<tmpdir>/test.db)` directly against the real `drizzle/` folder, so every integration test runs
the real migration.

## 7. Maintenance service (`src/server/services/maintenance.ts`)

| Signature | Behaviour |
| --- | --- |
| `stats(db): { sales, demoSales, products, customers, categories, demoRows }` | Row counts; `demoRows` = demo sales + demo products + demo customers + demo categories |
| `removeDemoData(db): { salesRemoved, productsRemoved, customersRemoved, categoriesRemoved }` | One transaction: (1) delete demo sales (lines cascade); (2) demo products still referenced by any `sale_items` row get `is_demo = false` (promoted, kept); demo customers still referenced by any sale likewise; (3) delete the remaining demo products and customers; (4) promote demo categories still referenced by any product; (5) delete the remaining demo categories. Counts are deleted rows only. User-entered sales that used demo products therefore keep working |
| `eraseAllData(db): void` | One transaction deleting all rows of `sale_items`, `sales`, `products`, `customers`, `categories`, in that order. `settings` is kept |
| `backupTo(db, dir): Promise<string>` | Create `dir` recursively; stamp = `new Date().toISOString()` with every `:` and `.` replaced by `-`; file = `<dir>/sales-<stamp>.db` (e.g. `sales-2026-03-15T14-30-00-000Z.db`); write it with the better-sqlite3 online backup API `await db.$client.backup(file)` (consistent while the app runs; never copy a live WAL database file); return the absolute file path |
| `listBackups(dir): { name, size, createdAt }[]` | `[]` when `dir` does not exist. Every `*.db` file directly in `dir` (sub-folders such as `pre-restore/` are not listed); `size` in bytes, `createdAt` = file `mtimeMs`; newest first |
| `verifyBackupFile(file): void` | Open read-only with `fileMustExist: true`; `PRAGMA integrity_check` (simple value) must equal `"ok"`, else throw `Integrity check failed: <result>`; each of `sales`, `sale_items`, `products`, `customers`, `categories`, `settings` must exist in `sqlite_master`, else throw `Not a sales dashboard database (missing table <name>)`; always close the handle |

API use: `GET /api/system` returns `{ dbPath, backupsDir, stats, backups }`; `POST /api/system/backup` returns 201
`{ "file": "<basename>" }`. Full contracts are in [05-Backend-and-API.md](05-Backend-and-API.md).

## 8. Backup and restore commands

### 8.1 `pnpm db:backup [dir]`

1. `openDatabase(dbPath())` (creates and migrates the DB if missing).
2. `backupTo(db, dir ?? backupsDir())`; `dir` is the first CLI argument when given.
3. Close the handle and print `Backup written: <file>`.

### 8.2 `pnpm db:restore <path-to-backup.db>`

The app must be stopped first: restore replaces the database file underneath any open handle.

1. No argument: print `Usage: pnpm db:restore <path-to-backup.db>` and exit with code 1.
2. Resolve the argument to an absolute path and run `verifyBackupFile` on it.
3. Target = `dbPath()`. If the target exists, open it with `openDatabase`, `backupTo` `<backupsDir()>/pre-restore`,
   close it, and print `Current database saved to <safety file>`. Nothing is overwritten without this safety copy.
4. Create the target directory if needed.
5. Delete `<target>-wal` and `<target>-shm` if present (stale WAL pages would corrupt the restored file).
6. Copy the backup file over the target.
7. `openDatabase(target)` then close, which applies any migrations newer than the backup.
8. Print `Restored <backup> -> <target>` and exit 0.

Any thrown error prints its message and exits with code 1. Example: restoring `data/backups/sales-2026-03-15T14-30-00-000Z.db`
writes `data/backups/pre-restore/sales-<now>.db` first, then replaces `data/sales.db`.

## 9. Integrity checklist

- DM-C1. A fresh `openDatabase` on an empty path yields six tables, every index in section 3 and exactly one settings
  row with defaults USD, en-US, `MMM d, yyyy`, 30d.
- DM-C2. Opening the same file twice runs no migration the second time and keeps one settings row.
- DM-C3. Direct SQL inserts violating any `CHECK` (negative price, quantity 0, discount above subtotal, unknown status
  or payment method, `settings.id = 2`) fail with `SQLITE_CONSTRAINT`.
- DM-C4. Deleting a product or customer referenced by sales fails at the database even if the service check were
  bypassed; deleting a sale removes its lines; deleting an unused category referenced only by old sale lines sets
  their `category_id` to `null` and leaves `category_name` intact.
- DM-C5. Inserting categories "Coffee" and "coffee" fails; two products with `sku = null` succeed.
- DM-C6. `backupTo` followed by `verifyBackupFile` on the new file succeeds.

Test cases covering these are listed in [07-Testing.md](07-Testing.md).
