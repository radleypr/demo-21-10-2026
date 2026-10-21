# 10 — Requirements Traceability

This table maps each requirement to the modules that implement it and the test or gate that proves it. Use it as the
checklist for Final acceptance step **F1** in [`BUILD_PLAN.md`](../../BUILD_PLAN.md).

- Requirement IDs (FR-xx, NFR-xx) come from [`01-Product-Spec.md`](01-Product-Spec.md).
- Test IDs (T-U-xx unit, T-I-xx integration, T-E-xx end-to-end) come from [`07-Testing.md`](07-Testing.md).
- **G1–G8** are phase gates and **F1–F9** are Final acceptance steps in [`BUILD_PLAN.md`](../../BUILD_PLAN.md).
- "Browser check" means the Gate 7 checklist: verify by hand in a real browser, or with Playwright or DevTools.
- Module paths are relative to the app root, and they are the layout required by
  [`02-Environment-and-Stack.md`](02-Environment-and-Stack.md).

A requirement counts as done only when every listed proof passes. If a row has only a browser check, record what you
observed in the build report.

## 1. Functional requirements

| IDs | Area | Implemented in | Proved by |
| --- | --- | --- | --- |
| FR-01–FR-08 | Shell, skip link, sidebar, active item, collapse and drawer, tab titles | `src/app/layout.tsx`, `src/components/shell/*`, `src/lib/nav.ts`, `src/lib/ui-store.ts`, page `metadata` | T-E-02, T-E-05; browser check |
| FR-09–FR-13 | Top bar: search button, date picker visibility, theme toggle, Add sale | `src/components/shell/top-bar.tsx`, `theme-toggle.tsx` | T-E-03, T-E-04 (Add sale); browser check |
| FR-14–FR-19 | Global date range: presets, custom range, persistence, minute clock, previous period | `src/lib/date-range.ts`, `src/components/shell/date-range-picker.tsx`, `src/lib/queries.ts` (`useDateRange`) | T-U-13–T-U-17, T-I-20, T-I-23 |
| FR-20–FR-23 | Command palette (⌘K / Ctrl+K), sale search, actions | `src/components/shell/command-menu.tsx` | browser check |
| FR-24–FR-29 | KPI cards, change badges, sparklines, net profit, pending/refunded note | `src/server/services/analytics.ts` (`kpis`, `series`), `src/components/dashboard/kpi-card.tsx`, `overview.tsx` | T-U-12, T-I-17–T-I-19, T-I-23; F8 |
| FR-30–FR-36 | Overview charts, bucket rule, insights, empty period | `analytics.ts` (`dashboard`, insights), `src/components/dashboard/charts.tsx`, `overview.tsx` | T-U-16, T-U-17, T-I-21–T-I-23, T-E-01 |
| FR-37–FR-39 | Recent sales card and its states | `overview.tsx`, `src/components/sales/sale-actions.tsx` | T-E-04; browser check |
| FR-40–FR-45 | Sale sheet modes, prefill, reset, catalog select, line items | `src/components/sales/sale-dialog.tsx`, `src/lib/ui-store.ts` (`useSaleDialog`) | T-E-04; browser check (edit and duplicate) |
| FR-46–FR-49 | Sale form layout, validation timing and messages, live totals | `sale-dialog.tsx`, `src/lib/validation.ts` (`saleFormSchema`), `src/lib/money.ts` (`computeSaleTotals`) | T-U-06–T-U-10, T-U-18–T-U-28, T-E-03 |
| FR-50–FR-53 | Submit, double-submit guard, server write, toasts, invalidation | `sale-dialog.tsx`, `src/lib/api-client.ts`, `src/server/services/sales.ts`, `src/app/api/sales/**` | T-I-06–T-I-10, T-E-04 |
| FR-54–FR-59 | Sales search, URL `?q=`, filters, clear filters, page reset | `src/components/sales/sales-table.tsx`, `sales.ts` (`salesWhere`, `listSales`), `validation.ts` (`salesQuerySchema`) | T-I-13–T-I-15 |
| FR-60–FR-64 | Sales columns, server-side sort and paging, previous page kept, empty states | `sales-table.tsx`, `listSales` | T-I-16; browser check |
| FR-65–FR-67 | Row selection and bulk delete | `sales-table.tsx`, `src/components/common/confirm.tsx`, `deleteSales`, `DELETE /api/sales` | T-I-11; browser check |
| FR-68–FR-71 | Row actions: view, edit, duplicate, delete | `sale-actions.tsx`, `GET/DELETE /api/sales/[id]` | T-I-08, T-I-11; browser check |
| FR-72 | Sales CSV export matches the current filters and sort | `src/server/services/exports.ts` (`salesCsv`), `src/lib/csv.ts`, `/api/sales/export` | T-U-30, T-U-31, T-I-29; G5 |
| FR-73–FR-80 | Products: list, sort, archive, delete rules, add/edit dialog, empty states | `src/components/catalog/products-view.tsx`, `src/server/services/catalog.ts`, `/api/products/**` | T-I-07, T-I-25, T-I-26; browser check |
| FR-81–FR-85 | Categories: add, rename, case-insensitive unique, delete only when unused | `products-view.tsx`, `catalog.ts`, `/api/categories/**` | T-I-24; browser check |
| FR-86–FR-92 | Customers: list, history sheet, add/edit, delete only without history | `src/components/customers/customers-view.tsx`, `src/server/services/customers.ts`, `/api/customers/**` | T-I-12, T-I-27; browser check |
| FR-93–FR-101 | Analytics: trends, rankings, donuts, weekday, customers, status, report CSVs | `src/components/dashboard/analytics-view.tsx`, `analytics.ts`, `exports.ts` (`reportCsv`), `/api/reports/[report]` | T-I-21, T-I-22, T-I-30; browser check |
| FR-102–FR-108 | Settings preferences: currency, locale, date format, default range, theme | `src/components/settings/*`, `src/server/services/settings.ts`, `/api/settings` | T-I-02 (persistence); browser check |
| FR-109–FR-113 | Data and storage: DB path, backups, demo data, erase all | `src/server/services/maintenance.ts`, `src/server/demo-seed.ts`, `/api/system/**` | T-I-05, T-I-28; G4; F7 |
| FR-114 | Settings export card | `exports.ts` (`productsCsv`, `customersCsv`), export routes | T-I-29; G5 |
| FR-115–FR-118 | Shared error, loading and empty states; invalidation after writes; confirm dialogs | `src/components/common/*`, `src/lib/queries.ts` | T-E-04; browser check |
| FR-119 | Server validation errors return the first field message as a 400 | `src/server/api.ts` (`errorResponse`) | T-I-09; G5 (`POST {}` → 400) |
| FR-120 | Demo rows are tagged wherever they appear | `is_demo` columns, `DEMO-` order refs, UI badges | T-I-28; browser check |

