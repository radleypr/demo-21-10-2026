# Spec index

Ten binding specs. [`BUILD_PLAN.md`](../../BUILD_PLAN.md) tells you when to read each one; this page helps you find a fact fast.

## Files

| Spec | Covers | Read in phase |
| --- | --- | --- |
| [01-Product-Spec](01-Product-Spec.md) | Pages, flows, UI copy, FR-xx and NFR-xx requirements | 6, 7 |
| [02-Environment-and-Stack](02-Environment-and-Stack.md) | Toolchain, exact versions, bootstrap commands, config files, project layout | 0, 1 |
| [03-Domain-Rules](03-Domain-Rules.md) | Money, tax/discount math, date ranges, validation schemas, CSV, metric definitions | 2, 4 |
| [04-Data-Model](04-Data-Model.md) | Tables, constraints, indexes, DB path, migrations, connection, backup/restore | 3 |
| [05-Backend-and-API](05-Backend-and-API.md) | Error model, services, HTTP API contract (API-xx), CSV exports, demo seed, CLI scripts | 4, 5 |
| [06-Frontend-and-Design](06-Frontend-and-Design.md) | Design tokens, providers, data layer, Zustand stores, every component, responsive, a11y | 1, 6, 7 |
| [07-Testing](07-Testing.md) | Test config, helpers and every test case (T-U, T-I, T-E) with expected values | 2, 4, 8 |
| [08-Operations](08-Operations.md) | Commands, ports, env vars, DB lifecycle, backup/restore, troubleshooting | 1, 8 |
| [09-Gotchas](09-Gotchas.md) | Known pitfalls of this stack | before coding, and on any failure |
| [10-Requirements-Traceability](10-Requirements-Traceability.md) | Requirement → module → test or gate | Final acceptance |

## Where to find…

| Looking for | Go to |
| --- | --- |
| Exact package versions | 02 §2 |
| `package.json` scripts, tsconfig, next/eslint/prettier/vitest/playwright config | 02 §4 |
| Folder and file layout, layering rules | 02 §5 |
| Money parsing, rounding, discount allocation, worked examples | 03 §2 |
| Date presets, previous period, chart buckets | 03 §3 |
| Zod schemas (wire and form) | 03 §5 |
| Revenue / profit / status rules and insights | 03 §7 |
| Tables, columns, checks, delete rules | 04 §3 |
| DB path resolution and `SALES_DB_PATH` | 04 §5, 08 §3 |
| Error responses and `withDb` | 05 §3 |
| Service function contracts | 05 §5 |
| Every HTTP route, body and status code | 05 §6 |
| CSV columns and filenames | 05 §7 (format rules in 03 §6) |
| Demo seed catalog and algorithm | 05 §8 |
| CLI scripts (`db:*`) | 05 §9, 08 §6 |
| Colour tokens, fonts, motion | 06 §2 |
| Query keys and invalidation | 06 §5 |
| Sale sheet behaviour | 01 §5, 06 §11.1 |
| Sales table (server-side paging/sorting) | 01 §6, 06 §11.4 |
| Overview / Analytics charts | 01 §4 and §9, 06 §12 |
| Settings page (backups, demo data, erase) | 01 §10, 06 §13.3 |
| Accessibility requirements | 01 §12, 06 §15 |
| Test case list and pass criteria | 07 §4–§7 |
| Backup and restore procedure | 08 §7–§8 |
| An error you are seeing | 08 §10, then 09 |
