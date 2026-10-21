# Contributing

How to work from this specification. Read this before writing any code.

## Project overview

**Ledgerline** is a local-first, single-user sales dashboard: a Next.js 16 (App Router) full-stack app on SQLite via
Drizzle + better-sqlite3. One person records sales, products and customers on their own machine and gets KPIs,
charts, insights, analytics and CSV exports. There is no authentication, no cloud, no FX conversion and no inventory
tracking.

This public repository is the **starting guide**: it holds the specification and build plan. Create your own **private
working repository** from it (fork or import), then build the app there, at the repository root and alongside these
documents, by following [`BUILD_PLAN.md`](BUILD_PLAN.md). Keep the spec and the app in that one repository; do not
create a separate repository for the application code. The public upstream stays documentation-only.

Where [`BUILD_PLAN.md`](BUILD_PLAN.md) refers to the **app root**, read the root of your working repository (the
directory that contains this file). The spec files under `docs/spec/` are already there and stay in place; Phase 8
replaces only the root `README.md` with the app's user guide.

## Reading order

1. **[`BUILD_PLAN.md`](BUILD_PLAN.md)**: the ordered build plan with gates.
2. **[`docs/spec/07-Testing.md`](docs/spec/07-Testing.md)**: the test plan. Every listed case must exist and pass;
   the tests are the executable acceptance spec.
3. **`docs/spec/01–06, 08`**: the binding specification (product, stack, domain rules, data model, API, frontend,
   operations).
4. **[`docs/spec/09-Gotchas.md`](docs/spec/09-Gotchas.md)**: pitfalls of this stack. Read before coding.
5. **[`docs/spec/10-Requirements-Traceability.md`](docs/spec/10-Requirements-Traceability.md)**: requirement →
   module → proof. Use it as the final checklist.
6. [`ARCHITECTURE.md`](ARCHITECTURE.md) (architecture and decisions) and [`README.md`](README.md) (overview,
   prerequisites).

To look up a specific fact (a route, a table, a token, a test case), use [`docs/spec/README.md`](docs/spec/README.md).

**Conflict rule:** test plan (07) → domain rules (03) → data model (04) → backend/API (05) → frontend (06) → product
spec (01) → `ARCHITECTURE.md` → `README.md`. The more specific, lower-level document wins. Record any conflict you
resolve in the build report.

## Current state

- Greenfield. The specification is complete; the app is built at the root of your private working repository,
  beside the docs.
- The public upstream is documentation-only. Commit application code to your private working repository (on a
  working branch, if you prefer), never to the public starting guide.
- All architecture decisions are closed (see `ARCHITECTURE.md`, Appendix A). The work is implementation and
  verification against the specs, not product design.

## Technology direction

| Layer | Choice (exact versions in `docs/spec/02-Environment-and-Stack.md`) |
| --- | --- |
| Runtime | Node.js **24 LTS**, pnpm **11** (`packageManager: pnpm@11.22.0`) |
| Framework | Next.js **16.4.0** App Router, Turbopack, `cacheComponents: true`, Node runtime only |
| UI | React 19.3, Tailwind CSS 4 (CSS-first), shadcn/ui 4 (`radix-nova` preset), lucide-react, Recharts 3, Geist font |
| Client state | TanStack Query 5 (server state); Zustand 5 (UI preferences and sale-sheet state only) |
| Forms & validation | React Hook Form 7 + Zod 4; schemas shared by client and server |
| Tables | TanStack Table **9** (new `useTable` API), server-side paging/sorting/filtering |
| Data | SQLite (WAL) via **better-sqlite3 13** + **Drizzle ORM 0.45**; SQL migrations in `drizzle/` |
| Testing | Vitest 5 (unit + integration on temporary DBs), Playwright 1.64 (Chromium e2e) |
| Quality | TypeScript 6 strict, ESLint 9 (eslint-config-next), Prettier 3 |

## Design principles

- **The server is the source of truth.** Every rule is enforced in services and route handlers. Client validation is
  a convenience only.
- **Money is integer minor units** everywhere, formatted only at the edge with `Intl.NumberFormat`.
- **History is immutable.** Sale lines snapshot product name, category, price and cost. Nothing that would erase
  history cascades.
- **Status drives metrics.** Revenue counts completed sales only, net of discounts, excluding tax. A refund is a full
  status change, so refunds can never be double counted.
- **Zero config, local first.** The DB path resolves from the project root, and the DB is created and migrated on
  the first request, never at build time.
- **Demo data is explicit and tagged** (`is_demo`, `DEMO-` refs) and is never loaded silently.

## Standards

- CSV: UTF-8 with BOM, CRLF, RFC 4180 quoting, formula-injection escaping, amounts in major units.
- Dates: half-open `[from, to)` epoch-ms ranges, local-time day buckets, weeks starting Monday.
- Accessibility: WCAG 2.2 AA intent. Labelled controls, keyboard operable, one `main` and one `h1` per page, skip
  link, `prefers-reduced-motion` respected.

## Implementation rules

1. **Follow `BUILD_PLAN.md` in order and never skip a gate.** A gate passes only when the actual output matches the
   expected result. On failure, find and fix the root cause. Never weaken, skip or delete tests or gate checks.
2. **Read before writing.** Read every spec a phase lists under **Read** before writing that phase's code, and
   re-check the phase's requirements afterwards.
3. **The specs are binding.** Names of routes, tables, columns, scripts, environment variables, JSON fields, UI copy,
   error messages, limits and formulas must match exactly. Where a spec leaves something open, choose the simplest
   option that satisfies every requirement and test, and note it in the build report.
4. **Exact versions.** Use the versions in `docs/spec/02-Environment-and-Stack.md`. Run each scaffolding command
   (create-next-app, shadcn) once, as specified, then edit the result as the specs require.
5. **Node 24 everywhere.** If `node -v` is not v24, fix the toolchain before continuing. If the shell defaults to
   another major, put Node 24 first on `PATH` in every shell.
6. **Use the installed docs.** Next.js 16 with Cache Components differs from older Next.js (async params,
   `connection()`, prerendered client components). When unsure, read `node_modules/next/dist/docs/` in the app.
   The same applies to TanStack Table v9 (`node_modules/@tanstack/react-table/skills/`), Zod 4 and Tailwind 4.
7. **Tests with code.** Write the tests from `07-Testing.md` in the phase that introduces the code they cover, never
   after the gate.
8. **Protect real data.** Never write demo data into a database implicitly and never commit a database. Gates and
   tests use temporary databases only (`SALES_DB_PATH=.tmp/<name>/sales.db`), deleted afterwards. `data/` holds
   real data.
9. **No extras.** Add no features, abstractions or dependencies beyond the specs.
10. **Long output to logs.** Redirect test and build output to a file under `.tmp/` and read the summary lines.
11. **Clean up.** Stop every background server you start before finishing a phase.

## Build report

When the build is complete, run **Final acceptance** at the end of `BUILD_PLAN.md` and write a short build report:
what was built, every command run with its result, and every decision or deviation with the reason. Separate
**verified** (command run, output seen) from **not verified**.

## Key external dependencies

| Dependency | Needed for | Failure mode |
| --- | --- | --- |
| npm registry | `pnpm install`, `pnpm dlx` scaffolding | no network: install fails |
| better-sqlite3 prebuilt binaries | native SQLite driver | no matching prebuild: a C/C++ toolchain is needed (README §6) |
| Playwright Chromium download (~200 MB) | e2e tests only | run `pnpm test:e2e:install` once |
