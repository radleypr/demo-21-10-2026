# 02 — Environment and Stack

This document fixes the toolchain, the exact package versions, the bootstrap procedure for an empty directory, the
required value of every configuration file and the required project layout of Ledgerline. Everything here is
normative: when a value is listed, use exactly that value. Commands, data locations and troubleshooting are in
[08-Operations.md](08-Operations.md); library pitfalls are in [09-Gotchas.md](09-Gotchas.md).

## 1. Toolchain

| ID | Item | Required value | Rationale |
| --- | --- | --- | --- |
| ENV-01 | Node.js | **24 LTS** (`engines.node` = `>=24 <25`, `.nvmrc` = `24`) | Current LTS. `better-sqlite3` 13 ships prebuilt binaries for Node 24. Another major (25, 26) loads a mismatched native binding and fails. |
| ENV-02 | pnpm | **11.22.0**, pinned through `packageManager: "pnpm@11.22.0"` | Run `corepack enable` once; Corepack then supplies the pinned version. |
| ENV-03 | OS | macOS, Linux or Windows | All supported. |
| ENV-04 | Native build tools | Only when no `better-sqlite3` prebuilt binary matches: Xcode Command Line Tools (macOS), `build-essential` + Python 3 (Linux), Visual Studio Build Tools (Windows) | `better-sqlite3` then compiles from source. |
| ENV-05 | Git | Any recent version | The project is a Git repository; `.gitignore` keeps data out of it. |

Every command (`pnpm`, `pnpm exec`, `node`, `tsx`, the Playwright web server) must run on Node 24. When the
machine's default `node` is another major, put Node 24 first on `PATH` for the whole shell session (for example
`nvm use`, or `export PATH=/opt/homebrew/opt/node@24/bin:$PATH` with Homebrew).

### 1.1 Toolchain check

```bash
node -v      # v24.x
pnpm -v      # 11.22.0
node -e "const D=require('better-sqlite3');console.log(new D(':memory:').prepare('select sqlite_version() v').get().v)"
```

The last command, run inside the project after install, must print a SQLite version (3.5x.x), not a bindings error.

## 2. Dependencies

The "Specifier" column is the literal text required in `package.json`. Caret ranges are intentional where shown;
everything else is an exact pin. For dev dependencies with a range, "Resolves to" is the version the lockfile must end
up with.

### 2.1 Runtime dependencies

| Package | Specifier | Purpose |
| --- | --- | --- |
| `next` | `16.4.0` | App Router, route handlers, Turbopack, Cache Components |
| `react` | `19.3.0` | UI runtime |
| `react-dom` | `19.3.0` | DOM renderer |
| `better-sqlite3` | `13.0.3` | Synchronous embedded SQLite driver (native module) |
| `drizzle-orm` | `0.45.4` | Typed schema, query builder and SQL-file migrator |
| `zod` | `4.6.7` | Validation schemas shared by client and server |
| `react-hook-form` | `7.90.0` | Form state |
| `@hookform/resolvers` | `5.9.1` | Zod resolver for React Hook Form |
| `@tanstack/react-query` | `5.104.2` | Client server-state cache and invalidation |
| `@tanstack/react-table` | `9.3.1` | Headless table, **v9 API** (`useTable`) |
| `zustand` | `5.0.15` | UI-only preferences (persisted) and sale-dialog state |
| `recharts` | `3.10.1` | Charts, used through the shadcn `chart` component |
| `date-fns` | `4.4.0` | Date arithmetic and formatting |
| `papaparse` | `5.7.0` | CSV generation and parsing |
| `sonner` | `2.0.8` | Toasts |
| `next-themes` | `0.4.6` | Light / dark / system theme |
| `lucide-react` | `1.57.0` | Icons |
| `geist` | `1.7.2` | Geist Sans and Geist Mono served locally (no network at build time) |
| `radix-ui` | `^1.8.1` | Primitives behind shadcn/ui |
| `cmdk` | `^1.1.1` | Command palette primitive (shadcn `command`) |
| `react-day-picker` | `^10.0.2` | Calendar primitive (shadcn `calendar`) |
| `class-variance-authority` | `^0.7.1` | Variant styling in shadcn components |
| `cn` | `^0.4.0` | Class-name merge helper; `src/lib/utils.ts` re-exports `cn` from it |
| `tw-animate-css` | `^1.4.0` | Animation utilities imported by `globals.css` |
| `server-only` | `^0.0.1` | Build-time guard for server-only modules |

