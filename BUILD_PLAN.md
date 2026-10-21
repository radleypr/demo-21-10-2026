# Ledgerline — Build Plan & Gates

This is the build plan. It is ordered, and every phase ends with a **Gate**: a phase is done only when the
gate results match. Work from the **repository root** of your private working repository (the directory that
contains this file), with `SPEC` pointing to the same directory and Node 24 first on `PATH`.

## 1. Build Approach

| Topic | Rule |
| --- | --- |
| Method | Layered, test-first where it pays off. Each phase builds one layer and proves it before anything depends on it |
| Specs first | Before each phase, read every spec listed under **Read**. While coding, keep the phase's requirement IDs (FR/NFR) and test IDs open |
| Tests | Write the test cases a phase introduces (from [`07-Testing.md`](docs/spec/07-Testing.md)) in that phase, before or alongside the code |
| Failures | Find the root cause. Never weaken tests, gate checks or specs to get a pass. Check [`09-Gotchas.md`](docs/spec/09-Gotchas.md) and [`08-Operations.md`](docs/spec/08-Operations.md) → Troubleshooting first |
| Data safety | Gates use `SALES_DB_PATH=.tmp/<phase>/sales.db` and delete it afterwards. Never create or modify `data/` except in Final acceptance step F7 |
| Logs | Redirect long output, for example `pnpm test > .tmp/test.log 2>&1; tail -20 .tmp/test.log` |
| Servers | Start dev/prod servers in the background, wait until the port answers, and always stop them at the end of the phase |
| Commits | One commit per passed gate is a good rhythm |

Shell setup used by every phase:

```bash
export SPEC="$(pwd)"                             # the root of your working repository
export PATH="/path/to/node24/bin:$PATH"          # only if your default node is not v24
mkdir -p .tmp
```

## 2. Workstreams → Phases

| Workstream | Phases | Main specs |
| --- | --- | --- |
| W1 Toolchain & scaffold | 0, 1 | 02 |
| W2 Domain logic | 2 | 03 |
| W3 Persistence & services | 3, 4 | 04, 05 |
| W4 HTTP API | 5 | 05 |
| W5 Frontend | 6, 7 | 01, 06 |
| W6 Quality & docs | 8, Final | 07, 08, 10 |

---

## Phase 0 — Toolchain

**Read:** [`README.md`](README.md) §6, [`ARCHITECTURE.md`](ARCHITECTURE.md),
[`02-Environment-and-Stack.md`](docs/spec/02-Environment-and-Stack.md), [`09-Gotchas.md`](docs/spec/09-Gotchas.md).

- [ ] **P0.1** Check the prerequisites. `node -v` must be v24.x. If it is not, **stop and fix the toolchain**; do
  not continue on another Node major.
- [ ] **P0.2** Run `corepack enable` if `pnpm` is missing or not 11.x. Check that `git` is installed.
- [ ] **P0.3** Confirm the app has not been scaffolded yet: no `package.json`, `src/`, `scripts/`, `tests/` or
  `drizzle/` at the root. The specification files are already present and stay untouched; hidden tool folders such
  as `.git` are expected.

**GATE 0**
```bash
node -v        # v24.x
pnpm -v        # 11.x
git --version
```

## Phase 1 — Scaffold, dependencies and configuration

**Read:** [`02-Environment-and-Stack.md`](docs/spec/02-Environment-and-Stack.md) (all of it),
[`08-Operations.md`](docs/spec/08-Operations.md).

- [ ] **P1.1** Scaffold with `create-next-app` at the pinned version and flags from 02 (TypeScript, App Router,
  `src/` directory, Tailwind, ESLint, alias `@/*`). The repository root already contains this specification, so
  scaffold in a temporary directory and copy the generated files into the root without overwriting `README.md`,
  `CONTRIBUTING.md`, `ARCHITECTURE.md`, `BUILD_PLAN.md` or `docs/`.
- [ ] **P1.2** Set `package.json` exactly as specified: `name` **`sales-dashboard`** (the DB path resolution depends
  on it), `engines`, `packageManager` and every script in the scripts table.