## 2. Non-functional requirements

| ID | Requirement | Implemented in | Proved by |
| --- | --- | --- | --- |
| NFR-01 | Zero-config first run | `src/db/paths.ts`, `src/db/client.ts` (create, migrate, settings row) | T-I-01; G8 (build creates no `data/`); F7 |
| NFR-02 | Persistence across restarts | SQLite file in WAL mode, `getDb` singleton | T-I-02, T-E-04 (reload); F6 |
| NFR-03 | Integrity: server is the source of truth, constraints, transactions, no history-erasing cascades | `src/db/schema.ts`, `drizzle/0000_init.sql`, services | T-I-03, T-I-07, T-I-10, T-I-26, T-I-27; G3 |
| NFR-04 | Privacy: local only, no external network calls | `geist` package fonts, no third-party services | G8 (build without network fonts); browser check (Network tab) |
| NFR-05 | No currency conversion | `formatMoney`, settings currency | T-U-11; browser check (currency help text) |
| NFR-06 | Accessibility (WCAG 2.2 AA intent) | `src/components/**`, 06 A11Y-1–A11Y-9 | T-E-01–T-E-05 (role and label locators); browser check |
| NFR-07 | Responsive from 360 px, checked at 390×844 | shell drawer, responsive layouts | T-E-05; browser check |
| NFR-08 | Light and dark themes, system default, no flash | `next-themes`, tokens in `globals.css` | browser check |
| NFR-09 | Visual quality | design tokens and patterns in 06 | browser check |
| NFR-10 | Performance target (server-side paging and SQL aggregates) | `listSales`, `analytics.ts` | T-I-16; browser check (a target, not a gate) |
| NFR-11 | Feedback: loading, empty and error states, toasts | `src/components/common/*`, Sonner | T-E-01, T-E-03, T-E-04; browser check |
| NFR-12 | Local-time bucketing and ranges | `date-range.ts`, SQL `'localtime'` grouping | T-U-13–T-U-17, T-I-20, T-I-21 |
| NFR-13 | Quality gates | `package.json` scripts | G8; F3, F4, F5 |

## 3. Domain and operations rules

| Rule | Source | Proved by |
| --- | --- | --- |
| Money is integer minor units; parsing has no float drift | [03](03-Domain-Rules.md) | T-U-01–T-U-05, T-U-29 |
| Discount before tax, allocated exactly; tax rounds half-up | [03](03-Domain-Rules.md) | T-U-06–T-U-10 |
| Revenue = completed sales, net of discount, excluding tax; a refund is a full status change | [03](03-Domain-Rules.md) | T-I-17, T-I-18; F8 |
| Half-open ranges, Monday weeks, zero-filled buckets | [03](03-Domain-Rules.md) | T-U-13–T-U-17, T-I-20, T-I-21 |
| CSV: BOM, CRLF, quoting, formula-injection escape, major-unit amounts | [03](03-Domain-Rules.md), [05](05-Backend-and-API.md) | T-U-30, T-U-31, T-I-29, T-I-30; G5 |
| Sale lines snapshot name, category, price and cost | [04](04-Data-Model.md) | T-I-06, T-I-07 |
| DB path resolves from the project root, not the working directory | [04](04-Data-Model.md), [08](08-Operations.md) | F6 |
| Clear error and HTTP 503 when the DB cannot be opened | [04](04-Data-Model.md), [05](05-Backend-and-API.md) | T-I-04 |
| Online, WAL-safe backup; verified restore with a safety copy | [04](04-Data-Model.md), [08](08-Operations.md) | T-I-05; G4 |
| Demo data is explicit, tagged and removable; the CLI refuses to seed a non-empty DB | [05](05-Backend-and-API.md) | T-I-28; G4 (second `db:seed` exits 1) |
| API status codes and JSON shapes (400, 404, 409, 503) | [05](05-Backend-and-API.md) | T-I-09, T-I-24–T-I-27; G5 |

## 4. Coverage check

- Every T-U, T-I and T-E case in 07 appears at least once above.
- Every requirement with only a "browser check" must be listed, with what you observed, in the F9 build report.
