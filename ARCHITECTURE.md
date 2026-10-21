# Ledgerline — Solution Architecture Design

Companion to [`docs/spec/`](docs/spec/). Where this document and a spec disagree, the spec wins (see the conflict
rule in [`CONTRIBUTING.md`](CONTRIBUTING.md#reading-order)).

## 1. Architecture Principles

| # | Principle | Consequence |
| --- | --- | --- |
| P1 | **Local-first, zero-config** | One process, one SQLite file; no external services, env vars or accounts |
| P2 | **Server is the source of truth** | All validation and metrics live in server services; the client only fetches and renders |
| P3 | **Integer money** | Minor units in DB, API and math; formatting happens only at the edge |
| P4 | **Immutable history** | Line items snapshot product data; FKs restrict deletes of referenced records |
| P5 | **One contract** | UI, CSV export and CLI scripts reuse the same services; the UI talks to them only over HTTP JSON |
| P6 | **Testable core** | Services are pure functions of `(db, input)` and run in Vitest on temp DBs |
| P7 | **Boring and minimal** | No abstractions, features or dependencies beyond the spec |

## 2. System Context

```
            ┌──────────────────────── one machine ────────────────────────┐
  User ───▶ │ Browser (React client components)                           │
            │    │  TanStack Query fetch (JSON) / CSV downloads           │
            │    ▼                                                        │
            │ Next.js 16 server (Node runtime)                            │
            │    /api/* route handlers ─▶ withDb() ─▶ services ─▶ Drizzle │
            │                                                   │         │
            │ pnpm db:* scripts (tsx) ─────────▶ services ──────┤         │
            │                                                   ▼         │
            │                     data/sales.db (+ -wal, -shm, backups/)  │
            └─────────────────────────────────────────────────────────────┘
```

There are no external systems. The only network use is `pnpm install` and the one-time Playwright browser download.

## 3. Load & Data Characterisation

| Dimension | Design point |
| --- | --- |
| Users | 1 (single-user, no auth) |
| Data volume | demo seed ≈ 1,600 sales / 400 days; comfortably handles 100k+ sales thanks to indexes on `occurred_at` and `(status, occurred_at)` |
| Query pattern | read-heavy aggregates over a date range; small transactional writes |
| Concurrency | one Node process; WAL lets backups and reads proceed during writes; `busy_timeout 5000` |

## 4. Functional Building Blocks

| Block | Pages / endpoints | Services |
| --- | --- | --- |
| Sales recording | sale sheet, `/api/sales*` | `sales.ts` |
| Sales ledger | `/sales`, `/api/sales`, `/api/sales/export` | `sales.ts`, `exports.ts` |
| Catalog | `/products`, `/api/products*`, `/api/categories*` | `catalog.ts` |
| Customers | `/customers`, `/api/customers*` | `customers.ts` |
| Reporting | `/`, `/analytics`, `/api/dashboard`, `/api/reports/[report]` | `analytics.ts`, `exports.ts` |
| Administration | `/settings`, `/api/settings`, `/api/system*` | `settings.ts`, `maintenance.ts`, `demo-seed.ts` |

## 5. Component Decisions by Layer

### 5.1 Client layer

- **Pages** are server components that only export `metadata` and render `<Suspense fallback={<PageSkeleton/>}>` around a
  `"use client"` view. Cache Components prerenders a static shell; data streams in on the client.
- **Server state**: TanStack Query hooks in `src/lib/queries.ts`. Query keys include every server-side input. After
  a mutation, `useInvalidateData()` invalidates everything except `settings`.
- **UI state**: Zustand `useUiStore` (persisted: sidebar, date range, chart type, page size; `skipHydration` and
  rehydrate after mount) and `useSaleDialog` (one global sale sheet).
- **Time**: read only after mount through a `useSyncExternalStore` minute clock whose server snapshot is `0`, so
  prerendering never reads the current time.
- **Design system**: shadcn/ui primitives + Tailwind 4 tokens in `globals.css` (cool neutrals, indigo accent,
  5-colour chart palette, light/dark).

### 5.2 Application layer

- **Route handlers** (`src/app/api/**/route.ts`) are thin: parse the query/body, call a service, return
  `Response.json` or CSV. Every handler is wrapped in `withDb()`, which calls `await connection()` (opting out of
  prerendering) and maps errors: Zod → 400, `AppError` → its status, `DatabaseInitError` → 503, anything else → 500.
- **Services** (`src/server/services/*`) are synchronous, take `db` first, and write inside transactions. They never
  import `server-only`, so tests can import them.
- **Validation**: one Zod module (`src/lib/validation.ts`) holds the wire schemas (server) and form schemas (client)
  plus converters.

### 5.3 Data layer

- SQLite through better-sqlite3 (sync), with pragmas WAL, `foreign_keys=ON`, `busy_timeout=5000`,
  `synchronous=NORMAL`.
- Drizzle schema plus one generated migration, `drizzle/0000_init.sql`. Migrations apply automatically on open, and
  re-applying is a no-op.
- The connection is a singleton per file on `globalThis` (survives HMR) and is opened lazily on the first request.

### 5.4 Operations

- CLI scripts (`scripts/*.ts` via tsx): migrate, seed (guarded), remove demo, online backup, verified restore.
- Quality gates: `pnpm check` (typecheck, lint, format, tests), `pnpm test:e2e`, `pnpm build`.

## 6. The Data Spine

### 6.1 Irreversible decisions (made correctly in `0000_init`)

| Decision | Why it is hard to change later |
| --- | --- |
| Money as integer minor units, tax as basis points | changing representation means migrating every amount |
| CHECK constraints on status, payment method, amounts | SQLite cannot add CHECKs via `ALTER`; a `sales` rebuild with FKs on cascades and deletes `sale_items` |
| Line-item snapshots (name, category, price, cost) | history cannot be reconstructed after catalog edits |
| `ON DELETE RESTRICT` for products/customers/categories in use; cascade only `sale_items` from their sale | prevents silent loss of history |
| Text UUID ids, epoch-ms timestamps | stable across export/import; simple range math |

### 6.2 Metric semantics

Revenue = Σ `net_minor` of **completed** sales in `[from, to)`. Pending, cancelled and refunded sales are reported
separately and never subtracted. Profit is reported only when every completed sale in range has a cost. Full
definitions are in [`docs/spec/03-Domain-Rules.md`](docs/spec/03-Domain-Rules.md).

## 7. Integration Architecture (internal flows)

**Record a sale.** `saleFormSchema` (client) → `saleFormToInput` → `POST /api/sales` → `saleInputSchema` (server) →
one transaction (customer/product/category find-or-create, snapshots, `computeSaleTotals`, unique ref) → 201 → toast →
invalidate queries → close sheet.

**Dashboard.** `useDateRange()` → `GET /api/dashboard?from&to[&bucket]` → `dashboard()` runs about 10 aggregate
queries (current and previous range, zero-filled series, rankings, insights) → charts.

**Backup.** `POST /api/system/backup` or `pnpm db:backup` → SQLite online backup API → `backups/sales-<ISO>.db`.
**Restore** (app stopped) → `verifyBackupFile` (integrity_check + tables) → safety copy → replace → migrate.

## 8. Deployment Topology

A single Node 24 process (`pnpm start`, default port 3000) on the user's machine. The build (`pnpm build`) never
touches the DB. The DB path resolves from the project root (the `package.json` named `sales-dashboard`), so
starting from any working directory uses the same file. Override it with `SALES_DB_PATH`.

## 9. Security & Privacy Architecture

- Single-user and local. There is no auth by design; the server binds to localhost in dev. **Do not expose it on a
  network** without adding authentication.
- All input is validated server-side with Zod. SQL is parameterised (Drizzle and `sql` templates). LIKE patterns are
  escaped.
- CSV cells are escaped against formula injection.
- Destructive actions need explicit confirmation: erase-all requires the exact phrase `DELETE ALL DATA`, and restore
  verifies the file and keeps a safety copy.
- Real data never enters git (`.gitignore` excludes `/data/`, `*.db*`, `.env*`).

## 10. Build Evolution (phases with gates)

The build is staged so each layer is proven before the next depends on it. Details are in [`BUILD_PLAN.md`](BUILD_PLAN.md).

| Phase | Delivers | Gate |
| --- | --- | --- |
| 0 | toolchain | Node 24, pnpm 11 |
| 1 | config + dependencies | frozen install, native SQLite loads |
| 2 | domain library | unit tests |
| 3 | database | migrate on a temp DB |
| 4 | services + scripts | 61 tests; seed/backup/restore cycle |
| 5 | HTTP API | curl contract checks |
| 6 | shell + data layer | typecheck |
| 7 | pages | typecheck + lint + browser checks |
| 8 | e2e + README | `pnpm check`, all e2e pass, build |
| ⛔ | **Final acceptance** | traceability, prod smoke from another cwd, clean first run, metric spot check |

## 11. Critical Cross-Cutting Concerns

1. **Prerendering vs live data.** Cache Components prerenders client components too. Reading the current time or
   running queries outside `<Suspense>`, or hitting the DB in a GET handler without `connection()`, breaks the build
   or creates a DB at build time.
2. **Native module ABI.** better-sqlite3 must be built for Node 24, and pnpm 11 needs `allowBuilds`.
3. **Generated code drift.** shadcn, create-next-app and drizzle-kit output changes over time, so the stack pins exact
   versions and each generator runs once, after which its output is owned and committed.
4. **Correct paging with aggregates.** List rows use correlated subqueries, not JOIN + GROUP BY + LIMIT.
5. **Local-time bucketing.** SQL `localtime` and JS `dayKey()` must agree.

## 12. Technology Radar (not adopted, and why)

| Option | Status | Reason |
| --- | --- | --- |
| Server Actions / RSC data fetching | hold | one HTTP contract shared by UI, CSV and scripts; avoids Cache Components data rules |
| Prisma / Postgres | hold | heavier; needs a server; violates P1 |
| node:sqlite (built-in) | watch | could replace better-sqlite3 once stable with an online backup API |
| TypeScript 7 / ESLint 10 | watch | blocked by typescript-eslint and eslint-plugin-react peer ranges |
| Multi-currency FX, partial refunds, inventory | out of scope | product decision; documented limitations |

## Appendix A — Architecture Decision Record (summary)

| ADR | Decision | Status | What would change it |
| --- | --- | --- | --- |
| ADR-01 | Next.js 16 App Router full-stack, Node runtime | accepted | need for a separate API consumer → extract services |
| ADR-02 | SQLite + better-sqlite3 + Drizzle, WAL | accepted | multi-user/networked use → Postgres |
| ADR-03 | Route handlers + client fetching (TanStack Query) | accepted | Cache Components data APIs becoming simpler than one HTTP contract |
| ADR-04 | Integer minor units; 2-decimal currencies only | accepted | need for JPY/KWD → per-currency exponent |
| ADR-05 | Revenue = completed net excl. tax; refunds as full status | accepted | partial refunds required → refund ledger table |
| ADR-06 | Line-item snapshots + RESTRICT FKs | accepted | — (core invariant) |
| ADR-07 | Zustand only for UI prefs and sale-sheet state | accepted | — |
| ADR-08 | Demo data explicit, `is_demo`-tagged, removable | accepted | — |
| ADR-09 | DB path from project root; `SALES_DB_PATH` override | accepted | — |
| ADR-10 | Pin exact versions; run generators once and commit their output | accepted | — (build stability) |

## Appendix B — Capacity Reference

| Item | Size |
| --- | --- |
| App source (expected) | ~145 files (~7–8k lines app, ~4k lines shadcn/ui, ~1.6k lines tests) |
| `node_modules` | ~1 GB |
| `.next` build | ~0.5 GB |
| Demo DB | ~1,600 sales, ~1–2 MB |

## Appendix C — Verification Strategy

Confidence comes from layered, automated evidence gathered during the build: unit tests for pure domain rules,
integration tests on temporary SQLite files for every service, an HTTP contract gate with curl, Playwright e2e tests
for the main journeys, a production build that must not touch the DB, and manual checks of every page in light/dark
and at 390 px with a clean console. See [`docs/spec/07-Testing.md`](docs/spec/07-Testing.md) and [`BUILD_PLAN.md`](BUILD_PLAN.md).