- [ ] **P1.3** Add every dependency and devDependency at the **exact versions** in 02. Configure
  `pnpm-workspace.yaml` build approvals (better-sqlite3 and esbuild allowed, the others denied) so the native driver
  compiles.
- [ ] **P1.4** Run `shadcn init` with the specified preset and flags, then `shadcn add` the exact component list
  from 02. Apply the one required edit to the generated `sidebar.tsx` (see
  [`06-Frontend-and-Design.md`](docs/spec/06-Frontend-and-Design.md)).
- [ ] **P1.5** Write or adjust every config file to the values in 02: `tsconfig.json`, `next.config.ts`,
  `eslint.config.mjs`, `.prettierrc.json`, `.prettierignore`, `.gitignore`, `.env.example`, `.nvmrc`,
  `components.json`, `vitest.config.mts`. (`playwright.config.ts` comes in Phase 8.)
- [ ] **P1.6** Run `git init` if the directory is not a repo yet, then `pnpm install`.

**GATE 1**
```bash
pnpm install                                   # exit 0, no unapproved-build warnings for better-sqlite3
node -e "const D=require('better-sqlite3');console.log(new D(':memory:').prepare('select sqlite_version() v').get().v)"   # 3.5x.x
node -e "const p=require('./package.json');console.log(p.name,p.engines.node,p.packageManager)"   # sales-dashboard >=24 <25 pnpm@11.22.0
pnpm typecheck && pnpm lint                    # both exit 0
git check-ignore -q data/sales.db && echo ignored   # ignored
```

## Phase 2 — Domain library

**Read:** [`03-Domain-Rules.md`](docs/spec/03-Domain-Rules.md), the unit section of
[`07-Testing.md`](docs/spec/07-Testing.md).

- [ ] **P2.1** Implement `src/lib/constants.ts`, `money.ts`, `date-range.ts`, `validation.ts` and `csv.ts` with
  exactly the exported names, types, semantics and messages in 03. (`src/lib/utils.ts` exists from shadcn.)
- [ ] **P2.2** Write every **unit** test case from 07 in `tests/unit/`.

**GATE 2**
```bash
pnpm exec vitest run tests/unit      # 4 test files, every unit case from 07 passes (31 tests)
pnpm typecheck
```

## Phase 3 — Database

**Read:** [`04-Data-Model.md`](docs/spec/04-Data-Model.md).

- [ ] **P3.1** Implement `src/db/schema.ts` (every table, column, default, CHECK, FK action and index in 04),
  `src/db/paths.ts` and `src/db/client.ts`, plus `src/server/db.ts` and `src/server/errors.ts`.
- [ ] **P3.2** Add `drizzle.config.ts` and generate the initial migration **once**:
  `pnpm exec drizzle-kit generate --name init` → `drizzle/0000_init.sql` + `drizzle/meta/`. Read the SQL and check it
  against 04 (CHECKs, FK actions, indexes). If it is wrong, fix the schema, delete `drizzle/` and generate again.
  Never edit the generated SQL by hand.
- [ ] **P3.3** Implement `scripts/db-migrate.ts`.

**GATE 3**
```bash
SALES_DB_PATH=.tmp/p3/sales.db pnpm db:migrate     # prints the resolved absolute path
node -e "const D=require('better-sqlite3');const d=new D('.tmp/p3/sales.db');console.log(d.prepare(\"select name from sqlite_master where type='table' and name not like '%sqlite%' and name not like '__drizzle%' order by name\").all().map(r=>r.name).join(','));console.log(d.pragma('journal_mode',{simple:true}))"
# categories,customers,products,sale_items,sales,settings
# wal
SALES_DB_PATH=.tmp/p3/sales.db pnpm db:migrate     # second run: succeeds, no changes
rm -rf .tmp/p3
ls data 2>/dev/null && echo "UNEXPECTED data/"     # prints nothing
```

## Phase 4 — Services, demo seed and scripts

**Read:** [`05-Backend-and-API.md`](docs/spec/05-Backend-and-API.md) (services, demo seed, scripts),
[`03-Domain-Rules.md`](docs/spec/03-Domain-Rules.md) (metrics), the integration section of
[`07-Testing.md`](docs/spec/07-Testing.md).