### 2.2 Dev dependencies

| Package | Specifier | Resolves to | Purpose |
| --- | --- | --- | --- |
| `typescript` | `^6.0.3` | 6.0.3 | Type checker. **Stay on 6.0.x, not 7**: `typescript-eslint` (pulled in by `eslint-config-next` 16.4.0) has a peer range of `typescript <6.1`. TypeScript 7 breaks `pnpm lint`. |
| `eslint` | `^9` | 9.39.x | Linter. **Stay on 9, not 10**: `eslint-plugin-react` (pulled in by `eslint-config-next`) does not accept ESLint 10 as a peer. |
| `eslint-config-next` | `16.4.0` | 16.4.0 | Next.js lint presets (`core-web-vitals`, `typescript`) |
| `prettier` | `^3.9.10` | 3.9.10 | Formatter |
| `tailwindcss` | `^4` | 4.3.x | Tailwind CSS 4 (CSS-first configuration) |
| `@tailwindcss/turbopack` | `^4` | 4.3.x | Tailwind loader for the Turbopack CSS rule (there is no PostCSS) |
| `shadcn` | `4.21.4` | 4.21.4 | shadcn CLI; also needed by the build because `globals.css` imports `shadcn/tailwind.css` |
| `drizzle-kit` | `0.31.11` | 0.31.11 | Generates SQL migrations from the schema |
| `tsx` | `^4.23.15` | 4.23.15 | Runs the TypeScript CLI scripts in `scripts/` |
| `vitest` | `5.0.4` | 5.0.4 | Unit and integration tests |
| `@playwright/test` | `1.64.0` | 1.64.0 | End-to-end tests (Chromium) |
| `@types/node` | `^24.19.2` | 24.19.x | Node 24 typings |
| `@types/react` | `^19` | 19.3.x | React typings |
| `@types/react-dom` | `^19` | 19.3.x | React DOM typings |
| `@types/better-sqlite3` | `^9.6.0` | 9.6.x | Driver typings |
| `@types/papaparse` | `^5.5.2` | 5.5.x | CSV library typings |

Do not add any other dependency. In particular: no PostCSS config or `@tailwindcss/postcss`, no
`next/font/google`, no second ORM, no test framework other than Vitest and Playwright, no jsdom.

## 3. Bootstrap procedure

Run from the empty project directory, on Node 24. Steps 1 to 10 produce the skeleton; section 4 then states the
required final value of every configuration file.

1. `corepack enable`, then confirm `pnpm -v` prints 11.x.
2. Scaffold Next.js 16.4.0 into the current directory:
   `pnpm create next-app@16.4.0 . --ts --tailwind --eslint --app --src-dir --import-alias "@/*" --use-pnpm --cache-components --disable-git --yes`.
   This yields TypeScript, an ESLint flat config, Tailwind 4 wired through `@tailwindcss/turbopack` (no PostCSS
   file), the `src/` directory, the App Router, Turbopack (the default bundler), the `@/*` alias, an `AGENTS.md`,
   a `pnpm-workspace.yaml` and a `next.config.ts` that already has `cacheComponents` and `partialPrefetching`.
   Do not pass `--react-compiler`. If the CLI rejects the directory name as a package name, scaffold into a
   temporary sub-directory and move its contents (including dotfiles) up.
3. In `package.json` set `"name": "sales-dashboard"` (mandatory; database path resolution depends on it, see
   [08-Operations.md](08-Operations.md) section 3), change `lint` to `eslint .`, and add `engines`.
4. Extend `pnpm-workspace.yaml` to the content in section 4.11 **before** installing native packages, so the
   `better-sqlite3` build script is allowed.
