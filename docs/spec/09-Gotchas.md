# 09 — Gotchas (Risk Register)

Known pitfalls of this exact stack and these versions. Each one is cheap to avoid up front and expensive to debug later.
Read the whole list before Phase 1 and again before Phases 5 and 6.

## Toolchain

1. **Node 24, not 26.** A newer default `node` can break the native `better-sqlite3` binary. Use `.nvmrc` or put
   Node 24 first on `PATH` for *every* command, including `pnpm exec`.
2. **pnpm 11 blocks build scripts.** Without `allowBuilds: { better-sqlite3: true }` in `pnpm-workspace.yaml` the
   install succeeds, but the first DB access fails with "Could not locate the bindings file".
3. **TypeScript 6, not 7.** typescript-eslint requires `typescript <6.1`. **ESLint 9, not 10**
   (eslint-plugin-react peer).
4. **tsx runs scripts as CommonJS** → top-level `await` fails. Wrap the code in `async function main()`.
5. `vitest.config.mts`, not `.ts`: Vite warns about loading ESM as CommonJS otherwise.

## Next.js 16 (Cache Components on)

6. `cacheComponents: true`: pages are prerendered. Uncached dynamic work must be inside `<Suspense>`.
   **Client components are prerendered too.** Reading `Date.now()`/`new Date()` during render, or rendering TanStack
   Query (which reads the time), outside Suspense fails the build with an "unstable value … in a Client Component"
   error. Fix: every page wraps its client view in `<Suspense>`. Time-dependent values come from a
   `useSyncExternalStore` clock whose **server snapshot is 0**; the date range is `null` until mounted, and queries
   are `enabled: !!range`.
7. **Sync DB in GET handlers runs at build time** unless the handler opts out. Call `await connection()` (from
   `next/server`) before touching the DB (`withDb` does this). Symptom: `next build` creates `data/sales.db`.
8. **Route params are Promises**: `const { id } = await ctx.params` with `ctx: RouteContext<"/api/x/[id]">`.
   `RouteContext`/`LayoutProps` are generated global types, so `typecheck` = `next typegen && tsc --noEmit`.
9. `next build` type-checks the **whole** tsconfig project, including tests and scripts.
10. Navigation keeps hidden routes mounted (Activity). Keep dialog state in global stores, not in a page's local
    state, so dialogs don't reappear unexpectedly.
11. `next dev` writes a managed block into `AGENTS.md` in the app root; keep that file under version
    control so it stops showing up as a change.

## Libraries

12. **TanStack Table v9 is a new API.** Use `useTable` (not `useReactTable`) and `tableFeatures({ rowPaginationFeature, rowSortingFeature, rowSelectionFeature })`
    at **module scope**, `createColumnHelper<typeof features, Row>()` and `helper.columns([...])`. Render with
    `<table.FlexRender header={h} />` / `cell={cell}`. Pass `data: query.data?.rows ?? EMPTY` (a module-scope
    constant). Use `manualPagination`, `manualSorting`, `rowCount`, and controlled `state` plus the matching
    `on*Change`. Never copy query rows into state. API docs live in `node_modules/@tanstack/react-table/skills/`.
13. **shadcn 4 (radix-nova)** imports `cn` from the `cn` package (`src/lib/utils.ts` re-exports it), and
    `globals.css` imports `shadcn/tailwind.css` → `shadcn` must be a dependency. `shadcn init` needs `CI=1` (or the
    flags) to avoid an interactive prompt.
14. **Generated `globals.css` self-references `--font-sans`.** Map it to `var(--font-geist-sans)`. Load fonts with
    the `geist` package (`GeistSans.variable`), not `next/font/google`, so builds need no network.
15. **`SidebarInset` renders `<main>`** in the generated component. The shell has its own `<main id="main">`, so change
    it to a `div` (the only edit allowed in `components/ui`; mark it with a comment explaining why).
16. `AlertDialogDescription` renders a `<p>`: put inputs in `Confirm`'s `children`, never inside the description.
17. `server-only` throws outside a React Server environment: only `src/server/db.ts` and `src/server/api.ts` import
    it. Services stay importable in Vitest.
18. **Zustand persist + SSR:** `skipHydration: true` and `persist.rehydrate()` in an effect, otherwise you get
    hydration mismatches.
19. `react-hooks/set-state-in-effect` lint rule (eslint-config-next 16): don't `setState` in effects for derived
    browser values; use `useSyncExternalStore`. The generated `src/hooks/use-mobile.ts` trips this rule, so rewrite it on
    `matchMedia` + `useSyncExternalStore` (server snapshot `false`), as 06 §3 requires.
20. Recharts in hidden or zero-size containers renders nothing, and it animates on mount: wait ~1–2s before
    judging screenshots.

## Data

21. **SQLite CHECK constraints can't be added with ALTER.** Rebuilding `sales` with FKs on cascades and deletes
    `sale_items`. Get every constraint into the schema **before** generating `0000_init`, and check the generated SQL against 04.
22. **Aggregates in paged lists:** use correlated subqueries per row, not JOIN + GROUP BY + LIMIT (wrong totals and
    duplicated rows).
23. **LIKE search:** escape `%`, `_` and `\` and add `ESCAPE '\'`, otherwise searching "100%" matches everything.
24. **Local-time bucketing:** `date(occurred_at/1000,'unixepoch','localtime')` in SQL and `dayKey()` in JS must
    agree, or zero-filling misaligns.
25. **DB path from the project root**, not `cwd`: `next start` run from `src/` must use the same file.
    `projectRoot()` looks for `package.json` with `name: "sales-dashboard"`, so don't rename the package (or set
    `SALES_DASHBOARD_ROOT`).

## Workflow

26. Long tool output (test runners, builds) can get truncated or mangled. Redirect it to a log file and read the
    summary lines.
27. Verify UI in a real browser (Playwright or DevTools) and check the console. Typecheck alone misses runtime
    prerender errors and hydration issues.
