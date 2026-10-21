# 06 — Frontend and Design

This document specifies the complete client side of Ledgerline: visual direction, design tokens, the root layout and providers, the client data layer, client state stores, every shell, common and feature component, page files, responsive behaviour and accessibility. Domain formulas (money, ranges, totals, insights) are defined in [03-Domain-Rules.md](03-Domain-Rules.md); endpoint contracts in [05-Backend-and-API.md](05-Backend-and-API.md); framework pitfalls in [09-Gotchas.md](09-Gotchas.md).

All UI copy in this document is exact. Implement strings character for character (including the typographic quotes, en dash "–", minus sign "−", ellipsis "…" and middle dot "·").

## 1. Design direction

| ID | Requirement |
| --- | --- |
| UI-D1 | Premium SaaS analytics look, restrained. Cool neutral surfaces (oklch hue about 264), one indigo primary, colour used for meaning (status, deltas, charts), not decoration. |
| UI-D2 | Cards are flat: 1px border plus \`bg-card\`. No heavy shadows, no gradients except the soft fill under area charts and sparklines. |
| UI-D3 | Every figure (money, counts, percentages, dates in tables, pagination counters) uses \`tabular-nums\`. Order references and SKUs use the mono font (\`font-mono text-xs\`). |
| UI-D4 | Density: page gutter 16px (mobile) / 24px (md+), grid gaps 12–16px (mobile) and 16–24px (md+). Main column capped at 1400px and centred. |
| UI-D5 | Headings use \`tracking-tight\`. The page title lives in the top bar (one \`h1\` per page). Card titles are shadcn \`CardTitle\`. |
| UI-D6 | Light, dark and system themes, switched by a class on \`<html>\`. No flash, no transition on theme switch. |
| UI-D7 | Motion is minimal: shadcn/tw-animate enter/exit animations for overlays, Recharts default draw animation on main charts, none on sparklines. Everything respects \`prefers-reduced-motion\`. |

## 2. Design tokens (src/app/globals.css)

### 2.1 File structure

1. Imports, in order: \`tailwindcss\`, \`tw-animate-css\`, \`shadcn/tailwind.css\`.
2. Dark variant: \`@custom-variant dark (&:is(.dark *));\`
3. One \`@theme inline\` block mapping every token below to a Tailwind colour (\`--color-<name>: var(--<name>)\`) for: background, foreground, card, card-foreground, popover, popover-foreground, primary, primary-foreground, secondary, secondary-foreground, muted, muted-foreground, accent, accent-foreground, destructive, success, warning, border, input, ring, chart-1..chart-5, sidebar, sidebar-foreground, sidebar-primary, sidebar-primary-foreground, sidebar-accent, sidebar-accent-foreground, sidebar-border, sidebar-ring. (\`--chart-muted\` is used directly as \`var(--chart-muted)\` and needs no Tailwind colour.)
4. Fonts in the same \`@theme inline\` block:

| Theme variable | Value |
| --- | --- |
| \`--font-sans\` | \`var(--font-geist-sans), ui-sans-serif, system-ui, sans-serif\` |
| \`--font-mono\` | \`var(--font-geist-mono), ui-monospace, monospace\` |
| \`--font-heading\` | \`var(--font-geist-sans), ui-sans-serif, system-ui, sans-serif\` |

\`--font-sans\` must reference \`--font-geist-sans\` (the variable the \`geist\` package sets), never itself; the shadcn init writes a self-reference that must be replaced.

5. Radius scale in \`@theme inline\` (base \`--radius: 0.625rem\`):

| Variable | Value |
| --- | --- |
| \`--radius-sm\` | \`calc(var(--radius) * 0.6)\` |
| \`--radius-md\` | \`calc(var(--radius) * 0.8)\` |
| \`--radius-lg\` | \`var(--radius)\` |
| \`--radius-xl\` | \`calc(var(--radius) * 1.4)\` |
| \`--radius-2xl\` | \`calc(var(--radius) * 1.8)\` |
| \`--radius-3xl\` | \`calc(var(--radius) * 2.2)\` |
| \`--radius-4xl\` | \`calc(var(--radius) * 2.6)\` |

### 2.2 Colour tokens

Light values go on \`:root\`, dark values on \`.dark\`.

| Token | Light (\`:root\`) | Dark (\`.dark\`) |
| --- | --- | --- |
| \`--background\` | \`oklch(0.985 0.002 260)\` | \`oklch(0.16 0.008 265)\` |
| \`--foreground\` | \`oklch(0.2 0.012 265)\` | \`oklch(0.95 0.004 264)\` |
| \`--card\` | \`oklch(1 0 0)\` | \`oklch(0.2 0.009 265)\` |
| \`--card-foreground\` | \`oklch(0.2 0.012 265)\` | \`oklch(0.95 0.004 264)\` |
| \`--popover\` | \`oklch(1 0 0)\` | \`oklch(0.21 0.009 265)\` |
| \`--popover-foreground\` | \`oklch(0.2 0.012 265)\` | \`oklch(0.95 0.004 264)\` |
| \`--primary\` | \`oklch(0.47 0.16 264)\` | \`oklch(0.68 0.14 264)\` |
| \`--primary-foreground\` | \`oklch(0.985 0.002 260)\` | \`oklch(0.16 0.008 265)\` |
| \`--secondary\` | \`oklch(0.955 0.004 264)\` | \`oklch(0.26 0.01 265)\` |
| \`--secondary-foreground\` | \`oklch(0.25 0.012 265)\` | \`oklch(0.95 0.004 264)\` |
| \`--muted\` | \`oklch(0.955 0.004 264)\` | \`oklch(0.25 0.01 265)\` |
| \`--muted-foreground\` | \`oklch(0.5 0.014 265)\` | \`oklch(0.7 0.012 264)\` |
| \`--accent\` | \`oklch(0.95 0.008 264)\` | \`oklch(0.27 0.012 265)\` |
| \`--accent-foreground\` | \`oklch(0.25 0.012 265)\` | \`oklch(0.95 0.004 264)\` |
| \`--destructive\` | \`oklch(0.577 0.215 27)\` | \`oklch(0.68 0.18 22)\` |
| \`--success\` | \`oklch(0.56 0.13 158)\` | \`oklch(0.72 0.13 158)\` |
| \`--warning\` | \`oklch(0.68 0.14 70)\` | \`oklch(0.78 0.13 75)\` |
| \`--border\` | \`oklch(0.915 0.005 264)\` | \`oklch(1 0 0 / 9%)\` |
| \`--input\` | \`oklch(0.89 0.006 264)\` | \`oklch(1 0 0 / 14%)\` |
| \`--ring\` | \`oklch(0.6 0.12 264)\` | \`oklch(0.6 0.12 264)\` |
| \`--chart-1\` (indigo) | \`oklch(0.55 0.17 264)\` | \`oklch(0.68 0.15 264)\` |
| \`--chart-2\` (teal) | \`oklch(0.68 0.11 190)\` | \`oklch(0.74 0.11 190)\` |
| \`--chart-3\` (amber) | \`oklch(0.76 0.13 75)\` | \`oklch(0.8 0.12 75)\` |
| \`--chart-4\` (violet) | \`oklch(0.62 0.15 320)\` | \`oklch(0.7 0.14 320)\` |
| \`--chart-5\` (coral) | \`oklch(0.66 0.14 25)\` | \`oklch(0.72 0.13 25)\` |
| \`--chart-muted\` (previous period) | \`oklch(0.8 0.01 264)\` | \`oklch(0.42 0.012 264)\` |
| \`--radius\` | \`0.625rem\` | (inherits) |
| \`--sidebar\` | \`oklch(0.975 0.003 264)\` | \`oklch(0.18 0.008 265)\` |
| \`--sidebar-foreground\` | \`oklch(0.3 0.012 265)\` | \`oklch(0.85 0.006 264)\` |
| \`--sidebar-primary\` | \`oklch(0.47 0.16 264)\` | \`oklch(0.68 0.14 264)\` |
| \`--sidebar-primary-foreground\` | \`oklch(0.985 0 0)\` | \`oklch(0.16 0.008 265)\` |
| \`--sidebar-accent\` | \`oklch(0.93 0.008 264)\` | \`oklch(0.25 0.01 265)\` |
| \`--sidebar-accent-foreground\` | \`oklch(0.2 0.012 265)\` | \`oklch(0.95 0.004 264)\` |
| \`--sidebar-border\` | \`oklch(0.915 0.005 264)\` | \`oklch(1 0 0 / 8%)\` |
| \`--sidebar-ring\` | \`oklch(0.6 0.12 264)\` | \`oklch(0.6 0.12 264)\` |

### 2.3 Base styles and motion

| Selector | Rule |
| --- | --- |
| \`*\` (in \`@layer base\`) | \`border-border\`, \`outline-ring/50\` |
| \`body\` | \`bg-background\`, \`text-foreground\` |
| \`html\` | \`font-sans\` |
| \`h1, h2, h3\` | \`tracking-tight\` |
| \`@media (prefers-reduced-motion: reduce)\` on \`*, ::before, ::after\` | \`animation-duration: 0.01ms !important; animation-iteration-count: 1 !important; transition-duration: 0.01ms !important; scroll-behavior: auto !important\` |

Tint conventions used across components (Tailwind opacity modifiers on tokens): success tint \`bg-success/12 text-success\`, warning tint \`bg-warning/14 text-warning\`, destructive tint \`bg-destructive/10 text-destructive\`, count pill \`bg-primary/12 text-primary\`, soft panels \`bg-muted/30\` to \`bg-muted/50\`.

## 3. shadcn/ui setup

\`components.json\`: style \`radix-nova\`, \`rsc: true\`, \`tsx: true\`, Tailwind css path \`src/app/globals.css\`, baseColor \`neutral\`, \`cssVariables: true\`, empty prefix, iconLibrary \`lucide\`, aliases \`@/components\`, \`@/lib/utils\`, \`@/components/ui\`, \`@/lib\`, \`@/hooks\`, menuColor \`default\`, menuAccent \`subtle\`.

Required primitives in \`src/components/ui/\` (generate with the shadcn CLI; do not hand-write): alert-dialog, avatar, badge, button, calendar, card, chart, checkbox, command, dialog, dropdown-menu, empty, field, input-group, input, kbd, label, popover, select, separator, sheet, sidebar, skeleton, sonner, table, tabs, textarea, toggle-group, toggle, tooltip. The sidebar also generates \`src/hooks/use-mobile.ts\`.

Required edits to generated files:

| File | Edit | Reason |
| --- | --- | --- |
| \`src/components/ui/sidebar.tsx\` | \`SidebarInset\` must render a \`div\` (props typed \`React.ComponentProps<"div">\`), not \`main\`. Keep its classes and \`data-slot="sidebar-inset"\`. This is the only edit to this file. | The app shell renders its own \`<main id="main">\`; a page must have exactly one \`main\` landmark. |
| \`src/hooks/use-mobile.ts\` | \`useIsMobile()\` uses \`useSyncExternalStore\` on \`matchMedia("(max-width: 767px)")\` (breakpoint 768) with server snapshot \`false\`. | Avoids a setState-in-effect lint error and hydration mismatch. |

Keep the generated sidebar constants: width \`16rem\`, mobile width \`18rem\`, icon width \`3rem\`, keyboard shortcut Cmd/Ctrl+B. The sonner wrapper reads the theme from \`next-themes\` and styles toasts with \`--normal-bg: var(--popover)\`, \`--normal-text: var(--popover-foreground)\`, \`--normal-border: var(--border)\`, \`--border-radius: var(--radius)\`.

## 4. Root layout and providers

### 4.1 src/app/layout.tsx (server component)

| Item | Requirement |
| --- | --- |
| Fonts | Import \`GeistSans\` from \`geist/font/sans\` and \`GeistMono\` from \`geist/font/mono\` (package \`geist\`). |
| \`metadata\` | \`title: { default: "Ledgerline", template: "%s · Ledgerline" }\`, \`description: "Local-first sales dashboard"\`. |
| \`viewport.themeColor\` | \`[{ media: "(prefers-color-scheme: light)", color: "#ffffff" }, { media: "(prefers-color-scheme: dark)", color: "#0b0d14" }]\` |
| \`<html>\` | \`lang="en"\`, \`suppressHydrationWarning\`, className = GeistSans.variable + GeistMono.variable + \`antialiased\`. |
| \`<body>\` | className \`min-h-svh\`. Children: \`Providers\` wrapping \`AppShell\` wrapping \`{children}\`. |
| Props type | The Next 16 global \`LayoutProps<"/">\` type. |
| Global CSS | Import \`./globals.css\`. |

### 4.2 src/components/providers.tsx (client)

Signature: \`Providers({ children }: { children: ReactNode })\`.

Nesting order, outermost first:

1. \`ThemeProvider\` (next-themes): \`attribute="class"\`, \`defaultTheme="system"\`, \`enableSystem\`, \`disableTransitionOnChange\`.
2. \`QueryClientProvider\` with the client from \`getQueryClient()\`.
3. \`TooltipProvider\` with \`delayDuration={300}\`.
4. \`{children}\` followed by \`<Toaster position="bottom-right" />\` (inside the tooltip provider).

Query client:

- Default query options: \`staleTime: 30_000\`, \`refetchOnWindowFocus: false\`, \`retry: 1\`.
- \`getQueryClient()\`: on the server always create a new client; in the browser create once and reuse a module-level singleton.

Store hydration: a mount effect calls \`useUiStore.persist.rehydrate()\` (see §6.1).

## 5. Client data layer

### 5.1 src/lib/api-client.ts

| Export | Signature | Behaviour |
| --- | --- | --- |
| \`ApiError\` | \`class ApiError extends Error { status: number; issues: { path: string; message: string }[] }\`, constructor \`(status, message, issues = [])\` | Thrown for every non-2xx response. |
| \`api\` | \`api<T>(path: string, init?: RequestInit & { json?: unknown }): Promise<T>\` | If \`json\` is defined: sets \`content-type: application/json\` (caller headers override) and \`body = JSON.stringify(json)\`. Parses the body as JSON only when the response \`content-type\` contains \`application/json\`, else the body is \`null\`. Non-OK: throw \`ApiError(status, body?.error ?? "Request failed (<status>)", body?.issues)\`. OK: return the parsed body. |
| \`qs\` | \`qs(params: Record<string, string \| number \| boolean \| undefined \| null \| string[]>): string\` | Skips \`undefined\`, \`null\`, \`""\` and \`false\`. Arrays are joined with \`,\` and skipped when empty. Returns \`""\` when nothing remains, else \`"?" + URLSearchParams\`. |
| \`errorMessage\` | \`errorMessage(e: unknown): string\` | \`e.message\` for any \`Error\`, else \`"Something went wrong"\`. Used as toast description and in \`ErrorState\`. |

### 5.2 src/lib/queries.ts (\`"use client"\`)

\`DEFAULT_SETTINGS: Settings = { id: 1, currency: "USD", locale: "en-US", dateFormat: "MMM d, yyyy", defaultRange: "30d", updatedAt: 0 }\`, used whenever settings are not loaded yet.

| Hook | Signature | Query key | Endpoint | Options |
| --- | --- | --- | --- | --- |
| \`useSettings\` | \`()\` | \`["settings"]\` | \`GET /api/settings\` → \`Settings\` | \`staleTime: Infinity\` |
| \`useFormat\` | \`(): Format\` | — | — | Memoised on currency, locale, dateFormat. |
| \`useDateRange\` | \`(): { selection: RangeSelection; range: DateRange \| null; setRange(s: RangeSelection \| null): void }\` | — | — | See §5.3. |
| \`useDashboard\` | \`(range: DateRange \| null, bucket?: Bucket)\` | \`["dashboard", range?.from, range?.to, bucket ?? "auto"]\` | \`GET /api/dashboard?from&to[&bucket]\` → \`Dashboard\` | \`enabled: range !== null\`, \`placeholderData: keepPreviousData\` |
| \`useSales\` | \`(params: SalesParams, enabled = true)\` | \`["sales", params]\` | \`GET /api/sales\` + \`qs(params)\` → \`{ rows: SaleRow[]; total: number }\` | \`placeholderData: keepPreviousData\`, \`enabled\` |
| \`useSale\` | \`(id: string \| null)\` | \`["sale", id]\` | \`GET /api/sales/:id\` → \`SaleWithItems\` | \`enabled: Boolean(id)\` |
| \`useProducts\` | \`(opts: { q?: string; archived?: boolean } = {})\` | \`["products", opts]\` | \`GET /api/products?q&archived=1\` (archived sent only when true) → \`ProductRow[]\` | \`placeholderData: keepPreviousData\` |
| \`useCategories\` | \`()\` | \`["categories"]\` | \`GET /api/categories\` → \`{ id; name; productCount }[]\` | defaults |
| \`useCustomers\` | \`(q?: string)\` | \`["customers", q ?? ""]\` | \`GET /api/customers?q\` → \`CustomerRow[]\` | \`placeholderData: keepPreviousData\` |
| \`useInvalidateData\` | \`(): () => Promise<void>\` | — | — | Invalidates every query whose \`queryKey[0] !== "settings"\`. |
| \`useApiMutation\` | \`<TVars, TResult = unknown>(fn: (v: TVars) => Promise<TResult>)\` | — | — | \`useMutation({ mutationFn: fn, onSuccess: invalidate })\`. |

\`SalesParams\`: \`{ q?: string; status?: string[]; paymentMethod?: string[]; productId?: string; categoryId?: string; from?: number; to?: number; sort?: string; dir?: "asc" | "desc"; page?: number; pageSize?: number }\`. Types \`Settings\`, \`Dashboard\`, \`SaleRow\`, \`SaleWithItems\`, \`ProductRow\`, \`CustomerRow\` are imported as types only from the schema and service modules (see [04-Data-Model.md](04-Data-Model.md), [05-Backend-and-API.md](05-Backend-and-API.md)); client code never imports server values.

Two components run their own queries: the customer history sheet (\`["customer", id]\`, \`GET /api/customers/:id\`, enabled when id is not null) and the settings data card (\`["system"]\`, \`GET /api/system\`).

Mutation invalidation rule: every write (sale, product, category, customer, demo seed/remove, erase, backup) invalidates all data queries except \`settings\`, because any write can change every aggregate. The settings mutation instead writes the returned row into \`["settings"]\` with \`setQueryData\`.

\`Format\` object returned by \`useFormat()\`:

| Member | Behaviour |
| --- | --- |
| \`currency\` | Settings currency code. |
| \`locale\` | Settings locale. |
| \`money(minor)\` | \`formatMoney(minor, currency, locale)\` |
| \`compactMoney(minor)\` | \`formatMoney(minor, currency, locale, { compact: true })\` (axis ticks, donut centre). |
| \`number(n)\` | \`Intl.NumberFormat(locale).format(n)\` |
| \`date(ms \| Date)\` | date-fns \`format(value, dateFormat)\` |
| \`dateTime(ms \| Date)\` | date-fns \`format(value, dateFormat + ", HH:mm")\` |

### 5.3 useDateRange and the minute clock

- Selection = the stored \`useUiStore.range\` if not null, else \`{ preset: settings.defaultRange ?? "30d" }\`.
- A module-level minute clock feeds \`useSyncExternalStore\`: subscribe sets a 60 000 ms \`setInterval\`; the client snapshot is \`Math.floor(Date.now() / 60_000)\` cached in a module variable so the same minute returns the same number; the server snapshot is \`0\`.
- \`range\` is \`null\` when the clock is \`0\` (server render and hydration) or when there is no stored selection and settings are still pending. Otherwise \`range = resolveRange(selection, new Date(minute * 60_000))\`, memoised on the JSON of the selection, the minute, \`stored\` and \`settings.isPending\`.
- Effect: no \`Date.now()\` during server render (required by Next 16 \`cacheComponents\`), "today" rolls over within a minute, and range-driven queries stay disabled until the client knows the range.

## 6. Client state (Zustand)

### 6.1 useUiStore (src/lib/ui-store.ts)

UI preferences only; never server data. Created with \`persist\` middleware, storage key \`"sales-dashboard-ui"\`, \`version: 1\`, \`skipHydration: true\`.

| Field | Type | Default | Setter |
| --- | --- | --- | --- |
| \`sidebarOpen\` | \`boolean\` | \`true\` | \`setSidebarOpen(open)\` |
| \`range\` | \`RangeSelection \| null\` (\`null\` = use Settings default) | \`null\` | \`setRange(range)\` |
| \`chartType\` | \`"area" \| "bar" \| "line"\` | \`"area"\` | \`setChartType(t)\` |
| \`salesPageSize\` | \`number\` | \`25\` | \`setSalesPageSize(n)\` |

\`skipHydration\` makes the server render and the first client render match; \`Providers\` calls \`useUiStore.persist.rehydrate()\` in a mount effect.

### 6.2 useSaleDialog (same file, not persisted)

One global sale sheet serves the whole app.

| Field / action | Effect |
| --- | --- |
| \`open: boolean\` | Sheet visibility. |
| \`saleId: string \| null\` | Sale being edited. |
| \`duplicateId: string \| null\` | Sale being duplicated. |
| \`openNew()\` | \`{ open: true, saleId: null, duplicateId: null }\` |
| \`openEdit(id)\` | \`{ open: true, saleId: id, duplicateId: null }\` |
| \`openDuplicate(id)\` | \`{ open: true, saleId: null, duplicateId: id }\` |
| \`close()\` | \`{ open: false }\` (ids kept so the closing animation keeps its content). |

## 7. Navigation (src/lib/nav.ts)

\`NAV: { href: string; label: string; icon: LucideIcon; description: string }[]\`, in this order:

| href | label | lucide icon | description |
| --- | --- | --- | --- |
| \`/\` | Overview | \`LayoutDashboard\` | Key metrics for the selected period |
| \`/sales\` | Sales | \`Receipt\` | Every transaction, searchable and exportable |
| \`/products\` | Products | \`Package\` | Catalog, categories and product performance |
| \`/customers\` | Customers | \`Users\` | Customer records and purchase history |
| \`/analytics\` | Analytics | \`BarChart3\` | Trends, rankings and breakdowns |
| \`/settings\` | Settings | \`Settings\` | Preferences, data and backups |

\`navItemFor(pathname: string)\`: \`/\` matches only exactly; other items match when the pathname starts with their href; fallback is the first item.

## 8. Page files and component layout

Each route has a server-component \`page.tsx\` that exports \`metadata\` (except Overview) and renders \`<Suspense fallback={<PageSkeleton />}><View /></Suspense>\`. Views are client components (\`"use client"\`). The Suspense boundary is mandatory: views read \`useSearchParams\` or client-only state, which under \`cacheComponents\` must sit under a Suspense boundary.

| File | metadata.title | Rendered view (file) |
| --- | --- | --- |
| \`src/app/page.tsx\` | none (default "Ledgerline") | \`Overview\` (\`src/components/dashboard/overview.tsx\`) |
| \`src/app/sales/page.tsx\` | \`"Sales"\` | \`SalesTable\` (\`src/components/sales/sales-table.tsx\`) |
| \`src/app/products/page.tsx\` | \`"Products"\` | \`ProductsView\` (\`src/components/catalog/products-view.tsx\`) |
| \`src/app/customers/page.tsx\` | \`"Customers"\` | \`CustomersView\` (\`src/components/customers/customers-view.tsx\`) |
| \`src/app/analytics/page.tsx\` | \`"Analytics"\` | \`AnalyticsView\` (\`src/components/dashboard/analytics-view.tsx\`) |
| \`src/app/settings/page.tsx\` | \`"Settings"\` | \`SettingsView\` (\`src/components/settings/settings-view.tsx\`) |

Required component file layout:

| Directory | Files |
| --- | --- |
| \`src/components/\` | \`providers.tsx\` |
| \`src/components/shell/\` | \`app-shell.tsx\`, \`app-sidebar.tsx\`, \`top-bar.tsx\`, \`date-range-picker.tsx\`, \`command-menu.tsx\`, \`theme-toggle.tsx\` |
| \`src/components/common/\` | \`confirm.tsx\`, \`multi-select.tsx\`, \`page-skeleton.tsx\`, \`states.tsx\`, \`status-badge.tsx\` |
| \`src/components/sales/\` | \`sale-dialog.tsx\`, \`sales-table.tsx\`, \`sale-actions.tsx\` |
| \`src/components/dashboard/\` | \`kpi-card.tsx\`, \`charts.tsx\`, \`overview.tsx\`, \`analytics-view.tsx\` |
| \`src/components/catalog/\` | \`products-view.tsx\` |
| \`src/components/customers/\` | \`customers-view.tsx\` |
| \`src/components/settings/\` | \`settings-view.tsx\` |
| \`src/lib/\` | \`api-client.ts\`, \`queries.ts\`, \`ui-store.ts\`, \`nav.ts\` (plus the domain modules from 03) |
| \`src/hooks/\` | \`use-mobile.ts\` |

## 9. Shell components

### 9.1 AppShell (shell/app-shell.tsx, client)

Inside \`SidebarProvider\` (controlled: \`open = useUiStore.sidebarOpen\`, \`onOpenChange = setSidebarOpen\`), in order:

1. Skip link \`<a href="#main">Skip to content</a>\`: \`sr-only\`; on focus visible, fixed at top 8px / left 8px, \`z-50\`, rounded, \`bg-background\`, padding 12px × 8px, shadow.
2. \`AppSidebar\`.
3. \`SidebarInset\` (a \`div\`, §3) containing \`TopBar\` then \`<main id="main">\` with classes \`mx-auto w-full max-w-[1400px] flex-1 px-4 pt-4 pb-10 md:px-6 md:pt-6\` wrapping the page.
4. \`SaleDialog\` (global).

### 9.2 AppSidebar (shell/app-sidebar.tsx)

- \`<Sidebar collapsible="icon" aria-label="Main navigation">\`.
- Header: link to \`/\`, height 40px; a 28px square monogram "L" (\`rounded-md bg-primary text-primary-foreground\`, 13px semibold, \`aria-hidden\`) and the wordmark "Ledgerline" (sm, semibold, tracking-tight), hidden when collapsed to icons.
- Content: one group/menu with one \`SidebarMenuButton asChild\` per \`NAV\` item: \`isActive\` for \`navItemFor(pathname)\`, \`tooltip={label}\` (shown when collapsed), Next \`Link\` with \`aria-current="page"\` on the active item, icon plus label.
- Every link click calls \`setOpenMobile(false)\` so the mobile sheet closes on navigation.
- Footer: "Local data · single user" (xs, muted), hidden when collapsed.
- \`SidebarRail\` for drag/click toggle.
- Below 768px the sidebar renders as a sheet (via \`useIsMobile\`).

### 9.3 TopBar (shell/top-bar.tsx)

\`<header>\`: sticky top, \`z-20\`, height 56px (\`h-14\`), bottom border, \`bg-background/85\` with \`backdrop-blur\` (\`bg-background/70\` where backdrop-filter is supported), horizontal padding 12px (16px md+).

Left to right:

| Element | Detail |
| --- | --- |
| \`SidebarTrigger\` | \`aria-label="Toggle navigation"\` |
| Vertical \`Separator\` | 20px tall |
| \`<h1>\` | \`navItemFor(pathname).label\`, 15px semibold tracking-tight, truncates. |
| \`DateRangePicker\` | Only on \`/\` and \`/analytics\`; inline from \`sm\` up. |
| Search button (lg+) | Outline, muted text: Search icon, "Search…", \`<Kbd>⌘K</Kbd>\`. Opens the command menu. |
| Search icon button (below lg) | Ghost icon, \`aria-label="Search"\`. |
| \`ThemeToggle\` | §9.6 |
| Add sale | Primary button, Plus icon; text "Add sale" visible from \`sm\` up, \`sr-only\` "Add sale" below. Calls \`openNew()\`. |

- Below \`sm\`, on \`/\` and \`/analytics\`, a full-width bar holding the date picker is absolutely positioned under the header (\`top-14\`, bottom border, \`bg-background\`, padding 12px × 8px). Those two views add \`pt-12 sm:pt-0\` so content clears it.
- Global shortcut: a window \`keydown\` listener toggles the command menu on \`k\` with Meta or Ctrl and calls \`preventDefault\`.
- The top bar renders \`CommandMenu\` (controlled \`open\` state).

### 9.4 DateRangePicker (shell/date-range-picker.tsx)

Props \`{ className?: string }\`. Uses \`useDateRange\` and \`useFormat\`.

- Trigger: outline button, \`CalendarDays\` icon, label, \`ChevronDown\`; \`aria-label="Date range: <label>"\`.
- Label: custom selection with a resolved range shows \`"<date(from)> – <date(to - 1)>"\`; otherwise \`PRESET_LABELS[preset]\`.
- Popover \`align="end"\`, no padding; column on mobile, row from \`sm\`.
- Preset list (\`aria-label="Presets"\`, 160px column from \`sm\`): every preset except \`custom\`, small buttons, \`secondary\` variant for the active preset, \`ghost\` otherwise. Click: \`setRange({ preset })\` and close.
- Calendar: \`mode="range"\`, \`numberOfMonths={2}\`, days after today disabled, \`defaultMonth\` = draft start or the last day of the current range.
- Draft: when the popover opens, seed it from the current custom selection (or leave empty).
- Footer: "Cancel" (ghost sm, closes) and "Apply range" (sm, disabled until a start day is picked). Apply: \`setRange({ preset: "custom", customFrom: dayKey(from), customTo: dayKey(to ?? from) })\` and close.

### 9.5 CommandMenu (shell/command-menu.tsx)

Props \`{ open: boolean; onOpenChange(o: boolean): void }\`. Built on \`CommandDialog\` with \`title="Search"\` and \`description="Jump to a page, find a sale or run an action"\`.

- Input placeholder "Search pages, order IDs, products, customers…"; empty text "No results.".
- Term = trimmed input. When the term has 2 or more characters:
  - Query \`useSales({ q: term, pageSize: 6 }, open && term.length >= 2)\`.
  - Group "Sales": first item "Search all sales for “<term>”" (Receipt icon) navigates to \`/sales?q=<encoded term>\`; then one item per result: Receipt icon, order ref (mono xs), "<firstProduct>[ · <customerName>]" (muted, truncated), total money right-aligned. Selecting a result opens it in the edit sheet.
  - Item \`value\`s must contain the term (e.g. "search-all <term>", "sale <id> <term>") so the cmdk client filter does not hide server results.
- Group "Actions": "Add sale" (Plus) → \`openNew()\`.
- Group "Pages": one item per \`NAV\` entry with its icon → \`router.push(href)\`.
- Every selection closes the dialog, clears the input, then runs its action.

### 9.6 ThemeToggle (shell/theme-toggle.tsx)

Ghost icon button, \`aria-label="Change theme"\`: Sun in light, Moon in dark (via \`dark:\` classes, so nothing depends on hydration). Dropdown (\`align="end"\`) with a radio group bound to the next-themes \`theme\` (default \`"system"\`): "Light" (Sun), "Dark" (Moon), "System" (Monitor).

## 10. Common components

### 10.1 Confirm (common/confirm.tsx)

\`Confirm({ open, onOpenChange, title: string, description: ReactNode, action: string, onConfirm, pending?, children? })\`. Controlled \`AlertDialog\`: header with title and description; optional \`children\` between header and footer (outside \`AlertDialogDescription\`, which renders a \`<p>\`); footer with "Cancel" and a destructive action button labelled \`action\`, disabled while \`pending\`.

### 10.2 MultiSelect (common/multi-select.tsx)

\`MultiSelect<T extends string>({ label, options: readonly T[], labels: Record<T, string>, value: T[], onChange(v: T[]) })\`. Outline button: label, count pill (\`bg-primary/12 text-primary\`, xs, tabular) when \`value.length > 0\`, ChevronDown. Dropdown (\`align="start"\`) of checkbox items; selecting toggles the item and keeps the menu open.

### 10.3 PageSkeleton (common/page-skeleton.tsx, server-safe)

Container \`aria-busy="true"\`, \`aria-label="Loading"\`: a grid (1 column, 2 from \`sm\`, 5 from \`xl\`) of five 112px skeletons, then one 320px skeleton.

### 10.4 States (common/states.tsx)

| Component | Props | Render |
| --- | --- | --- |
| \`EmptyState\` | \`{ icon?: LucideIcon (default Inbox); title: string; description?: ReactNode; action?: ReactNode; className?: string }\` | shadcn \`Empty\` with icon media, title, optional description, optional action content. |
| \`ErrorState\` | \`{ error: unknown; onRetry?: () => void; className?: string }\` | \`EmptyState\` with \`AlertTriangle\`, title "Something went wrong", description \`errorMessage(error)\`; with \`onRetry\`, an outline sm button "Try again" (wired to the query's \`refetch\`). |

### 10.5 StatusBadge (common/status-badge.tsx)

\`StatusBadge({ status: string })\`: outline \`Badge\`, \`font-medium\`, a 6px round dot (\`bg-current\`, \`aria-hidden\`) then the \`STATUS_LABELS\` label (fallback: the raw status). The text always names the status.

| Status | Label | Classes |
| --- | --- | --- |
| completed | Completed | \`bg-success/12 text-success border-success/25\` |
| pending | Pending | \`bg-warning/14 text-warning border-warning/30\` |
| cancelled | Cancelled | \`bg-muted text-muted-foreground border-border\` |
| refunded | Refunded | \`bg-destructive/10 text-destructive border-destructive/25\` |

Demo rows show a small "Demo" marker next to the badge or name: \`text-[10px] uppercase tracking-wide text-muted-foreground\`.

## 11. Sales components

### 11.1 SaleDialog (sales/sale-dialog.tsx)

Container: \`Sheet\` \`side="right"\`, content \`w-full sm:max-w-xl\`, no padding, no gap. Any close (overlay, Escape, close button) calls \`close()\`.

Header (bottom border):

| Mode | Title | Description |
| --- | --- | --- |
| New | Add sale | Record a transaction. Amounts are in your reporting currency. |
| Edit (\`saleId\`) | Edit sale | Changes update every report that includes this sale. |
| Duplicate (\`duplicateId\`) | Duplicate sale | Record a transaction. Amounts are in your reporting currency. |

Loading: edit/duplicate load \`useSale(open ? saleId ?? duplicateId : null)\`. Without data show skeletons (two 36px, one 128px, 24px padding); on error show the message in \`text-destructive\`. Then render the form keyed by \`"<saleId>|<duplicateId>"\` so each open starts fresh.

Initial values:

| Field | New | Edit | Duplicate |
| --- | --- | --- | --- |
| occurredAt | now as \`yyyy-MM-dd'T'HH:mm\` | sale time | now |
| orderRef | "" | sale ref | "" |
| customerMode | walkin | existing if customerId, else walkin | as edit |
| customerId | "" | sale customerId or "" | as edit |
| customerName | "" | "" when linked, else walk-in name or "" | as edit |
| customerEmail | "" | "" | "" |
| paymentMethod | card | sale value | sale value |
| status | completed | sale value | completed |
| items | one empty item | one per line (below) | as edit |
| discount | "" | "" when 0, else \`toMajorString\` | as edit |
| taxRate | "0" | \`bpToPercentString(taxRateBp)\` | as edit |
| notes | "" | sale notes or "" | as edit |

Empty item: \`{ productMode: "existing", productId: "", productName: "", categoryName: "", quantity: "1", unitPrice: "", unitCost: "" }\`. Item from a sale line: \`existing\` when the line has a productId (name and category blank), else \`new\` with the snapshot name and category; quantity as string; prices via \`toMajorString\`; unitCost "" when null.

Form: React Hook Form, \`zodResolver(saleFormSchema)\`, \`mode: "onTouched"\`, \`noValidate\`, \`aria-label\` "Edit sale" or "Add sale", \`useFieldArray({ name: "items" })\`. The body scrolls (padding 24px × 20px, 24px between sections); totals and footer stay fixed below it.

Sections in order:

| # | Section | Fields and layout |
| --- | --- | --- |
| 1 | Header fields (2 columns from \`sm\`) | "Date & time" (\`datetime-local\`, id \`occurredAt\`); "Order reference" (id \`orderRef\`, placeholder "Auto-generated if blank"). |
| 2 | Fieldset "Items" | One item card per line (§11.2); "Add item" (outline sm, Plus, disabled at 50 items); array-level error below. |
| 3 | Fieldset "Customer" | Joined outline toggle group \`aria-label="Customer type"\`: "Walk-in", "Existing", "New" (deselect ignored). Walk-in: "Name on receipt (optional)", placeholder "Walk-in customer". Existing: full-width select, \`sr-only\` label "Customer", placeholder "Loading customers…" while pending else "Select a customer", options = all customers by name. New: 2 columns, "Name" and "Email (optional)" (\`type="email"\`). |
| 4 | Terms (2 columns from \`sm\`) | "Payment method" select (PAYMENT_LABELS, PAYMENT_METHODS order); "Status" select (STATUS_LABELS); "Discount" input group with the currency code as leading addon, \`inputMode="decimal"\`, placeholder "0.00"; "Tax rate" input group with trailing "%" addon, \`inputMode="decimal"\`. |
| 5 | Notes | "Notes" textarea, 2 rows. |

A field with an error sets \`data-invalid\` on its \`Field\`, \`aria-invalid\` on its control, and renders \`FieldError\` (role alert). Selects use \`position="popper"\`.

Validation messages (\`saleFormSchema\`):

| Path | Message |
| --- | --- |
| occurredAt | Enter a valid date and time |
| items (empty) | Add at least one item |
| items.N.productId | Select a product |
| items.N.productName | Product name is required |
| items.N.quantity | Quantity must be a whole number from 1 to 100,000 |
| items.N.unitPrice | Enter a valid price; Price is too large |
| items.N.unitCost | Enter a valid cost or leave blank |
| customerId | Select a customer |
| customerName (new) | Customer name is required |
| customerEmail | Enter a valid email |
| discount | Enter a valid discount or leave blank; Discount cannot exceed the subtotal |
| taxRate | Tax rate must be between 0 and 100% |

Live totals (below the scroll area; top border, \`bg-muted/40\`, padding 24px × 12px, sm, tabular, \`aria-live="polite"\`): 2-column \`dl\` with "Subtotal", "Discount" (value prefixed "−"), "Tax", "Total" (label medium, value base size semibold). Compute from the watched items, discount and tax rate: quantity = \`max(0, parseInt || 0)\`; price = \`parseMoney ?? 0\`; discount clamped to the subtotal; tax basis points clamped to 10 000; then \`computeSaleTotals\`. If it throws, render nothing.

Footer (top border, right-aligned): "Cancel" (ghost) and submit "Save changes" (edit) or "Record sale" (new/duplicate), disabled while saving with a spinning \`Loader2\`.

Submit (ignored while pending): \`POST /api/sales\` (new/duplicate) or \`PUT /api/sales/:id\` (edit) with \`saleFormToInput(values)\`.
- Success: toast.success "Sale recorded" or "Sale updated" with description "<orderRef> · <money(total)>"; invalidate data; close.
- Error: toast.error "Could not save sale" with description \`errorMessage(e)\`.

### 11.2 Item card

Card: rounded-lg border, \`bg-muted/30\`, 12px padding, \`role="group"\`, \`aria-label="Item <n>"\`. Inputs use \`bg-background\`. Control ids \`item-<index>-product|name|category|qty|price|cost\`.

1. Top row: joined outline sm toggle group \`aria-label="Product source"\`: "Catalog product" / "New product". Right: remove button (ghost icon-sm, Trash2, \`aria-label="Remove item <n>"\`), only when there is more than one item.
2. Catalog mode: "Product" select, placeholder "Loading products…" or "Select a product". Options: non-archived products plus the currently selected product even if archived (suffix " (archived)"). With no options show "No products yet — choose “New product”." Picking a product sets Unit price to its default price (validated) and Unit cost to its default cost (or "").
3. New mode (2 columns from \`sm\`): "Product name"; "Category" (placeholder "e.g. Coffee").
4. Bottom row, always 3 columns: "Qty" (\`inputMode="numeric"\`), "Unit price" (decimal, placeholder "0.00"), "Unit cost" (decimal, placeholder "Optional").

Products come from \`useProducts({ archived: true })\`, customers from \`useCustomers()\`.

### 11.3 SaleActions (sales/sale-actions.tsx)

\`SaleActions({ id, orderRef })\`. Trigger: ghost icon-sm \`MoreHorizontal\`, \`aria-label="Actions for <orderRef>"\`. Menu (\`align="end"\`): "View" (Eye), "Edit" (Pencil → \`openEdit\`), "Duplicate" (Copy → \`openDuplicate\`), separator, "Delete" (Trash2, destructive variant).

Delete confirmation (AlertDialog):
- Title: "Delete sale <orderRef>?"
- Description: "This permanently removes the sale and its line items from every report. To keep history, mark it Cancelled or Refunded instead."
- Buttons: "Keep sale" and destructive "Delete sale" (disabled while pending).
- Request: \`DELETE /api/sales/:id\` via \`useApiMutation\`. Toasts: "Sale <orderRef> deleted"; error "Could not delete sale" plus message.

View dialog (\`sm:max-w-lg\`; loads \`useSale(id)\` only while open):
- Title: order ref in mono ("Sale" while loading). Description: "<dateTime(occurredAt)> · <customerName or "Walk-in customer">", or "Loading…".
- Loading body: 160px skeleton.
- Body (sm):
  1. \`StatusBadge\`, payment label (muted), "Demo" chip when \`isDemo\`.
  2. Items table: "Item" (name, category on a muted second line), "Qty", "Price", "Line" (line subtotal); numbers right-aligned.
  3. \`dl\`: "Subtotal", "Discount" (−), "Tax (<rate>%)", "Total" (semibold).
  4. Notes in a muted panel when present.

### 11.4 SalesTable (sales/sales-table.tsx)

Server-driven table: TanStack Table v9 with manual sorting and pagination. The v9 API differs from v8:

\`\`\`ts
const features = tableFeatures({ rowPaginationFeature, rowSortingFeature, rowSelectionFeature });
const helper = createColumnHelper<typeof features, SaleRow>();
const table = useTable({ features, columns, data, rowCount, state, ... });
// render with <table.FlexRender header={h} /> and <table.FlexRender cell={c} />
\`\`\`

Use a stable module-level empty array for \`data\` while loading. Options: \`getRowId: r => r.id\`, \`manualPagination\`, \`manualSorting\`, \`enableSortingRemoval: false\`, \`sortDescFirst: true\`, \`rowCount = total\`. Initial sorting \`[{ id: "date", desc: true }]\`; page size from \`useUiStore.salesPageSize\`. A sorting change also resets to page 0.

Columns:

| id | Header | Sortable | Cell |
| --- | --- | --- | --- |
| select | Checkbox "Select all on this page" (checked / indeterminate / unchecked) | no | Checkbox "Select <orderRef>" |
| date | Date | yes | \`dateTime(occurredAt)\`, nowrap, muted |
| orderRef | Order | no | mono xs |
| firstProduct | Product | no | first product (or "—") plus muted " +N more" when itemCount > 1; max width 224px, truncated |
| customerName | Customer | no | name, or muted "Walk-in"; max width 160px, truncated |
| quantity | Qty (right) | yes | units, right, tabular |
| amount | Total (right) | yes | \`money(totalMinor)\`, right, medium, tabular |
| paymentMethod | Payment | no | PAYMENT_LABELS |
| status | Status | yes | StatusBadge plus "Demo" marker |
| actions | sr-only "Actions" | no | \`SaleActions\` |

Sortable headers are ghost sm buttons: label plus \`ArrowUp\` (asc), \`ArrowDown\` (desc) or \`ArrowUpDown\` (unsorted, \`text-muted-foreground/60\`); numeric headers right-aligned. Sorted header cells set \`aria-sort\` "ascending"/"descending". The column id is sent as \`sort\` (date, quantity, amount, status) with \`dir\`.

Toolbar (wrapping row, 8px gap), in order:

| Control | Detail |
| --- | --- |
| Search | Input group with Search icon; full width on mobile, 288px from \`sm\`; \`aria-label="Search sales"\`; placeholder "Order ID, product or customer". Debounced 250 ms (trimmed). Seeded from \`?q=\`; when \`?q=\` changes (command menu), box and query both update. |
| Date | Select 144px, \`aria-label="Date range"\`: "All time" plus every preset except custom. A preset is resolved once, at selection, into \`from\`/\`to\`; "All time" clears them. Independent of the global range. |
| Status | MultiSelect "Status" over SALE_STATUSES. |
| Payment | MultiSelect "Payment" over PAYMENT_METHODS. |
| Category | Select 160px, \`aria-label="Category"\`: "All categories" plus categories. |
| Product | Select 176px, \`aria-label="Product"\`: "All products" plus all products including archived. |
| Clear filters | Ghost button with X, only when any filter (search, date, status, payment, category, product) is active; resets all. |
| Right group | "Export CSV" (outline, Download): \`<a download>\` to \`/api/sales/export\` + \`qs(filters, sort, dir)\`. "Add" (Plus, primary) only below \`sm\`. |

A change of the filter set (compare the serialised filters) resets to page 0 and clears the selection; do this during render, not in an effect.

Bulk bar (when at least one row is selected; rounded border, \`bg-muted/40\`, \`role="status"\`): "<n> selected", "Clear selection" (ghost sm), "Delete selected" (destructive sm, Trash2, right). Confirmation: title "Delete <n> sales?", description "These sales and their line items are removed from every report. This cannot be undone.", buttons "Cancel" and "Delete sales". Request \`DELETE /api/sales\` with JSON \`{ ids }\`. Toasts: "<deleted> sales deleted" (then clear selection); error "Could not delete sales" plus message.

Body (in a \`Card\` with no vertical padding):

| State | Render |
| --- | --- |
| Error | \`ErrorState\` with retry |
| No data yet | Eight 36px skeleton rows |
| \`total === 0\`, filters active | EmptyState (Receipt): "No sales match these filters" / "Try removing a filter or widening the date range." / outline "Clear filters" |
| \`total === 0\`, no filters | EmptyState (Receipt): "No sales yet" / "Record your first sale, or load demo data from Settings." / "Add sale" (Plus) |
| Rows | Table with \`aria-busy\` = fetching, 60% opacity while showing placeholder data, selected rows \`data-state="selected"\`, first and last columns 40px wide with 16px outer padding. |

Pagination (\`<nav aria-label="Pagination">\`, only when total > 0): left "<first>–<last> of <total>" (total via \`toLocaleString()\`, muted, tabular). Right: "Rows" plus a select (sm, 80px, \`aria-label="Rows per page"\`, 10/25/50/100) that saves to the store and resets to page 0; "Page <n> of <pages>" (pages = max(1, ceil(total / size))); "Previous page" / "Next page" outline icon-sm buttons (ChevronLeft/Right) disabled at the ends.

## 12. Dashboard components

### 12.1 KpiCard (dashboard/kpi-card.tsx)

\`KpiCard({ label, value, current, previous, spark?, hint? })\` (\`value\` is the formatted string).

- Card: 16px padding, 8px gap. Title 13px medium muted. Value \`text-2xl\` semibold tracking-tight tabular, truncated, \`title={value}\`.
- Delta row (\`title="Compared with the previous period of equal length"\`), \`change = percentChange(current, previous)\`:
  - null: muted \`hint ?? "No prior-period data"\`.
  - otherwise a pill with arrow icon and \`(change > 0 ? "+" : "") + change.toFixed(1) + "%"\`, then sr-only "vs previous period".

| Condition | Icon | Pill classes |
| --- | --- | --- |
| change > 0.05 | ArrowUpRight | \`bg-success/12 text-success\` |
| change < −0.05 | ArrowDownRight | \`bg-destructive/10 text-destructive\` |
| otherwise | Minus | \`bg-muted\` |

- Sparkline at the right of the delta row, only when \`spark\` has more than one point: Recharts \`AreaChart\` 72×28, \`aria-hidden\`, monotone area, stroke \`var(--chart-1)\` width 1.5, vertical gradient fill from \`var(--chart-1)\` at opacity 0.35 to 0 (gradient id unique per label), no dots, no axes, \`isAnimationActive={false}\`.
- \`KpiSkeleton\`: card with skeleton lines 16×96px, 28×128px, 12×112px.

### 12.2 Charts (dashboard/charts.tsx)

All charts use shadcn \`ChartContainer\` with a \`ChartConfig\` and set \`accessibilityLayer\`. \`PALETTE = [chart-1, chart-2, chart-3, chart-4, chart-5, chart-muted]\` (as \`var(--…)\`). Main charts keep the Recharts default animation.

**TimeSeriesChart** \`({ series, previous?, bucket, metric = "revenueMinor", kind = "area", className = "h-64 w-full" })\`

- \`SeriesMetric\`: \`"revenueMinor" | "orders" | "units" | "aovMinor"\`, labels Revenue / Sales / Units / Avg. order value. Revenue and AOV are money, the others numbers. \`ChartKind\`: \`"area" | "bar" | "line"\`.
- Points: \`{ key, current: series[i][metric], previous: previous?.[i]?.[metric] ?? null }\` (previous aligned by bucket index).
- Config: \`current\` = metric label, \`var(--chart-1)\`; \`previous\` = "Previous period", \`var(--chart-muted)\`.
- Margins left 4, right 8, top 8, bottom 0. Horizontal grid lines only.
- X axis: \`key\`, no tick/axis line, tick margin 8, \`minTickGap\` 24, labels \`formatBucketLabel(key, bucket)\`.
- Y axis: no lines, width 64, ticks \`compactMoney\` (money) or \`number\`, \`allowDecimals={false}\`.
- Tooltip: label = bucket label; rows show series label (muted) and full value (\`money\` or \`number\`, medium, tabular).

| kind | Previous series | Current series |
| --- | --- | --- |
| area | monotone area, stroke previous colour, dash "4 4", no fill, width 1.5 | monotone area, stroke width 2, vertical gradient fill of current colour, opacity 0.28 to 0.02 |
| bar | bar, previous colour, top radius 3 | bar, current colour, top radius 3 |
| line | line, dash "4 4", no dots, width 1.5 | line, no dots, width 2 |

**RankingChart** \`({ rows: { name; value }[], money = true, className = "h-64 w-full" })\`

- Horizontal bars (\`layout="vertical"\`), margins left 4, right 16, top 0, bottom 0, vertical grid lines only, hidden numeric X axis.
- Category Y axis: width 120, no lines; names longer than 18 characters are cut to 17 plus "…".
- Bars \`var(--chart-1)\`, radius 4. Tooltip: no cursor, no indicator, value via \`money\` or \`number\`. Config label "Revenue" (money) or "Value".

**DonutChart** \`({ rows, centerLabel = "Total", className = "h-56 w-full" })\`

- More than 6 rows: the first 5 plus "Other" (sum of the rest).
- Layout: 2 columns from \`sm\`; chart left (square, max height 224px), legend right.
- Pie: inner radius 62%, outer 92%, segment stroke \`var(--card)\` width 2, colours \`PALETTE[i % 6]\`.
- Centre: total via \`compactMoney\` (base, semibold, foreground) above \`centerLabel\` (xs, muted).
- Tooltip: name (muted) and \`money\` value.
- Legend: \`aria-hidden\` 10px rounded-sm swatch, truncated name, share \`toFixed(1)\` + "%" ("0.0%" when the total is 0), muted, tabular.

### 12.3 Overview (dashboard/overview.tsx)

Uses \`useDateRange().range\` and \`useDashboard(range)\` (auto bucket). On error the whole view is \`ErrorState\` with retry. Root: vertical stack, 16px gap (24px md+), \`pt-12 sm:pt-0\`.

1. **KPI grid** (\`<section aria-label="Key metrics">\`; 1 column, 2 from \`sm\`, 3 from \`lg\`, 5 from \`xl\`; gap 12px, 16px md+). Five \`KpiSkeleton\`s until data, then:

| Card | Value | current / previous | Sparkline |
| --- | --- | --- | --- |
| Total revenue | money(revenueMinor) | revenueMinor | series revenueMinor |
| Sales | number(salesCount) | salesCount | series orders |
| Avg. order value | money(aovMinor) | aovMinor | series aovMinor |
| Units sold | number(units) | units | series units |
| Net profit | money(profitMinor) when current profit is not null, else "—" | profit vs (previous profit ?? 0); unknown: 0 vs 0 | none |

Net profit hint: known profit, "Revenue minus product cost"; unknown, "<salesCount − costedCount> sales missing cost" when there are completed sales, else "No completed sales".

2. **Revenue note** (xs muted), only when pendingCount > 0 or refundedCount > 0: "Revenue counts completed sales only (net of discounts, excluding tax). " then, if pending, "<money(pendingMinor)> pending in <pendingCount> sales. " then, if refunded, "<money(refundedMinor)> refunded across <refundedCount> sales."
3. **No-sales state**: when loaded and the completed, pending, refunded and cancelled counts are all 0, replace rows A and B with a card holding EmptyState "No sales in this period" / "Record a sale or pick a wider date range. You can also load demo data from Settings." / "Add sale" (Plus) and an outline link "Open settings" (\`/settings\`).
4. **Row A** (3 columns from \`lg\`): "Revenue over time" card spanning 2 columns, description "By <bucket>, dashed line is the previous period" (a non-breaking space while loading), area \`TimeSeriesChart\` with the previous series, 288px (288px skeleton while loading); then the Insights card.
5. **Row B** (3 columns from \`lg\`; content 224px):

| Card | Description | Content |
| --- | --- | --- |
| Sales volume | "Completed sales per <bucket>" ("day" while loading) | TimeSeriesChart, metric orders, kind bar, no previous |
| Revenue by product | "Top 6 products" | RankingChart of the first 6 products by revenue, or EmptyState "No completed sales" |
| Sales by category | "Share of revenue" | DonutChart of categories, centre label "Revenue", 176px, or EmptyState "No completed sales" |

6. **Recent sales** card (always shown).
   - Description "Latest transactions in this period, any status"; card action: ghost sm link "View all" with ArrowRight to \`/sales\`.
   - Data: \`useSales({ from, to, sort: "date", dir: "desc", pageSize: 8 }, range !== null)\`.
   - States: \`ErrorState\` on error; five 36px skeletons while loading; EmptyState "No sales in this period" when empty.
   - Columns: Date (\`date\`, muted, nowrap), Order (mono), Product (muted " +N" when itemCount > 1, truncated), Customer (hidden below \`md\`; muted "Walk-in"), Qty (right), Total (right, medium), Payment (hidden below \`lg\`), Status (badge), actions (\`SaleActions\`). Outer columns get 24px padding.

**Insights card**: title "Insights", description "What stands out this period".
- Loading: 240px skeleton.
- Empty list: EmptyState (Lightbulb) "Not enough data yet" / "Insights appear once there are at least 3 completed sales in the period."
- Otherwise a divided \`dl\`; per insight \`{ id, title, value, detail?, tone, money? }\`:
  - Term: title (sm, muted).
  - Value (sm medium, right-aligned): tone \`positive\` adds TrendingUp and \`text-success\`; \`negative\` adds TrendingDown and \`text-destructive\`. For id \`best-day\` the value is a day key shown as \`date(parseDay(value))\`.
  - Detail: xs muted line, formatted with \`money\` when \`money\` is true.

### 12.4 AnalyticsView (dashboard/analytics-view.tsx)

Local state: bucket (\`"auto" | "day" | "week" | "month"\`, default auto), metric (default revenueMinor), rank-by (\`"revenue" | "units"\`, default revenue). Chart kind is \`useUiStore.chartType\` (persisted). Data: \`useDashboard(range, bucket === "auto" ? undefined : bucket)\`; on error the view is \`ErrorState\`. Root: stack, \`pt-12 sm:pt-0\`.

1. Header row: muted sm text "Revenue = completed sales, net of discounts, excluding tax. All figures in <currency>." and, right-aligned, an "Export report" dropdown (outline, Download, disabled until the range is known). Menu label "CSV for the selected range"; items are \`<a download>\` links to \`/api/reports/<id>?from&to\`:

| id | Label |
| --- | --- |
| timeseries | Revenue over time |
| products | Product ranking |
| categories | Categories |
| customers | Customers |
| payments | Payment methods |

2. **Trends** card: title "Trends", description "Dashed line shows the previous period of equal length." Controls on the right (wrapping):
   - Tabs: Revenue / Sales / Avg. order / Units.
   - Grouping select (sm, 112px, \`aria-label="Grouping"\`): Auto / Daily / Weekly / Monthly.
   - Chart-type toggle group (outline sm, \`aria-label="Chart type"\`), icon items "Area chart", "Bar chart", "Line chart" (lucide AreaChart, BarChart3, LineChart).
   - Body: TimeSeriesChart with previous series, chosen metric and kind, 288px.
3. Grid, 2 columns from \`lg\`:

| Card | Description | Content |
| --- | --- | --- |
| Product ranking | "Top products by revenue" / "Top products by units sold" | Card action: toggle "Revenue"/"Units". RankingChart of the top 10 by the chosen measure, \`money\` only for revenue, 320px. |
| Revenue by category | "Share of completed revenue" | DonutChart, 320px |
| Payment methods | "Revenue by how customers paid" | DonutChart with PAYMENT_LABELS names |
| Day of week | "Revenue by weekday (local time)" | RankingChart, 224px, rows Mon, Tue, Wed, Thu, Fri, Sat, Sun (dow 1–6 then 0), missing days 0 |
| Top customers (spans 2) | "Walk-in sales are grouped together" | Table of the first 10: Customer (medium; walk-in row muted), Orders, Avg. order (\`round(revenue / orders)\`, 0 when no orders), Revenue (medium); numbers right-aligned |
| Sales by status (spans 2) | "Only completed sales count toward revenue. Refunds are full refunds and remove the sale from revenue." | Table: Status (badge), Sales, Net amount |

Empty ranking/donut/customer datasets show the centred muted line "No completed sales in this period." Loading shows a skeleton at the card's chart height.

## 13. Catalog, customers and settings views

### 13.1 ProductsView (catalog/products-view.tsx)

\`Tabs\` (default "products", 16px gap), triggers "Products" (Package) and "Categories" (Tags).

**Products tab**

Toolbar:
- Search (288px from \`sm\`, \`aria-label="Search products"\`, placeholder "Name or SKU", debounced 250 ms).
- Checkbox "Show archived".
- Right: "Export CSV" (\`<a download>\` to \`/api/products/export\`), "Add product" (Plus).

Data \`useProducts({ q, archived: showArchived })\`. Card states:
- Error: \`ErrorState\` with retry.
- Loading: six 36px skeletons.
- Empty with search: "No products match your search".
- Empty without search: "No products yet" / "Add products to your catalog, or create them on the fly while recording a sale." / "Add product".

| Column | Detail |
| --- | --- |
| Product | Name (medium, truncated at 224px), outline badge "Archived", "Demo" marker. Archived rows use muted text. |
| SKU | Hidden below \`md\`; mono xs; "—" when empty. |
| Category | Name, or muted "Uncategorized". |
| Price | Right; \`money(defaultPriceMinor)\`. |
| Cost | Hidden below \`lg\`; right; muted "—" when null. |
| Units sold | Right; \`number\`. |
| Revenue | Right; medium. |
| Actions | Menu \`aria-label="Actions for <name>"\`: "Edit"; "Archive" / "Restore" (Archive / ArchiveRestore; \`PATCH /api/products/:id\` \`{ archived }\`; toast "<name> archived" / "<name> restored"; error toast with the message); separator; "Delete" (destructive). |

Footnote (xs muted): "Units and revenue count completed sales only. Sales keep a snapshot of the product name, price and cost, so editing a product never rewrites history."

Delete \`Confirm\`:
- Title "Delete <name>?"; description "Products that appear in sales cannot be deleted; archive them instead to hide them from new sales."; action "Delete product".
- Request \`DELETE /api/products/:id\`. Toasts "Product deleted" / "Could not delete product" plus message.

Product dialog (\`sm:max-w-lg\`):
- Title "Add product" / "Edit product"; description "Default price and cost prefill new sales; past sales are unchanged."
- Form (RHF + \`productFormSchema\`, keyed by id or "new"):
  - "Name" (autofocus).
  - 2-column grid: "SKU" (placeholder "Optional"); "Category" select ("Uncategorized" plus categories; "Uncategorized" maps to an empty value); "Default price" (decimal, placeholder "0.00"); "Unit cost" (decimal, placeholder "Optional").
  - "Description" textarea, 2 rows.
- Footer: "Cancel" and "Add product" / "Save changes".
- Request \`POST /api/products\` or \`PUT /api/products/:id\` with \`productFormToInput\`. Toasts "Product added" / "Product updated"; error "Could not save product" plus message.
- Messages: "Name is required", "Enter a valid price", "Price is too large", "Enter a valid cost or leave blank".

**Categories tab** (max width 672px)
- Add form: input (\`aria-label\` and placeholder "New category name", maxLength 60) and submit "Add" (Plus), disabled when the trimmed name is empty or while pending. \`POST /api/categories { name }\`; toast "Category added" and clear; error toast with the message.
- List card: 96px skeleton while loading; EmptyState (Tags) "No categories yet" / "Categories group products in reports."
- Row: name (medium), "<n> product" / "<n> products" (muted, tabular), ghost icon buttons "Rename <name>" (Pencil) and "Delete <name>" (Trash2).
- Inline rename: autofocused input (\`aria-label="Category name"\`, maxLength 60), "Save" (disabled when blank), "Cancel"; Escape cancels. \`PUT /api/categories/:id\`; toast "Category renamed"; error toast with the message.
- Footnote: "Renaming a category does not change the category recorded on past sales."
- Delete \`Confirm\`: title "Delete category <name>?"; description "Only categories with no products can be deleted. Move or delete its products first."; action "Delete category"; toasts "Category deleted" / "Could not delete category" plus message.

### 13.2 CustomersView (customers/customers-view.tsx)

Toolbar: search (\`aria-label="Search customers"\`, placeholder "Name, email or phone", debounced 250 ms); right: "Export CSV" (\`/api/customers/export\`), "Add customer".

Card states:
- Error: \`ErrorState\` with retry.
- Loading: six 36px skeletons.
- Empty with search: "No customers match your search".
- Empty without search: "No customers yet" / "Customers are optional. Walk-in sales are recorded without a customer record." / "Add customer".

| Column | Detail |
| --- | --- |
| Customer | Name as a text button (medium, underline on hover, visible focus ring) that opens the history sheet; "Demo" marker. |
| Email | Hidden below \`md\`; muted; "—". |
| Phone | Hidden below \`lg\`; muted; "—". |
| Orders | Right. |
| Revenue | Right, medium. |
| Last sale | Hidden below \`sm\`; right; \`date\` or "—". |
| Actions | Menu \`aria-label="Actions for <name>"\`: "View history" (Users), "Edit" (Pencil), separator, "Delete". |

Footnote: "Orders and revenue count completed sales only."

Delete \`Confirm\`: title "Delete <name>?"; description "Customers with recorded sales cannot be deleted, so their history stays intact."; action "Delete customer"; toasts "Customer deleted" / "Could not delete customer" plus message.

History sheet (right, \`w-full sm:max-w-lg\`, query \`["customer", id]\`):
- Title: customer name ("Customer" while loading). Description: email and phone joined by " · ", or "Purchase history".
- Body: \`ErrorState\` with retry on error, 160px skeleton while loading, then:
  1. Three stat tiles "Revenue", "Orders", "Avg. order" ("—" when no orders), computed from completed history rows (revenue = sum of \`netMinor\`).
  2. Notes panel (preserves line breaks).
  3. History list, or "No sales recorded for this customer yet." Each row is a button that opens the sale in the edit sheet: order ref (mono xs), "<date> · <payment label>", status badge, total (96px, right).
  4. Footnote "Revenue excludes tax and counts completed sales only. Showing up to 200 most recent sales."

Customer dialog (\`sm:max-w-md\`):
- Title "Add customer" / "Edit customer"; description "Contact details are optional and stay on this machine."
- Fields: "Name" (autofocus); 2 columns "Email" (email, placeholder "Optional") and "Phone" (tel, placeholder "Optional"); "Notes" (3 rows).
- Footer: "Cancel" and "Add customer" / "Save changes".
- Request \`POST /api/customers\` or \`PUT /api/customers/:id\`. Toasts "Customer added" / "Customer updated"; error "Could not save customer" plus message.
- Messages: "Name is required", "Enter a valid email".

### 13.3 SettingsView (settings/settings-view.tsx)

Centred column, max width 768px, 24px gap: Preferences, Data & storage, Export.

**Preferences**: description "Changes are saved immediately and apply across the app." Five 36px skeletons while loading; \`ErrorState\` on error. Each row is a responsive \`Field\` (label and description left, select 224px from \`sm\`), disabled while saving:

| id | Label | Description | Options |
| --- | --- | --- | --- |
| currency | Reporting currency | Amounts are displayed in this currency. Existing values are not converted — no exchange rates are applied. | CURRENCIES |
| locale | Number locale | "Controls number and currency formatting, e.g. " + Intl currency format of 1234.5 in the current locale and currency | LOCALES |
| date-format | Date format | none | DATE_FORMATS, each labelled by formatting 14 March 2026 15:30 (local) with that pattern |
| default-range | Default date range | Used when the app opens. | presets except custom, PRESET_LABELS |
| theme | Theme | Stored in this browser. | Light / Dark / System (next-themes; never sent to the server) |

Saving: \`PUT /api/settings\` with the full current settings merged with the changed field. On success write the response into \`["settings"]\` with \`setQueryData\`; if \`defaultRange\` changed call \`setRange(null)\` so the new default applies now; toast "Settings saved". On error toast the message.

**Data & storage**: description "Everything is stored locally in a single SQLite file. Nothing leaves this machine." Data \`GET /api/system\` → \`{ dbPath, backupsDir, stats: { sales, demoSales, products, customers, categories, demoRows }, backups: { name, size, createdAt }[] }\`. \`ErrorState\` on error; 96px skeleton while loading.

1. Info \`dl\` (label column 160px from \`sm\`):
   - "Database file": mono xs, breaks anywhere, \`data-testid="db-path"\`.
   - "Backups folder".
   - "Records": "<sales> sales · <products> products · <customers> customers · <categories> categories".
2. Backups (h3 "Backups"):
   - "Uses SQLite's online backup API, so it is safe while the app is running."
   - "Back up now" (outline, DatabaseBackup) → \`POST /api/system/backup\`.
   - Up to 5 backups: mono name, "<round(size / 1024)> KB · <dateTime(createdAt)>".
   - "To restore, stop the app and run \`pnpm db:restore <backup-file>\`. The current database is saved to \`backups/pre-restore\` first."
3. Demo data (h3 "Demo data"):
   - With demo rows: "<demoSales> demo sales are loaded. Demo rows are tagged and can be removed without touching your own records." and "Remove demo data" (Trash2, opens a confirm).
   - Without: "Load about a year of sample sales to explore the dashboard. Demo rows are tagged and removable." and "Load demo data" (Sparkles) → \`POST /api/system/demo { action: "seed" }\`.
4. Danger zone (\`border-destructive/30\`, 16px padding, h3 "Erase all data" in destructive colour): "Deletes every sale, product, customer and category. Settings are kept. Back up first." and destructive button "Erase all data".

All four actions disable their buttons while running. On success they invalidate data, close the confirm, clear the typed phrase and toast:
- Backup: "Backup created: <file>"
- Seed: "Demo data added"
- Remove: "Demo data removed"
- Erase: "All data erased"

Errors toast the message.

Remove demo confirmation: title "Remove demo data?"; description "All demo sales are deleted. Demo products or customers used by your own sales are kept."; action "Remove demo data"; request \`POST /api/system/demo { action: "remove" }\`.

Erase confirmation:
- Title "Erase all data?"; description "This permanently deletes all sales, products, customers and categories. It cannot be undone."; action "Erase everything".
- Children: field labelled "Type DELETE ALL DATA to confirm" (phrase in mono), input id \`erase-confirm\`, \`autoComplete="off"\`.
- The action stays disabled until the input equals \`DELETE ALL DATA\` exactly.
- Request \`POST /api/system/erase { confirm: phrase }\`. Closing the dialog clears the phrase.

**Export**: description "CSV files open in any spreadsheet. Filtered exports are available on the Sales and Analytics pages." Outline download links "All sales" (\`/api/sales/export\`), "Products" (\`/api/products/export\`), "Customers" (\`/api/customers/export\`).

## 14. Responsive behaviour

Tailwind default breakpoints: \`sm\` 640px, \`md\` 768px, \`lg\` 1024px, \`xl\` 1280px.

| Width | Behaviour |
| --- | --- |
| below 640px | Date picker in a full-width bar under the header (Overview, Analytics); "Add sale" icon-only; sales table shows a mobile "Add" button; toolbars wrap with full-width search; forms single-column. |
| below 768px | Sidebar is an off-canvas sheet that closes on navigation; main padding 16px; Recent sales hides Customer; products hide SKU; customers hide Email. |
| 768px+ | Persistent sidebar, collapsible to a 3rem icon rail (state in \`useUiStore.sidebarOpen\`). |
| below 1024px | Search is an icon button; Payment (recent sales), Cost (products) and Phone (customers) hidden; dashboard grids single-column. |
| 1024px+ | Overview rows use 3 columns (main chart spans 2); analytics grid 2 columns. |
| 1280px+ | All 5 KPI cards in one row. |

Tables sit inside cards and scroll horizontally when they do not fit.

## 15. Accessibility requirements

| ID | Requirement |
| --- | --- |
| A11Y-1 | Exactly one \`main\` (\`#main\`) and one \`h1\` (top bar) per page; the skip link "Skip to content" is the first focusable element. |
| A11Y-2 | Every icon-only button has an \`aria-label\` (exact labels in §9–§13). Decorative icons, swatches, status dots and sparklines are \`aria-hidden\`. |
| A11Y-3 | Form fields have visible labels (or \`sr-only\` labels where a legend already names them), \`aria-invalid\` on error, and \`FieldError\` with \`role="alert"\`. Item cards use \`role="group"\` named "Item <n>". |
| A11Y-4 | Full keyboard support for menus, selects, sheets, dialogs and the command palette (Cmd/Ctrl+K; Cmd/Ctrl+B toggles the sidebar). Visible focus rings (\`outline-ring/50\` and component \`focus-visible\` rings). |
| A11Y-5 | Charts set Recharts \`accessibilityLayer\`. Chart meaning is also available as text (KPI values, tables, donut legend with percentages). |
| A11Y-6 | Colour is never the only signal: status badges carry text; deltas carry an arrow, a signed percentage and "vs previous period". |
| A11Y-7 | Sale totals \`aria-live="polite"\`; bulk selection bar \`role="status"\`; loading containers \`aria-busy\`; sorted headers \`aria-sort\`; the active nav link \`aria-current="page"\`. |
| A11Y-8 | Reduced motion is honoured globally (§2.3); sparklines never animate. |
| A11Y-9 | Text contrast meets WCAG 2.2 AA in both themes with the token values in §2.2. |

End-to-end checks for these flows are listed in [07-Testing.md](07-Testing.md); requirement coverage is tracked in [10-Requirements-Traceability.md](10-Requirements-Traceability.md).