5. Add runtime dependencies at exact versions:
   `pnpm add next@16.4.0 react@19.3.0 react-dom@19.3.0 better-sqlite3@13.0.3 drizzle-orm@0.45.4 zod@4.6.7 react-hook-form@7.90.0 @hookform/resolvers@5.9.1 @tanstack/react-query@5.104.2 @tanstack/react-table@9.3.1 zustand@5.0.15 recharts@3.10.1 date-fns@4.4.0 papaparse@5.7.0 sonner@2.0.8 next-themes@0.4.6 lucide-react@1.57.0 geist@1.7.2 server-only@^0.0.1`
6. Add dev dependencies (this also upgrades the scaffold's `typescript ^5` and `@types/node ^20`):
   `pnpm add -D typescript@^6.0.3 eslint@^9 eslint-config-next@16.4.0 prettier@^3.9.10 tailwindcss@^4 @tailwindcss/turbopack@^4 shadcn@4.21.4 drizzle-kit@0.31.11 tsx@^4.23.15 vitest@5.0.4 @playwright/test@1.64.0 @types/node@^24.19.2 @types/react@^19 @types/react-dom@^19 @types/better-sqlite3@^9.6.0 @types/papaparse@^5.5.2`
7. Initialise shadcn/ui non-interactively:
   `CI=1 pnpm dlx shadcn@4.21.4 init -b radix -p nova -y --no-monorepo --no-rtl`.
   Without `CI=1` the CLI prompts and blocks a non-interactive run. It writes `components.json` (section 4.9) and
   `src/lib/utils.ts`, and rewrites `src/app/globals.css`.
8. Add the shadcn components in one command:
   `pnpm dlx shadcn@4.21.4 add alert-dialog avatar badge button calendar card chart checkbox command dialog dropdown-menu empty field input-group input kbd label popover select separator sheet sidebar skeleton sonner table tabs textarea toggle toggle-group tooltip`.
   This also creates `src/hooks/use-mobile.ts`.
9. Reconcile `package.json` with section 2: every specifier must match exactly, `shadcn` must be in
   `devDependencies`, and nothing else may be present. Run `pnpm install`.
10. Apply sections 4.1 to 4.14, create the layout in section 5, then `git init`.
11. Verify: the toolchain check in section 1.1, `pnpm typecheck`, and `git status` shows no `data/`, `.tmp/` or
    `*.db` entries.

Required edits to generated files after step 8 (the only edits allowed inside `src/components/ui/` and
`src/hooks/`):

| File | Required change | Reason |
| --- | --- | --- |
| `src/components/ui/sidebar.tsx` | `SidebarInset` renders a `div` instead of `main` (props typed as `div` props), preceded by a comment starting `ponytail:` that names the reason | The app shell renders its own `<main id="main">`; nested `main` landmarks are invalid |
| `src/app/globals.css` | `--font-sans` and `--font-heading` map to `var(--font-geist-sans), ui-sans-serif, system-ui, sans-serif`; `--font-mono` maps to `var(--font-geist-mono), ui-monospace, monospace` | The generated file refers `--font-sans` back to itself; fonts come from the `geist` package |
| `src/hooks/use-mobile.ts` | `useIsMobile()` returns `useSyncExternalStore` over `matchMedia("(max-width: 767px)")` (subscribing to its `change` event) with a server snapshot of `false` | A version that calls `setState` inside an effect fails the `react-hooks/set-state-in-effect` lint rule |

The scaffold's `src/app/page.tsx`, `src/app/layout.tsx` and `README.md` are replaced by the application versions.
Design tokens and the rest of `globals.css` are specified in [06-Frontend-and-Design.md](06-Frontend-and-Design.md).

## 4. Configuration files

### 4.1 `package.json`

| Key | Required value |
| --- | --- |
| `name` | `sales-dashboard` (never change it) |
| `version` | `0.1.0` |
| `private` | `true` |
| `packageManager` | `pnpm@11.22.0` |
| `engines.node` | `>=24 <25` |
| `dependencies` / `devDependencies` | Exactly section 2 |

Scripts (all required, exact commands):

| Script | Command | Purpose |
| --- | --- | --- |
| `dev` | `next dev` | Development server on port 3000 |
| `build` | `next build` | Production build; must not create or touch the database |
| `start` | `next start` | Serve the production build on port 3000 (`PORT` overrides) |
| `lint` | `eslint .` | Lint the whole project |
| `format` | `prettier --write .` | Format in place |
| `format:check` | `prettier --check .` | Formatting gate |
| `typecheck` | `next typegen && tsc --noEmit` | Generates the global route types (`RouteContext`, `LayoutProps`) before type checking |
| `test` | `vitest run` | Unit and integration tests, single run |
| `test:watch` | `vitest` | Watch mode |
| `test:e2e` | `playwright test` | End-to-end tests |
| `test:e2e:install` | `playwright install chromium` | One-time browser download |
| `db:generate` | `drizzle-kit generate` | New migration after a schema change |
| `db:migrate` | `tsx scripts/db-migrate.ts` | Create and migrate the database explicitly |
| `db:seed` | `tsx scripts/db-seed.ts` | Load demo data (`--force` to seed a non-empty database) |
| `db:demo:remove` | `tsx scripts/db-demo-remove.ts` | Delete demo rows only |
| `db:backup` | `tsx scripts/db-backup.ts` | Online backup; optional target directory argument |
| `db:restore` | `tsx scripts/db-restore.ts` | Verified restore of a backup file |
| `check` | `pnpm typecheck && pnpm lint && pnpm format:check && pnpm test` | Full local quality gate (e2e excluded) |

### 4.2 `.nvmrc`

Single line: `24`.

### 4.3 `tsconfig.json`

| Key | Value |
| --- | --- |
| `compilerOptions.target` | `ES2022` (the scaffold writes `ES2017`; change it) |
| `compilerOptions.lib` | `["dom", "dom.iterable", "esnext"]` |
| `compilerOptions.allowJs` | `true` |
| `compilerOptions.skipLibCheck` | `true` |
| `compilerOptions.strict` | `true` |
| `compilerOptions.noEmit` | `true` |
| `compilerOptions.esModuleInterop` | `true` |
| `compilerOptions.module` | `esnext` |
| `compilerOptions.moduleResolution` | `bundler` |
| `compilerOptions.resolveJsonModule` | `true` |
| `compilerOptions.isolatedModules` | `true` |
| `compilerOptions.jsx` | `react-jsx` |
| `compilerOptions.incremental` | `true` |
| `compilerOptions.plugins` | `[{ "name": "next" }]` |
| `compilerOptions.paths` | `{ "@/*": ["./src/*"] }` |
| `include` | `["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts", ".next/dev/types/**/*.ts", "**/*.mts"]` |
| `exclude` | `["node_modules"]` |

The project is type-checked as one unit: `next build` and `pnpm typecheck` both cover `src/`, `scripts/` and
`tests/`. `tsx` honours the `@/*` alias, so scripts import from `@/db/...` and `@/server/...`.

### 4.4 `next.config.ts`

Typed as `NextConfig`, default export.

| Key | Value | Effect |
| --- | --- | --- |
| `cacheComponents` | `true` | Cache Components: pages are prerendered; dynamic work must sit inside `<Suspense>` |
| `partialPrefetching` | `true` | Prefetch only the static shell of routes |
| `turbopack.rules["*.css"].loaders` | `["@tailwindcss/turbopack"]` | Tailwind compiles through Turbopack |
| `turbopack.rules["*.css"].as` | `"*.css"` | Output remains CSS |

No other keys: no `output`, no `images`, no webpack config. The edge runtime is not used anywhere.

### 4.5 `eslint.config.mjs`

Flat config; default export of `defineConfig([...])` from `eslint/config`, containing in order:

1. Every entry of `eslint-config-next/core-web-vitals` (spread).
2. Every entry of `eslint-config-next/typescript` (spread).
3. `globalIgnores([".next/**", "out/**", "build/**", "next-env.d.ts"])`, also from `eslint/config`.

No custom rules or overrides. `pnpm lint` must exit 0.

### 4.6 Prettier

`.prettierrc.json`: `{ "printWidth": 140 }` and nothing else (Prettier defaults otherwise: double quotes,
semicolons, trailing commas `all`, 2-space indent).

`.prettierignore` entries, one per line:

| Entry | Reason |
| --- | --- |
| `pnpm-lock.yaml` | Generated |
| `drizzle` | Generated SQL migrations |
| `drizzle/meta` | Generated migration metadata |
| `.next` | Build output |
| `data` | Local database and backups |
| `src/components/ui` | Generated shadcn components keep their generated formatting |
| `.tmp` | Throwaway databases and logs |
| `test-results` | Playwright artefacts |
| `playwright-report` | Playwright artefacts |
| `next-env.d.ts` | Generated |

### 4.7 `.gitignore`

Keep every scaffold entry (`/node_modules`, `/.pnp`, `.pnp.*`, `.yarn/*` with the `!.yarn/patches`,
`!.yarn/plugins`, `!.yarn/releases`, `!.yarn/versions` exceptions, `/coverage`, `/.next/`, `/out/`, `/build`,
`.DS_Store`, `*.pem`, `npm-debug.log*`, `yarn-debug.log*`, `yarn-error.log*`, `.pnpm-debug.log*`, `.env*`,
`.vercel`, `*.tsbuildinfo`, `next-env.d.ts`) and append:

| Entry | Reason |
| --- | --- |
| `/data/` | Real sales data and backups are never committed |
| `*.db`, `*.db-wal`, `*.db-shm` | Any SQLite file anywhere |
| `/test-results/`, `/playwright-report/` | Playwright artefacts |
| `/.tmp/` | Throwaway test and gate databases |
| `!.env.example` | Last line; re-includes the example file excluded by `.env*` |

### 4.8 `.env.example`

Two comment lines; no variable is set:

```text
# Optional. Defaults to data/sales.db under the project root. Absolute, or relative to the project root.
# SALES_DB_PATH=data/sales.db
```

### 4.9 `components.json`

| Key | Value |
| --- | --- |
| `$schema` | `https://ui.shadcn.com/schema.json` |
| `style` | `radix-nova` |
| `rsc` | `true` |
| `tsx` | `true` |
| `tailwind.config` | `""` |
| `tailwind.css` | `src/app/globals.css` |
| `tailwind.baseColor` | `neutral` |
| `tailwind.cssVariables` | `true` |
| `tailwind.prefix` | `""` |
| `iconLibrary` | `lucide` |
| `rtl` | `false` |
| `aliases.components` | `@/components` |
| `aliases.utils` | `@/lib/utils` |
| `aliases.ui` | `@/components/ui` |
| `aliases.lib` | `@/lib` |
| `aliases.hooks` | `@/hooks` |
| `menuColor` | `default` |
| `menuAccent` | `subtle` |
| `registries` | `{}` |

`src/lib/utils.ts` contains a single re-export of `cn` from the `cn` package.

### 4.10 `drizzle.config.ts`

`defineConfig` from `drizzle-kit`:

| Key | Value |
| --- | --- |
| `dialect` | `sqlite` |
| `schema` | `./src/db/schema.ts` |
| `out` | `./drizzle` |
| `dbCredentials.url` | `process.env.SALES_DB_PATH`, falling back to `data/sales.db` |

The initial migration is `drizzle/0000_init.sql` with `drizzle/meta/_journal.json` and
`drizzle/meta/0000_snapshot.json`. Generate it with `pnpm db:generate --name init` once `src/db/schema.ts` matches
[04-Data-Model.md](04-Data-Model.md), and make sure every CHECK constraint from that document is in this first
migration before it is ever applied: SQLite cannot add a CHECK constraint to a table that already exists.

### 4.11 `pnpm-workspace.yaml`

| Key | Value | Reason |
| --- | --- | --- |
| `allowBuilds.better-sqlite3` | `true` | pnpm 11 blocks dependency build scripts by default; without this the native binding is never built |
| `allowBuilds.esbuild` | `true` | Needed by `tsx` and Vitest |
| `allowBuilds.sharp` | `false` | Not used (scaffold default) |
| `allowBuilds.unrs-resolver` | `false` | Not needed (scaffold default) |
| `minimumReleaseAgeExclude` | `["lucide-react@1.57.0"]` | Exempts the pinned icon release from pnpm's minimum-release-age policy |

No `packages` key: the project is a single package, not a monorepo.

### 4.12 `vitest.config.mts`

The extension must be `.mts` (a `.ts` config is loaded as CommonJS and Vite warns). `defineConfig` from
`vitest/config`:

| Key | Value |
| --- | --- |
| `resolve.alias["@"]` | Absolute path of `src`, resolved from `import.meta.dirname` |
| `test.environment` | `node` |
| `test.include` | `["tests/unit/**/*.test.ts", "tests/integration/**/*.test.ts"]` |

No jsdom, no setup files, no coverage thresholds. Tests never touch `data/`; they use temporary databases (see
[07-Testing.md](07-Testing.md)).

### 4.13 `playwright.config.ts`

`defineConfig` and `devices` from `@playwright/test`. Two constants: port `3200` and database
`.tmp/e2e/sales.db`.

| Key | Value | Reason |
| --- | --- | --- |
| `testDir` | `tests/e2e` | |
| `fullyParallel` | `false` | One shared SQLite file |
| `workers` | `1` | Tests run in order against the same database |
| `retries` | `1` when `process.env.CI` is set, else `0` | |
| `reporter` | `github` when `CI` is set, else `list` | |
| `expect.timeout` | `20000` | The dev server compiles each route on first hit |
| `timeout` | `60000` | Per test |
| `use.baseURL` | `http://localhost:3200` | |
| `use.trace` | `retain-on-failure` | |
| `projects` | One project: `name: "chromium"`, `use: devices["Desktop Chrome"]` | Chromium only |
| `webServer.command` | `node -e "require('fs').rmSync('.tmp/e2e',{recursive:true,force:true})" && next dev -p 3200` | Wipes the throwaway database cross-platform, then starts a dev server |
| `webServer.url` | `http://localhost:3200/api/settings` | Readiness probe; the first request also creates and migrates the database |
| `webServer.env` | `SALES_DB_PATH=.tmp/e2e/sales.db`, `NEXT_TELEMETRY_DISABLED=1` | Never touches `data/sales.db` |
| `webServer.reuseExistingServer` | `false` | Always a fresh server and database |
| `webServer.timeout` | `180000` | First compile can be slow |

### 4.14 `AGENTS.md` (framework-managed by Next.js)

`next dev` keeps a block delimited by `<!-- BEGIN:nextjs-agent-rules -->` and `<!-- END:nextjs-agent-rules -->`
in `AGENTS.md` at the project root and re-adds it whenever it is removed. Keep the file and commit it.

## 5. Required project layout

Every file below must exist. Paths are relative to the project root. Behaviour and module interfaces are specified
in the documents named in the comments (01 to 09 refer to the sibling spec files).

```text
.
├── AGENTS.md                     Created and maintained by `next dev` (section 4.14)
├── README.md                     User readme: prerequisites, quick start, database, demo data, backups, metrics, tests, troubleshooting, limitations
├── package.json                  4.1
├── pnpm-lock.yaml                Lockfile written by pnpm install (committed)
├── pnpm-workspace.yaml           4.11
├── tsconfig.json                 4.3
├── next.config.ts                4.4
├── eslint.config.mjs             4.5
├── .prettierrc.json              4.6
├── .prettierignore               4.6
├── .gitignore                    4.7
├── .env.example                  4.8
├── .nvmrc                        4.2
├── components.json               4.9
├── drizzle.config.ts             4.10
├── vitest.config.mts             4.12
├── playwright.config.ts          4.13
├── public/                       Scaffold SVGs (file, globe, next, vercel, window); unused by the app
├── drizzle/
│   ├── 0000_init.sql             Initial migration: tables, indexes, foreign keys, CHECK constraints (04)
│   └── meta/                     _journal.json and 0000_snapshot.json
├── scripts/                      tsx CLI entry points (08 section 6)
│   ├── db-migrate.ts             Create and migrate the database, print its path
│   ├── db-seed.ts                Seed demo data; refuse on a non-empty DB unless --force
│   ├── db-demo-remove.ts         Remove demo rows only, print counts
│   ├── db-backup.ts              Online backup to data/backups or a given directory
│   └── db-restore.ts             Verify a backup, save a safety copy, replace the DB, migrate
├── src/
│   ├── app/
│   │   ├── layout.tsx            Root layout: metadata, viewport theme colours, Geist fonts, Providers, AppShell (06)
│   │   ├── globals.css           Tailwind 4 imports, theme tokens, light and dark palettes (06)
│   │   ├── favicon.ico           App icon (scaffold)
│   │   ├── page.tsx              Overview page; Overview inside Suspense
│   │   ├── sales/page.tsx        Sales page; SalesTable inside Suspense
│   │   ├── products/page.tsx     Products page; ProductsView inside Suspense
│   │   ├── customers/page.tsx    Customers page; CustomersView inside Suspense
│   │   ├── analytics/page.tsx    Analytics page; AnalyticsView inside Suspense
│   │   ├── settings/page.tsx     Settings page; SettingsView inside Suspense
│   │   └── api/                  Route handlers (05)
│   │       ├── sales/route.ts                GET list, POST create, DELETE bulk
│   │       ├── sales/[id]/route.ts           GET, PUT, DELETE one sale
│   │       ├── sales/export/route.ts         GET filtered sales CSV
│   │       ├── products/route.ts             GET list, POST create
│   │       ├── products/[id]/route.ts        PUT update, PATCH archive state, DELETE
│   │       ├── products/export/route.ts      GET products CSV
│   │       ├── categories/route.ts           GET list, POST create
│   │       ├── categories/[id]/route.ts      PUT rename, DELETE
│   │       ├── customers/route.ts            GET list, POST create
│   │       ├── customers/[id]/route.ts       GET detail, PUT, DELETE
│   │       ├── customers/export/route.ts     GET customers CSV
│   │       ├── dashboard/route.ts            GET KPIs, series, rankings, breakdowns, insights
│   │       ├── reports/[report]/route.ts     GET report CSV
│   │       ├── settings/route.ts             GET, PUT settings
│   │       ├── system/route.ts               GET DB path, backups dir, row stats, backup list
│   │       ├── system/backup/route.ts        POST create backup
│   │       ├── system/demo/route.ts          POST seed or remove demo data
│   │       └── system/erase/route.ts         POST erase all data (typed confirmation)
│   ├── components/
│   │   ├── providers.tsx         Theme, QueryClient, Tooltip and Toaster providers; UI store rehydration
│   │   ├── shell/app-shell.tsx   Sidebar provider, skip link, sidebar, top bar, <main id="main">, global SaleDialog
│   │   ├── shell/app-sidebar.tsx Navigation sidebar (desktop rail, mobile drawer)
│   │   ├── shell/top-bar.tsx     Page title, date range, command menu trigger, new sale, theme toggle
│   │   ├── shell/command-menu.tsx       Command palette
│   │   ├── shell/date-range-picker.tsx  Preset and custom range selection
│   │   ├── shell/theme-toggle.tsx       Light / dark / system switch
│   │   ├── dashboard/overview.tsx       Overview body: KPIs, trend chart, top lists, insights
│   │   ├── dashboard/analytics-view.tsx Analytics body: charts, rankings, breakdowns, report exports
│   │   ├── dashboard/charts.tsx         TimeSeriesChart, RankingChart, DonutChart, chart palette
│   │   ├── dashboard/kpi-card.tsx       KPI card with delta and sparkline; skeleton
│   │   ├── sales/sales-table.tsx        Server-paged, sortable, filterable sales table with bulk actions
│   │   ├── sales/sale-dialog.tsx        Create / edit / duplicate sale form (one global instance)
│   │   ├── sales/sale-actions.tsx       Per-row action menu
│   │   ├── catalog/products-view.tsx    Products and categories management
│   │   ├── customers/customers-view.tsx Customers list, detail and editing
│   │   ├── settings/settings-view.tsx   Preferences, data info, backups, demo data, erase
│   │   ├── common/confirm.tsx           Confirmation dialog wrapper (AlertDialog)
│   │   ├── common/multi-select.tsx      Multi-value filter control
│   │   ├── common/page-skeleton.tsx     Suspense fallback for pages
│   │   ├── common/states.tsx            EmptyState and ErrorState
│   │   ├── common/status-badge.tsx      Sale status badge
│   │   └── ui/                   Generated shadcn components (section 3 step 8); only sidebar.tsx is edited
│   ├── db/
│   │   ├── schema.ts             Drizzle tables and inferred row types (04)
│   │   ├── client.ts             openDatabase, getDb singleton, DatabaseInitError, DB type (08 section 4)
│   │   └── paths.ts              projectRoot, dbPath, migrationsDir, backupsDir (08 section 3)
│   ├── hooks/use-mobile.ts       useIsMobile via useSyncExternalStore, breakpoint 768 px
│   ├── lib/
│   │   ├── utils.ts              Re-export of cn
│   │   ├── constants.ts          Statuses, payment methods, date formats, locales, page sizes, labels (03)
│   │   ├── money.ts              Currencies, limits, parsing, formatting, sale totals (03)
│   │   ├── date-range.ts         Presets, range resolution, previous period, buckets (03)
│   │   ├── validation.ts         Zod schemas for wire input, forms and queries (03, 05)
│   │   ├── csv.ts                CSV building with formula-injection guard, parsing, CSV responses (05)
│   │   ├── nav.ts                Navigation items and lookup by pathname (06)
│   │   ├── api-client.ts         fetch wrapper, ApiError, query-string builder, error message helper (06)
│   │   ├── queries.ts            TanStack Query hooks, mutation helper, default settings (06)
│   │   └── ui-store.ts           Zustand: persisted UI preferences, transient sale-dialog state (06)
│   └── server/
│       ├── db.ts                 server-only re-export of getDb
│       ├── api.ts                server-only withDb, errorResponse, readJson (05)
│       ├── errors.ts             AppError, notFound, conflict (05)
│       ├── demo-seed.ts          hasData, deterministic seedDemoData (05)
│       └── services/
│           ├── sales.ts          Sale CRUD, listing, filtering, export query
│           ├── catalog.ts        Categories and products
│           ├── customers.ts      Customers and customer detail
│           ├── analytics.ts      KPIs, series, rankings, breakdowns, insights, dashboard
│           ├── exports.ts        CSV builders for sales, reports, products, customers
│           ├── maintenance.ts    stats, removeDemoData, eraseAllData, backupTo, listBackups, verifyBackupFile (08)
│           └── settings.ts       getSettings, updateSettings
└── tests/                        (07)
    ├── helpers.ts                tmpDir, testDb, at, sale fixture; cleanup after each test
    ├── unit/                     money, date-range, validation, csv (*.test.ts)
    ├── integration/              db, sales, catalog, analytics, exports (*.test.ts)
    └── e2e/app.spec.ts           Playwright journeys against a throwaway database
```

### 5.1 Layering rules

| Rule | Detail |
| --- | --- |
| `server-only` | Imported only by `src/server/db.ts` and `src/server/api.ts`. `src/db/*` and `src/server/services/*` must stay importable from Vitest and `scripts/`, where `server-only` would throw. |
| Runtime | Node.js runtime only; no route or page declares the edge runtime. |
| Imports | The `@/` alias for everything under `src/`, including from `scripts/` and `tests/`. |
| Scripts | `tsx` executes scripts as CommonJS, so top-level `await` is not available: async scripts wrap their body in an `async function main()` whose rejection prints the error message and exits with code 1. |
| Generated directories | `.next/`, `data/`, `.tmp/`, `test-results/`, `playwright-report/` are never committed. |

See [../../ARCHITECTURE.md](../../ARCHITECTURE.md) for the layering rationale and [../../BUILD_PLAN.md](../../BUILD_PLAN.md) for
the build order.