- [ ] **P4.1** Write `tests/helpers.ts` (temporary DB per test) and the **integration** test cases from 07 for each
  service as you build it.
- [ ] **P4.2** Implement `src/server/services/`: `sales.ts`, `catalog.ts`, `customers.ts`, `settings.ts`,
  `analytics.ts`, `exports.ts`, `maintenance.ts` with the function signatures, transactions, errors and messages
  in 05. Services must not import `server-only`.
- [ ] **P4.3** Implement `src/server/demo-seed.ts` and the remaining scripts: `db-seed.ts`, `db-demo-remove.ts`,
  `db-backup.ts`, `db-restore.ts`.

**GATE 4**
```bash
pnpm test                                                      # 9 test files, 61 tests, all passed
export SALES_DB_PATH=.tmp/p4/sales.db
pnpm db:seed                                                   # seeds ~1,600 demo sales, 16 products, 20 customers
pnpm db:seed; echo "exit=$?"                                   # refuses (DB not empty): exit=1
pnpm db:backup                                                 # prints .tmp/p4/backups/sales-<ISO>.db
pnpm db:restore "$(ls .tmp/p4/backups/sales-*.db | head -1)"   # verified restore + pre-restore safety copy
pnpm db:demo:remove                                            # reports removed demo rows
unset SALES_DB_PATH; rm -rf .tmp/p4
```

## Phase 5 — HTTP API

**Read:** [`05-Backend-and-API.md`](docs/spec/05-Backend-and-API.md) (error model and API contract),
[`09-Gotchas.md`](docs/spec/09-Gotchas.md) (Cache Components items).

- [ ] **P5.1** Implement `src/server/api.ts` (`withDb`, `errorResponse`, `readJson`) and every route handler under
  `src/app/api/` in the contract (18 route files), with the exact statuses, JSON shapes and CSV filenames.

**GATE 5**: start `SALES_DB_PATH=.tmp/p5/sales.db pnpm exec next dev -p 3100 > .tmp/p5.log 2>&1 &`, wait until
`curl -s localhost:3100/api/settings` answers, then:
```bash
curl -s localhost:3100/api/settings                                     # {"id":1,"currency":"USD","locale":"en-US",...}
curl -s -XPOST localhost:3100/api/system/demo -H 'content-type: application/json' -d '{"action":"seed"}'   # {"sales":~1600,"products":16,"customers":20}
curl -s "localhost:3100/api/sales?pageSize=2"                           # {"rows":[2 rows],"total":>1000}
curl -s -o /dev/null -w '%{http_code}\n' -XPOST localhost:3100/api/sales -H 'content-type: application/json' -d '{}'   # 400
curl -s -o /dev/null -w '%{http_code}\n' localhost:3100/api/sales/does-not-exist                                         # 404
curl -s localhost:3100/api/sales/export | head -c 60 | od -c | head -2  # starts with 357 273 277 (BOM), then the CSV header
curl -s "localhost:3100/api/dashboard?from=0&to=$(node -e 'console.log(Date.now())')" | head -c 200   # {"range":...,"kpis":...}
```
Then call each endpoint in the contract at least once and compare the response with 05. Stop the server (kill
whatever listens on port 3100) and run `rm -rf .tmp/p5`.

## Phase 6 — Design system, shell and client data layer

**Read:** [`06-Frontend-and-Design.md`](docs/spec/06-Frontend-and-Design.md),
[`01-Product-Spec.md`](docs/spec/01-Product-Spec.md) (global layout, sale sheet).

- [ ] **P6.1** Replace `src/app/globals.css` tokens with the light/dark palette, chart colours, radius and fonts from
  06. Replace `layout.tsx` (Geist via the `geist` package, metadata, providers, skip link).
- [ ] **P6.2** Implement the client data layer: `src/lib/api-client.ts`, `queries.ts`, `ui-store.ts`, `nav.ts`, and
  `src/components/providers.tsx`.
- [ ] **P6.3** Implement `src/components/shell/` (app shell, sidebar, top bar, date-range picker, command menu, theme
  toggle) and `src/components/common/` (confirm, multi-select, page skeleton, states, status badge).
- [ ] **P6.4** Implement `src/components/sales/` (sale dialog, sales table, sale actions). The shell mounts the
  global sale sheet.
- [ ] **P6.5** Replace the scaffold home page with a minimal placeholder so the shell renders.

**GATE 6**
```bash
pnpm typecheck && pnpm lint          # both exit 0
```
Start `SALES_DB_PATH=.tmp/p6/sales.db pnpm dev` and check: the sidebar shows the six sections, the theme toggle
works, **Add sale** opens the sheet, a valid sale saves with a success toast, and the console has no errors. Stop the
server and run `rm -rf .tmp/p6`.

## Phase 7 — Pages

**Read:** [`01-Product-Spec.md`](docs/spec/01-Product-Spec.md) (every page),
[`06-Frontend-and-Design.md`](docs/spec/06-Frontend-and-Design.md) (feature views, charts).

- [ ] **P7.1** Implement `src/components/dashboard/` (KPI card, charts, overview, analytics view),
  `src/components/catalog/`, `src/components/customers/` and `src/components/settings/`.
- [ ] **P7.2** Implement every `page.tsx`: `/`, `/sales`, `/products`, `/customers`, `/analytics`, `/settings`
  (server component with `metadata` and a `Suspense` boundary around the client view).

**GATE 7**
```bash
pnpm typecheck && pnpm lint && pnpm test     # all exit 0
```
Then start `SALES_DB_PATH=.tmp/p7/sales.db pnpm dev` and check in a browser (or with Playwright / DevTools):
- [ ] the empty Overview shows the empty state from 01 (**"No sales in this period"**);
- [ ] Settings → **Load demo data** fills every page with charts and tables;
- [ ] **Add sale** records a sale (success toast) and it appears in Sales, Overview and Products;
- [ ] Sales: search, every filter, sort, paging, bulk delete and CSV export work;
- [ ] Products, Customers and Analytics behave as in 01, including CSV exports;
- [ ] the theme toggle switches light/dark and both look correct;
- [ ] at 390 px width the sidebar becomes a drawer and nothing overflows horizontally;
- [ ] the browser console has no errors or hydration warnings.

Stop the server and run `rm -rf .tmp/p7`.

## Phase 8 — E2E, README and polish

**Read:** [`07-Testing.md`](docs/spec/07-Testing.md) (e2e section), [`08-Operations.md`](docs/spec/08-Operations.md).

- [ ] **P8.1** Add `playwright.config.ts` with the values from 02/07 and write every **e2e** case from 07 in
  `tests/e2e/`.
- [ ] **P8.2** Run `pnpm test:e2e:install` (downloads Chromium once).
- [ ] **P8.3** Write the app `README.md` covering prerequisites, Node version, install, start, production build, DB
  location and override, migrations, demo data, backup/restore, tests, troubleshooting and native dependencies
  (content in 08).
- [ ] **P8.4** Run `pnpm format`.

**GATE 8**
```bash
pnpm check > .tmp/check.log 2>&1; echo "exit=$?"    # exit=0 (typecheck + lint + format:check + 61 tests)
pnpm test:e2e > .tmp/e2e.log 2>&1; echo "exit=$?"   # exit=0; 5 passed
pnpm build > .tmp/build.log 2>&1; echo "exit=$?"    # exit=0
ls data 2>/dev/null && echo "UNEXPECTED data/"      # prints nothing: the build never opens the DB
```

---

## ⛔ Final acceptance

Run every item and report the result of each with the command and the observed output.

- [ ] **F1 Traceability**: go through [`10-Requirements-Traceability.md`](docs/spec/10-Requirements-Traceability.md)
  row by row. Every FR/NFR is implemented in the named module and proved by the named test or gate. List any gaps.
- [ ] **F2 Hygiene**: `git status --porcelain` shows no `data/`, `*.db`, `.tmp/`, `test-results/` or `.next/`
  entries (all ignored).
- [ ] **F3 Quality**: `pnpm check` exit 0, **61** tests passed.
- [ ] **F4 E2E**: `pnpm test:e2e` → **5** passed.
- [ ] **F5 Build**: `pnpm build` exit 0. Every page route is static (○) and every `/api/*` route is dynamic (ƒ).
- [ ] **F6 Production smoke from another working directory**:
  `cd src && SALES_DB_PATH=.tmp/smoke/sales.db PORT=3300 pnpm start` (in the background), then
  `curl -s localhost:3300/api/system` shows `dbPath` = `<repository root>/.tmp/smoke/sales.db` (resolved from the project
  root, **not** `src/.tmp`), and `curl -s -o /dev/null -w '%{http_code}' localhost:3300/` → `200`. Stop the server
  and run `rm -rf .tmp/smoke`.
- [ ] **F7 Clean first run**: `rm -rf data`, run `pnpm dev`, and open `/`. `data/sales.db` is created and migrated,
  the empty state shows, and there is no demo data until it is loaded from Settings.
- [ ] **F8 Metric spot check** ([`03-Domain-Rules.md`](docs/spec/03-Domain-Rules.md)), on that fresh DB:
  - record a **completed** sale of 3 × 12.50 with 10 % tax → Total Revenue rises by **$37.50** (tax excluded; the
    sale total is $41.25);
  - edit it to **Refunded** → revenue drops back and the refunded amount appears in the KPI note;
  - no negative amounts anywhere.
  Afterwards stop the server and delete the test data (Settings → Erase all, or `rm -rf data`) unless you want
  to keep it.
- [ ] **F9 Report**: what was built, dependency versions (`pnpm list --depth 0`), DB location, every command with its
  result, every decision the specs left open, and every deviation with the reason. Separate verified from not
  verified.

---

## Appendix A — Dependency Register

| Phase | Needs | Provided by |
| --- | --- | --- |
| 1 | npm registry; native build approval | network; `pnpm-workspace.yaml` |
| 2 | `zod`, `date-fns`, `papaparse` | Phase 1 |
| 3 | constants (CHECK values) | Phase 2 |
| 4 | DB client and migration; temporary DBs in tests | Phase 3 |
| 5 | services; `connection()` from `next/server` | Phase 4 |
| 6 | API contract (client hooks call it); validation form schemas | Phases 2, 5 |
| 7 | shell, query hooks, UI primitives, sale sheet | Phase 6 |
| 8 | all pages; Playwright Chromium | Phase 7; `pnpm test:e2e:install` |

## Appendix B — Critical Path

`P0 → P1 → P2 → P3 → P4 (61 tests) → P5 → P6 → P7 → P8 (check, e2e, build) → Final`. The path is linear because each
layer compiles only against earlier ones. Within a phase, independent files can be written in any order.

## Appendix C — Risk Register

| Risk | Symptom | Mitigation |
| --- | --- | --- |
| Wrong Node major | `NODE_MODULE_VERSION` / `bindings` error; engines warning | use Node 24; `pnpm rebuild better-sqlite3` |
| Build scripts blocked | better-sqlite3 has no `.node` binary | set `allowBuilds` in `pnpm-workspace.yaml`; reinstall |
| No prebuilt binary | node-gyp compile errors | install a C/C++ toolchain (README §6.1) |
| Version drift | APIs differ from the specs (Next, TanStack Table, Zod, Tailwind) | install the exact versions in 02; read the package docs in `node_modules` |
| Prerender errors | build fails with current-time or uncached-data errors | follow the Cache Components rules in 09 (Suspense, `connection()`, no time reads during render) |
| Port in use | dev/e2e servers fail to start | free 3100/3200/3300 or stop old servers |
| Spec drift | behaviour differs from 01/05 | re-read the spec section; the test plan and lower-level specs win |
| Real data touched | `data/` created during gates | always set `SALES_DB_PATH=.tmp/...` |

## Appendix D — Change Control

The specs are the contract. If you find a genuine contradiction or gap, resolve it with the conflict rule in
[`CONTRIBUTING.md`](CONTRIBUTING.md), implement the simplest consistent behaviour, and record it in the build report. Do not
change gates, test expectations or requirement IDs.
