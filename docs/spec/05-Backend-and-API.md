# 05 — Backend and API

This document specifies the server side of Ledgerline: the layering of route handlers and services, the error
contract, every service function, the full HTTP API, the CSV exports, the demo data generator and the database CLI
scripts. Money rules, date ranges, metric definitions and form validation live in [03-Domain-Rules.md](03-Domain-Rules.md);
tables, constraints and database opening live in [04-Data-Model.md](04-Data-Model.md). Every message string, status
code, field name and default named here is normative.

## 1. Conventions

| Topic | Rule |
| --- | --- |
| Transport | JSON over HTTP on same-origin Route Handlers under `/api`. No auth (single local user). |
| Ids | Text UUIDs from `crypto.randomUUID()`. |
| Time | Integer epoch milliseconds everywhere (`occurredAt`, `createdAt`, `from`, `to`). Date ranges are half-open `[from, to)`. |
| Money | Integer minor units; field names end in `Minor`. Tax rate in basis points (`taxRateBp`). CSV is the only place money is rendered in major units. |
| Booleans | `isDemo` and `archived` are JSON booleans (stored as 0/1, mapped with Drizzle `mode: "boolean"`). |
| Optional text | Trimmed; an empty string is stored as `null`. |
| Name ordering | `name COLLATE NOCASE` ascending. |
| Runtime | Node.js runtime only (better-sqlite3 is native). No Edge. |
| Unsupported method | Left to Next.js (405). |

## 2. Server architecture

### 2.1 Layers

```text
Route handler   src/app/api/**/route.ts
  withDb(fn)    src/server/api.ts            request-time DB access + error mapping
    service     src/server/services/*.ts     sync, db-first, validates raw input with Zod
      Drizzle + better-sqlite3               one process-wide connection
```

Rules:

1. Route handlers are thin: read params, query and body, call one service, wrap the result in `Response.json` (or a
   CSV response). No SQL and no business rules in handlers.
2. Every handler body runs inside `withDb`. Nothing touches the database at module scope.
3. Services are **synchronous** (better-sqlite3 is sync) and take the database handle as the **first parameter**
   (`fn(db: DB, ...)`). Where a body is involved they receive raw `unknown` input and parse it with the shared Zod schema
   themselves. This makes them testable against a temporary database without HTTP (see [07-Testing.md](07-Testing.md)).
4. Services must **not** import `server-only`, so Vitest and the `tsx` CLI scripts can import them.
5. `src/server/db.ts` is the only module that hands the connection to app code. It imports `server-only` and re-exports
   `getDb` from `src/db/client.ts`. `src/server/api.ts` also imports `server-only`.
6. Writes touching more than one row run in `db.transaction((tx) => ...)` (synchronous callback). A throw inside the
   callback rolls everything back.

### 2.2 Required file layout

| Path | Contents |
| --- | --- |
| `src/server/db.ts` | `import "server-only"`; re-export `getDb`. |
| `src/server/errors.ts` | `AppError`, `notFound`, `conflict`. |
| `src/server/api.ts` | `errorResponse`, `withDb`, `readJson` (server-only). |
| `src/server/services/sales.ts` | Sale writes, reads, listing, export query. |
| `src/server/services/catalog.ts` | Categories and products. |
| `src/server/services/customers.ts` | Customers. |
| `src/server/services/analytics.ts` | KPIs, series, rankings, insights, `dashboard`. |
| `src/server/services/exports.ts` | CSV builders. |
| `src/server/services/maintenance.ts` | Stats, demo removal, erase, backup, backup verification. |
| `src/server/services/settings.ts` | Settings singleton. |
| `src/server/demo-seed.ts` | `hasData`, `seedDemoData`. |
| `src/lib/csv.ts` | `safeCell`, `toCsv`, `csvResponse`, `parseCsv`. |
| `src/app/api/**/route.ts` | One file per path in section 6. |
| `scripts/db-*.ts` | CLI scripts in section 9. |

Shared types: `type DB` comes from `src/db/client.ts`. Row types (`Sale`, `SaleItem`, `Product`, `Category`, `Customer`,
`Settings`) are the `$inferSelect` types of `src/db/schema.ts`. The transaction handle type is
`type Tx = Parameters<Parameters<DB["transaction"]>[0]>[0]`; helpers usable inside and outside a transaction accept
`Tx | DB`.

## 3. Errors and request plumbing

### 3.1 `src/server/errors.ts`

| Export | Signature | Behaviour |
| --- | --- | --- |
| `AppError` | `class AppError extends Error { constructor(readonly status: number, message: string) }` | `name = "AppError"`. |
| `notFound` | `(what: string) => AppError` | 404, message `"<what> not found"` (e.g. `Sale not found`). |
| `conflict` | `(message: string) => AppError` | 409 with the given message. |

### 3.2 `errorResponse(e: unknown): Response`

Checked in this order:

| Thrown value | Status | JSON body | Logged |
| --- | --- | --- | --- |
| `ZodError` | 400 | `{ "error": <first issue message, or "Invalid input">, "issues": [{ "path": <issue path joined with ".">, "message": <issue message> }] }` | no |
| `AppError` | `e.status` | `{ "error": e.message }` | no |
| `DatabaseInitError` (from `src/db/client.ts`) | 503 | `{ "error": e.message, "dbFile": e.dbFile }` | `console.error` |
| anything else | 500 | `{ "error": "Unexpected server error" }` | `console.error` |

Issue paths: `["items", 0, "quantity"]` becomes `"items.0.quantity"`; a root-level issue has path `""`.
Example: `{ "error": "Add at least one item", "issues": [{ "path": "items", "message": "Add at least one item" }] }`.

`DatabaseInitError` messages (the class is defined with the client, see [04-Data-Model.md](04-Data-Model.md)):

- ``Cannot create the data directory for <file>. Check folder permissions.``
- ``Cannot open SQLite database at <file>: <cause>. If better-sqlite3 failed to load, run `pnpm rebuild better-sqlite3` with the Node version you start the app with.``
- ``Database migration failed for <file>: <cause>``

### 3.3 `withDb(fn: (db: DB) => Response | Promise<Response>): Promise<Response>`

1. `await connection()` from `next/server`. With `cacheComponents` enabled this marks the handler as request-time, so
   `next build` never prerenders it and never opens the database (no `data/` directory may appear after a build).
2. Inside `try`: `return await fn(getDb())`. On any throw: `return errorResponse(e)`. Because `getDb()` runs inside the
   `try`, an initialisation failure becomes a 503 JSON response instead of a crash.

Next 16 pitfall: request-time opt-in is the `connection()` call, not `export const dynamic`:

```ts
import { connection } from "next/server";
export async function GET() {
  await connection(); // excluded from prerendering from here on
  // only now open the database
}
```

### 3.4 `readJson(req: Request): Promise<unknown>`

Returns `await req.json()`. Any parse failure (including an empty body) throws
`AppError(400, "Request body must be valid JSON")`.

### 3.5 Route params

Dynamic segment params are a Promise in Next 16. Type the context with the global `RouteContext` generated by
`next typegen` (the `typecheck` script runs it before `tsc`):

```ts
export const GET = (_req: Request, ctx: RouteContext<"/api/sales/[id]">) =>
  withDb(async (db) => Response.json(getSale(db, (await ctx.params).id)));
```

## 4. Wire schemas used by the API

All schemas live in `src/lib/validation.ts` and are shared with the client. `optionalText(max)` = string, trimmed, at most
`max` characters, may be omitted or `null`; an empty result becomes `null`. `minor` = integer `0..1_000_000_000`.

### 4.1 `SaleInput` (`saleInputSchema`)

| Field | Type / rule | Message |
| --- | --- | --- |
| `occurredAt` | positive integer (ms) | Zod default |
| `orderRef` | `optionalText(64)` | Zod default |
| `customer` | discriminated union on `mode` (below) | |
| `paymentMethod` | `card | cash | bank_transfer | e_wallet | other` | Zod default |
| `status` | `completed | pending | cancelled | refunded` | Zod default |
| `items` | array, 1..50 | `Add at least one item` (min) |
| `discountMinor` | `minor`, and ≤ Σ(quantity × unitPriceMinor) | `Discount cannot exceed the subtotal` (path `discountMinor`) |
| `taxRateBp` | integer 0..10000 | Zod default |
| `notes` | `optionalText(2000)` | Zod default |

`customer` variants:

| `mode` | Fields |
| --- | --- |
| `walkin` | `name`: `optionalText(120)` |
| `existing` | `customerId`: non-empty string |
| `new` | `name`: trimmed 1..120 (`Customer name is required`); `email`: valid email, optional, `null` or `""` (→ `null`); `phone`: `optionalText(40)` |

Item (`saleItemInputSchema`):

| Field | Type / rule |
| --- | --- |
| `productId` | non-empty string, optional or `null` |
| `newProduct` | optional or `null`: `{ name: trimmed 1..120 ("Product name is required"), categoryName: optionalText(60), sku: optionalText(64) }` |
| `quantity` | integer 1..100000 |
| `unitPriceMinor` | `minor` |
| `unitCostMinor` | `minor` or `null` (key required) |
| refinement | exactly one of `productId` / `newProduct` is set, else `Choose an existing product or create a new one` (path `productId`) |

### 4.2 Other bodies

| Schema | Fields |
| --- | --- |
| `productInputSchema` | `name` trimmed 1..120 (`Name is required`); `sku` `optionalText(64)`; `categoryId` non-empty string, or null/omitted (→ `null`); `description` `optionalText(1000)`; `defaultPriceMinor` `minor`; `defaultCostMinor` `minor`, or null/omitted (→ `null`); `archived` optional boolean |
| `categoryInputSchema` | `name` trimmed 1..60 (`Name is required`) |
| `customerRecordSchema` | `name` trimmed 1..120 (`Name is required`); `email` valid email (`Enter a valid email`), optional/`null`/`""` (→ `null`); `phone` `optionalText(40)`; `notes` `optionalText(2000)` |
| `settingsInputSchema` | all required: `currency` ∈ USD, EUR, GBP, CAD, AUD, NZD, SGD, CHF, IDR, INR; `locale` ∈ en-US, en-GB, en-AU, de-DE, fr-FR, es-ES, id-ID, ja-JP; `dateFormat` ∈ `MMM d, yyyy`, `d MMM yyyy`, `yyyy-MM-dd`, `MM/dd/yyyy`, `dd/MM/yyyy`; `defaultRange` ∈ today, 7d, 30d, thisMonth, prevMonth, thisYear (not `custom`) |
| archive body (route-local) | `{ archived: boolean }`, **strict** (unknown keys → 400) |
| bulk delete body (route-local) | `{ ids: string[] }`, each non-empty, 1..1000 entries |
| demo body (route-local) | `{ action: "seed" | "remove" }` |
| erase body (route-local) | `{ confirm: "DELETE ALL DATA" }` (exact literal), message `Type "DELETE ALL DATA" to confirm` |

### 4.3 Query schemas

`salesQuerySchema` with `parseSalesQuery(sp: URLSearchParams): SalesQuery`. Each param is read with `sp.get`; an empty
string counts as absent. `status` and `paymentMethod` are comma-separated lists (empty items dropped).

| Param | Type | Default | Limits |
| --- | --- | --- | --- |
| `q` | string, trimmed | none | ≤ 100 chars |
| `status` | list of statuses | none | each a valid status |
| `paymentMethod` | list of methods | none | each a valid method |
| `productId`, `categoryId` | string | none | |
| `from`, `to` | coerced integer ms | none | half-open |
| `sort` | `date | amount | quantity | status` | `date` | |
| `dir` | `asc | desc` | `desc` | |
| `page` | coerced integer, **0-based** | `0` | ≥ 0 |
| `pageSize` | coerced integer | `25` | 1..100 (UI offers 10, 25, 50, 100) |

`rangeQuerySchema`: `{ from, to }`, both coerced integers. Refinements, in order: `to > from` else `Invalid date range`;
`to − from ≤ 11,000 days` (`1000 × 86_400_000 × 11` ms) else `Date range too large`. Handlers pass `sp.get("from")`
directly, so a missing value coerces to `0` and fails with a 400.

## 5. Service modules

### 5.1 `settings.ts`

| Function | Behaviour |
| --- | --- |
| `getSettings(db): Settings` | Row `id = 1`: `{ id, currency, locale, dateFormat, defaultRange, updatedAt }`. The row always exists (inserted when the database is opened; defaults USD, en-US, `MMM d, yyyy`, 30d). |
| `updateSettings(db, raw): Settings` | Parse `settingsInputSchema`, update row 1 (bumps `updatedAt`), return `getSettings`. |

### 5.2 `catalog.ts`

| Function | Behaviour |
| --- | --- |
| `findOrCreateCategory(tx: Tx | DB, name: string, isDemo = false): Category` | Trim `name`; return the row where `name = ? COLLATE NOCASE`; otherwise insert `{ id, name, isDemo }` and return the inserted row. |
| `listCategories(db)` | Rows `{ id, name, productCount }`; `productCount` = correlated `count(*)` of products (archived included) in the category. Order name NOCASE asc. |
| `createCategory(db, raw): Category` | Parse `categoryInputSchema`. NOCASE match exists → 409 `Category "<name>" already exists`. Else `findOrCreateCategory`. |
| `renameCategory(db, id, raw): Category` | Parse. NOCASE match with a different id → 409 `Category "<name>" already exists` (changing only the case of its own name is allowed). Update; 0 rows → 404 `Category not found`. Return the updated row. Sale lines keep their `category_name` snapshot. |
| `deleteCategory(db, id): void` | `n` = products in the category (archived included). `n > 0` → 409 `This category is used by <n> product(s). Reassign them first.` Delete; 0 rows → 404 `Category not found`. Sale lines referencing it get `category_id = NULL` through the FK (`ON DELETE SET NULL`); the name snapshot remains. |
| `listProducts(db, opts: { q?: string; includeArchived?: boolean } = {}): ProductRow[]` | See below. |
| `createProduct(db, raw): Product` | Parse `productInputSchema`; `assertSkuFree(sku)`; `assertCategory(categoryId)`; insert with `archived = input.archived ?? false`; return the full row. |
| `updateProduct(db, id, raw): Product` | Parse; `assertSkuFree(sku, exceptId = id)`; `assertCategory`; replace all editable fields and set `archived = input.archived ?? false`. 0 rows → 404 `Product not found`. Return the full row. Never rewrites sale-line snapshots. |
| `setProductArchived(db, id, archived: boolean): Product` | Update `archived`; 0 rows → 404 `Product not found`; return the full row. |
| `deleteProduct(db, id): void` | Any `sale_items` row with this `product_id` (any status) → 409 `This product appears in past sales. Archive it instead to keep history intact.` Delete; 0 rows → 404 `Product not found`. |

Private helpers: `assertSkuFree(db, sku: string | null, exceptId?: string)` — when `sku` is non-null and another product
has it → 409 `SKU "<sku>" is already used by another product` (SKU match is exact, case-sensitive).
`assertCategory(db, categoryId: string | null)` — a non-null id that does not exist → 404 `Category not found`.

`listProducts` query:

- `FROM products LEFT JOIN categories`.
- Filters (AND): unless `includeArchived`, `archived = 0`; when `q` is non-empty, `name LIKE %q%` OR `sku LIKE %q%`
  (with the LIKE escaping of 5.4, so `%` and `_` match literally).
- Order: `archived` asc (active first), then name NOCASE asc. No limit.
- `ProductRow` = `{ id, name, sku, categoryId, categoryName, description, defaultPriceMinor, defaultCostMinor, archived, isDemo, createdAt, updatedAt, unitsSold, revenueMinor }`.
- `unitsSold` = correlated `coalesce(sum(si.quantity), 0)`, `revenueMinor` = correlated `coalesce(sum(si.line_net_minor), 0)`,
  both over `sale_items si JOIN sales s` where `si.product_id` is the product and `s.status = 'completed'`. All-time.

### 5.3 `customers.ts`

| Function | Behaviour |
| --- | --- |
| `listCustomers(db, opts: { q?: string } = {}): CustomerRow[]` | When `q` non-empty: `name` OR `email` OR `phone` `LIKE %q%` (escaped as in 5.4). Order name NOCASE asc. No limit. |
| `getCustomerDetail(db, id): { customer: CustomerRow; history: HistoryRow[] }` | `customer` = the list row for `id` (404 `Customer not found`). `history` = up to **200** of the customer's sales, all statuses, `occurred_at` desc. |
| `createCustomer(db, raw): Customer` | Parse `customerRecordSchema`, insert, return the full row. Names need not be unique. |
| `updateCustomer(db, id, raw): Customer` | Parse; replace name, email, phone, notes; 0 rows → 404 `Customer not found`; return the full row. Does **not** rewrite `customer_name` snapshots on past sales. |
| `deleteCustomer(db, id): void` | `n` = sales with this `customer_id` (any status). `n > 0` → 409 `This customer has <n> sale(s). Customers with history cannot be deleted.` Delete; 0 rows → 404 `Customer not found`. |

- `CustomerRow` = `{ id, name, email, phone, notes, isDemo, createdAt, orders, revenueMinor, lastSaleAt }` via correlated
  subqueries: `orders` = count of the customer's **completed** sales; `revenueMinor` = `coalesce(sum(net_minor), 0)` of
  completed sales; `lastSaleAt` = `max(occurred_at)` over **all** statuses, `null` when none.
- `HistoryRow` = `{ id, orderRef, occurredAt, status, totalMinor, netMinor, paymentMethod }`.
- `Customer` (full row) = `{ id, name, email, phone, notes, isDemo, createdAt, updatedAt }`.

### 5.4 `sales.ts`

Types:

- `Sale` = `{ id, orderRef, occurredAt, customerId, customerName, paymentMethod, status, subtotalMinor, discountMinor, netMinor, taxRateBp, taxMinor, totalMinor, costMinor, notes, isDemo, createdAt, updatedAt }`.
- `SaleItem` = `{ id, saleId, productId, categoryId, productName, categoryName, quantity, unitPriceMinor, unitCostMinor, lineSubtotalMinor, lineNetMinor, lineCostMinor, position }`.
- `SaleWithItems = Sale & { items: SaleItem[] }`.
- `SaleRow` = `{ id, orderRef, occurredAt, customerId, customerName, paymentMethod, status, netMinor, totalMinor, itemCount, units, firstProduct, isDemo }`.

| Function | Behaviour |
| --- | --- |
| `createSale(db, raw, opts: { isDemo?: boolean } = {}): SaleWithItems` | Parse `saleInputSchema`. One transaction: if `orderRef` given, `assertRefFree`; `writeSale(isNew = true)` with `orderRef ?? newOrderRef(tx)` and `isDemo = opts.isDemo ?? false`. Return `getSale`. |
| `updateSale(db, id, raw): SaleWithItems` | Parse. One transaction: load the sale (404 `Sale not found`); `ref = input.orderRef ?? existing.orderRef`; `assertRefFree(ref, exceptId = id)`; `writeSale(isNew = false)` keeping the stored `isDemo` and `createdAt`. Return `getSale`. |
| `deleteSales(db, ids: string[]): number` | Empty array → 0. Else one transaction `DELETE FROM sales WHERE id IN (...)`; return rows changed. Items go by `ON DELETE CASCADE`. |
| `getSale(db, id): SaleWithItems` | 404 `Sale not found`; items ordered by `position` asc. |
| `salesWhere(q): SQL | undefined` | Filter builder shared by list and export (below). Takes the query without sort, dir, page, pageSize. |
| `listSales(db, q: SalesQuery): { rows: SaleRow[]; total: number }` | One page plus the total for the same WHERE. |
| `listSalesForExport(db, q: SalesQuery)` | All matching rows (no paging) with the export columns of 7.2. |

Write algorithm `writeSale(tx, id, input, orderRef, isDemo, isNew)`:

1. **Resolve the customer.**
   - `walkin` → `customerId = null`, `customerName` = the given name or `null` (no placeholder is stored).
   - `existing` → load by id; missing → `AppError(400, "Selected customer no longer exists")`. Snapshot its current name.
   - `new` → insert `{ name, email ?? null, phone ?? null, isDemo }` and use it. Every write in `new` mode creates a
     fresh customer, on update too.
2. **Resolve each item**, in input order.
   - `productId` → load the product `LEFT JOIN categories`; missing → `AppError(400, "Selected product no longer exists")`.
     Archived products are allowed. Snapshot `productName`, `categoryId`, `categoryName` from the current rows.
   - `newProduct` → category = `findOrCreateCategory(tx, categoryName, isDemo)` when `categoryName` is non-null. Insert a
     product with `sku` (or `null`), `defaultPriceMinor = unitPriceMinor`, `defaultCostMinor = unitCostMinor`, `isDemo`.
     A taken SKU → 409 `SKU "<sku>" is already used by another product`. Snapshot its name and category.
   - Unit price and cost always come from the input line, never from product defaults.
3. **Totals** = `computeSaleTotals(items, discountMinor, taxRateBp)` (see [03-Domain-Rules.md](03-Domain-Rules.md)).
4. **Sale row** values: `orderRef, occurredAt, customerId, customerName, paymentMethod, status, subtotalMinor, discountMinor, netMinor, taxRateBp, taxMinor, totalMinor, costMinor, notes`.
   New → insert with `id` and `isDemo`. Existing → update the row (bumps `updatedAt`) and delete all its items.
5. **Items**: one row per line with a fresh id, the snapshots, `quantity`, `unitPriceMinor`, `unitCostMinor`,
   `lineSubtotalMinor / lineNetMinor / lineCostMinor` from `totals.lines[i]`, `position = i`.

Helpers: `newOrderRef(tx)` = `ORD-` + 8 uppercase hex characters from 4 random bytes, retried until unused.
`assertRefFree(tx, ref, exceptId?)` → 409 `Order reference "<ref>" is already used` when another sale owns the ref.
A failure anywhere in the transaction, including a database constraint, leaves no customer, product, category, sale or
item behind.

Listing SQL strategy:

- **Per-row aggregates are correlated subqueries**, never `JOIN sale_items ... GROUP BY`, so `LIMIT/OFFSET` and
  `count(*)` count sales, not lines:
  - `itemCount` = `count(*)` of the sale's items;
  - `units` = `coalesce(sum(quantity), 0)` of the sale's items;
  - `firstProduct` = `product_name` of the item with the lowest `position`;
  - export only: `products` = `group_concat(product_name || ' x' || quantity, '; ')` over the sale's items.
- `salesWhere` conditions (AND of those present):

| Input | Condition |
| --- | --- |
| `from` | `occurred_at >= from` |
| `to` | `occurred_at < to` |
| `status` (non-empty) | `status IN (...)` |
| `paymentMethod` (non-empty) | `payment_method IN (...)` |
| `productId` | `EXISTS (SELECT 1 FROM sale_items si WHERE si.sale_id = sales.id AND si.product_id = ?)` |
| `categoryId` | the same `EXISTS` on `si.category_id` |
| `q` | `order_ref LIKE :p ESCAPE '\'` OR `customer_name LIKE :p ESCAPE '\'` OR `EXISTS` an item with `product_name LIKE :p ESCAPE '\'` |

  LIKE escaping: `:p = "%" + escaped(q) + "%"`, where `escaped` prefixes each backslash, `%` and `_` with a backslash.
  Matching is ASCII case-insensitive (SQLite LIKE). Searching `%` matches only values containing a literal percent sign.
- Sort mapping: `date` → `occurred_at`, `amount` → `total_minor`, `quantity` → the `units` subquery, `status` → `status` text.
  List order: `<sort> <dir>`, then `occurred_at DESC`, then `id DESC` (stable paging). Export order: `<sort> <dir>`, then
  `occurred_at DESC`.
- Paging: `LIMIT pageSize OFFSET page × pageSize`. `total` = `count(*)` with the same WHERE.

### 5.5 `analytics.ts`

Metric definitions are in [03-Domain-Rules.md](03-Domain-Rules.md). Every function takes `(db, r: DateRange)` and filters
`s.occurred_at >= r.from AND s.occurred_at < r.to`. Local day expression:
`date(s.occurred_at / 1000, 'unixepoch', 'localtime')`.

| Function | Output and SQL |
| --- | --- |
| `kpis(db, r): Kpis` | One conditional-aggregate query over `sales` plus one units query over completed items. `Kpis = { revenueMinor, salesCount, aovMinor, units, costMinor, costedCount, profitMinor, pendingMinor, pendingCount, refundedMinor, refundedCount, cancelledCount }`. `aovMinor = round(revenue / count)`, or 0. `costMinor` = Σ non-null `cost_minor` of completed. `costedCount` = completed with `cost_minor IS NOT NULL`. `profitMinor = revenue − cost` only when `salesCount > 0` and `costedCount = salesCount`, else `null`. Pending and refunded amounts are Σ `net_minor` of those statuses. |
| `series(db, r, bucket): SeriesPoint[]` | `SeriesPoint = { key, revenueMinor, orders, units, aovMinor }`. SQL aggregates completed sales per local day (revenue, orders) and completed item units per local day; each day rolls into `bucketKeyForDay(day, bucket)`. Start from `bucketKeys(r, bucket)` so every bucket exists with zeros; a day whose key is not in the list is ignored. `aovMinor = round(revenue / orders)` or 0. Key order. |
| `productRanking(db, r, limit = 10)` | Rows `{ productId, name, revenueMinor, units, orders }` from completed items, grouped by `coalesce(product_id, product_name)`; `name` = a `product_name` snapshot of the group; `revenueMinor = Σ line_net_minor`, `units = Σ quantity`, `orders = count(DISTINCT sale)`. Order revenue desc, units desc. |
| `categoryBreakdown(db, r)` | Rows `{ name, revenueMinor, units }` grouped by `coalesce(category_name, 'Uncategorized')` (the snapshot, so renamed or deleted categories keep history). Revenue desc. No limit. |
| `customerRanking(db, r, limit = 10)` | Rows `{ customerId, name, revenueMinor, orders }` from completed sales `LEFT JOIN customers`, grouped by `coalesce(customer_id, '__walkin__')`. `name` = `'Walk-in customers'` when `customer_id` is null, else the customer's **current** name (fallback: the snapshot). Revenue desc. |
| `paymentBreakdown(db, r)` | Rows `{ method, revenueMinor, orders }` from completed sales grouped by `payment_method`, revenue desc. Only methods present. |
| `statusBreakdown(db, r)` | Rows `{ status, netMinor, orders }` over **all** statuses, one per status present. |
| `weekdayBreakdown(db, r)` | Exactly 7 rows `{ dow, revenueMinor, orders }`, `dow` 0 = Sunday … 6 = Saturday, from completed sales via `strftime('%w', occurred_at / 1000, 'unixepoch', 'localtime')`; missing days zero-filled. |
| `dashboard(db, range, bucketOverride?: Bucket): Dashboard` | See below. |

`dashboard` algorithm:

1. `previousRange = previousRange(range)`; `bucket = bucketOverride ?? autoBucket(range)`.
2. `kpis` for both ranges; `series` for both ranges with the same bucket; `daily` = the current series when the bucket is
   `day`, else `series(range, "day")`.
3. Rankings and breakdowns for the current range.
4. Insights, constant `MIN_SALES_FOR_INSIGHTS = 3`: empty unless `current.salesCount >= 3`; otherwise in this order:

| id | title | Condition | value | detail | tone / money |
| --- | --- | --- | --- | --- | --- |
| `best-product` | `Best-selling product` | a product row exists | top product name | `"<units> units"` | `money: false` |
| `top-category` | `Top category` | a category row exists | top category name | `"<share>% of revenue"`, share = `round(cat / revenue × 100)`, 0 when revenue is 0 | — |
| `best-day` | `Best day` | top `daily` revenue > 0 (earliest wins ties) | day key `yyyy-MM-dd` | revenue minor as a string | `money: true` |
| `trend` | `Trend within period` | `daily.length >= 4` and change not null | `"Up N%"` or `"Down N%"` (N = absolute value, 0 decimals; change ≥ 0 is Up) | `Second half vs first half` | positive if > 2, negative if < −2, else neutral |
| `vs-prev` | `Vs previous period` | always | `"+12.5%"` / `"-3.0%"` (1 decimal, `+` when ≥ 0), or `No prior sales` when the change is null | `Revenue change` | positive if > 0, negative if < 0, else neutral |

   Trend: `half = floor(len / 2)`; first = Σ revenue of the first `half` days; second = Σ of the last `half` days (the
   middle day is skipped when `len` is odd); change = `percentChange(second, first)`.
   `Insight = { id, title, value: string | number, detail?, tone?: "positive" | "negative" | "neutral", money?: boolean }`.

`Dashboard` JSON shape:

```json
{
  "range": { "from": 0, "to": 0 },
  "previousRange": { "from": 0, "to": 0 },
  "bucket": "day | week | month",
  "kpis": { "current": "Kpis", "previous": "Kpis" },
  "series": ["SeriesPoint"], "previousSeries": ["SeriesPoint"], "daily": ["SeriesPoint"],
  "products": ["productRanking row"], "categories": ["categoryBreakdown row"],
  "customers": ["customerRanking row"], "payments": ["paymentBreakdown row"],
  "statuses": ["statusBreakdown row"], "weekdays": ["weekday row x7"],
  "insights": ["Insight"]
}
```

### 5.6 `exports.ts`

| Function | Behaviour |
| --- | --- |
| `salesCsv(db, q: SalesQuery, currency: string): string` | Section 7.2. |
| `REPORTS` | `["timeseries", "products", "categories", "customers", "payments"] as const`, with `type ReportName`. |
| `reportCsv(db, report: ReportName, range: DateRange, currency: string): string` | Calls `dashboard(db, range)` (automatic bucket) and renders one table (section 7.3). |
| `productsCsv(db, currency: string): string` | Section 7.4. |
| `customersCsv(db, currency: string): string` | Section 7.5. |

### 5.7 `maintenance.ts`

| Function | Behaviour |
| --- | --- |
| `stats(db)` | `{ sales, demoSales, products, customers, categories, demoRows }`; `demoRows` = demo sales + demo products + demo customers + demo categories. Products include archived. |
| `removeDemoData(db)` | One transaction, in order: (1) delete sales with `is_demo = 1` (items cascade); (2) set `is_demo = 0` on demo products still referenced by any `sale_items.product_id`; (3) same for demo customers referenced by any `sales.customer_id`; (4) delete remaining demo products; (5) delete remaining demo customers; (6) set `is_demo = 0` on demo categories still referenced by any `products.category_id`; (7) delete remaining demo categories. Returns `{ salesRemoved, productsRemoved, customersRemoved, categoriesRemoved }`. Real rows are never deleted. |
| `eraseAllData(db): void` | One transaction deleting every row of `sale_items`, `sales`, `products`, `customers`, `categories`, in that order. Settings are kept. |
| `backupTo(db, dir: string): Promise<string>` | `mkdir -p dir`; file = `<dir>/sales-<stamp>.db`, stamp = `new Date().toISOString()` with every `:` and `.` replaced by `-` (e.g. `sales-2026-03-15T09-30-00-123Z.db`); `await db.$client.backup(file)` (SQLite online backup API: consistent while the app runs, includes WAL content). Returns the full path. |
| `listBackups(dir: string)` | `[]` when the directory is missing. Else the directory's own entries (not recursive) ending in `.db`, as `{ name, size, createdAt: mtimeMs }`, newest first. |
| `verifyBackupFile(file: string): void` | Open read-only with `fileMustExist: true`. `PRAGMA integrity_check` must be `ok`, else throw `Integrity check failed: <result>`. Tables `sales, sale_items, products, customers, categories, settings` must exist, else throw `Not a sales dashboard database (missing table <name>)`. Always close the handle. |

### 5.8 `demo-seed.ts`

| Function | Behaviour |
| --- | --- |
| `hasData(db): boolean` | True when at least one sale **or** one product exists. |
| `seedDemoData(db, opts: { now?: Date; days?: number; seed?: number } = {}): { sales: number; products: number; customers: number }` | Section 8. One transaction. Never modifies existing rows. |

## 6. HTTP API contract

Every endpoint goes through `withDb` and uses the error shapes of 3.2. Bodies refer to section 4. Success is 200 unless
stated.

### 6.1 Sales

| ID | Method and path | Input | Success | Errors |
| --- | --- | --- | --- | --- |
| API-01 | `GET /api/sales` | query per 4.3 | `{ "rows": SaleRow[], "total": number }` | 400 invalid query |
| API-02 | `POST /api/sales` | `SaleInput` | **201** `SaleWithItems` | 400 JSON or validation; 400 `Selected customer no longer exists`; 400 `Selected product no longer exists`; 409 `Order reference "<ref>" is already used`; 409 SKU |
| API-03 | `DELETE /api/sales` | `{ ids }` (1..1000) | `{ "deleted": n }` | 400 validation; 404 `No matching sales` when nothing was deleted (unknown ids are ignored when at least one matches) |
| API-04 | `GET /api/sales/[id]` | — | `SaleWithItems` | 404 `Sale not found` |
| API-05 | `PUT /api/sales/[id]` | `SaleInput` (blank `orderRef` keeps the current ref) | `SaleWithItems` | as API-02, plus 404 `Sale not found` |
| API-06 | `DELETE /api/sales/[id]` | — | `{ "deleted": 1 }` | 404 `Sale not found` |
| API-07 | `GET /api/sales/export` | query per 4.3 (page and pageSize ignored) | CSV `sales-<yyyy-MM-dd>.csv` (server-local today) | 400 invalid query |

### 6.2 Analytics and reports

| ID | Method and path | Input | Success | Errors |
| --- | --- | --- | --- | --- |
| API-08 | `GET /api/dashboard` | `from`, `to` (ms, required); `bucket` optional `day | week | month` | `Dashboard` | 400 `Invalid date range`, `Date range too large`, invalid bucket |
| API-09 | `GET /api/reports/[report]` | path `report` ∈ `REPORTS`; `from`, `to` | CSV `report-<report>.csv` | 400 unknown report (checked first); 400 range |

### 6.3 Products and categories

| ID | Method and path | Input | Success | Errors |
| --- | --- | --- | --- | --- |
| API-10 | `GET /api/products` | `q`; `archived=1` includes archived (any other value excludes) | `ProductRow[]` | — |
| API-11 | `POST /api/products` | `productInputSchema` | **201** `Product` | 400; 404 `Category not found`; 409 SKU |
| API-12 | `PUT /api/products/[id]` | `productInputSchema` | `Product` | 400; 404 `Product not found` / `Category not found`; 409 SKU |
| API-13 | `PATCH /api/products/[id]` | `{ archived: boolean }` (strict) | `Product` | 400; 404 `Product not found` |
| API-14 | `DELETE /api/products/[id]` | — | `{ "deleted": 1 }` | 404; 409 `This product appears in past sales. Archive it instead to keep history intact.` |
| API-15 | `GET /api/products/export` | — | CSV `products.csv` | — |
| API-16 | `GET /api/categories` | — | `[{ id, name, productCount }]` | — |
| API-17 | `POST /api/categories` | `{ name }` | **201** `Category` = `{ id, name, isDemo, createdAt, updatedAt }` | 400; 409 `Category "<name>" already exists` |
| API-18 | `PUT /api/categories/[id]` | `{ name }` | `Category` | 400; 404 `Category not found`; 409 duplicate |
| API-19 | `DELETE /api/categories/[id]` | — | `{ "deleted": 1 }` | 404; 409 `This category is used by <n> product(s). Reassign them first.` |

### 6.4 Customers

| ID | Method and path | Input | Success | Errors |
| --- | --- | --- | --- | --- |
| API-20 | `GET /api/customers` | `q` | `CustomerRow[]` | — |
| API-21 | `POST /api/customers` | `customerRecordSchema` | **201** `Customer` | 400 |
| API-22 | `GET /api/customers/[id]` | — | `{ "customer": CustomerRow, "history": HistoryRow[] }` | 404 `Customer not found` |
| API-23 | `PUT /api/customers/[id]` | `customerRecordSchema` | `Customer` | 400; 404 |
| API-24 | `DELETE /api/customers/[id]` | — | `{ "deleted": 1 }` | 404; 409 `This customer has <n> sale(s). Customers with history cannot be deleted.` |
| API-25 | `GET /api/customers/export` | — | CSV `customers.csv` | — |

### 6.5 Settings and system

| ID | Method and path | Input | Success | Errors |
| --- | --- | --- | --- | --- |
| API-26 | `GET /api/settings` | — | `Settings` | — |
| API-27 | `PUT /api/settings` | `settingsInputSchema` (all four fields) | `Settings` | 400 |
| API-28 | `GET /api/system` | — | `{ "dbPath": string, "backupsDir": string, "stats": Stats, "backups": [{ "name", "size", "createdAt" }] }` (absolute paths) | — |
| API-29 | `POST /api/system/backup` | no body | **201** `{ "file": "<basename>" }`, written to `backupsDir` | 500 on I/O failure |
| API-30 | `POST /api/system/demo` | `{ action }` | `seed` → `{ sales, products, customers }`; `remove` → `{ salesRemoved, productsRemoved, customersRemoved, categoriesRemoved }` | 400 |
| API-31 | `POST /api/system/erase` | `{ confirm: "DELETE ALL DATA" }` | `{ "ok": true }` | 400 `Type "DELETE ALL DATA" to confirm` |

Notes:

- API-30 `seed` has no `hasData` guard: it is an explicit user action and every row it adds is tagged `is_demo`. The
  Settings page offers "Load demo data" only while `stats.demoRows = 0` and "Remove demo data" otherwise.
- `backupsDir` = `<directory of the database file>/backups` (see [04-Data-Model.md](04-Data-Model.md)).
- Restore is CLI-only (section 9). There is no restore endpoint.

## 7. CSV exports

### 7.1 Format (`src/lib/csv.ts`)

| Function | Rule |
| --- | --- |
| `safeCell(v: unknown): string | number` | `null`/`undefined` → `""`; numbers pass through; strings starting with `=`, `+`, `-`, `@`, tab or CR get a leading `'` (formula-injection guard). |
| `toCsv(headers: string[], rows: unknown[][]): string` | UTF-8 BOM (`U+FEFF`) + papaparse `unparse({ fields: headers, data })` with `newline: "\r\n"`; every cell passes through `safeCell`. Papaparse quotes cells containing commas, quotes or line breaks. |
| `csvResponse(filename: string, csv: string): Response` | Headers `Content-Type: text/csv; charset=utf-8`, `Content-Disposition: attachment; filename="<name>"` (characters outside `[A-Za-z0-9_.-]` replaced with `_`), `Cache-Control: no-store`. |
| `parseCsv(text: string): Record<string, string>[]` | Strip a leading BOM; papaparse with `header: true, skipEmptyLines: true`. Used by tests to round-trip exports. |

Value formatting: money → `toMajorString(minor)` (e.g. `1234.50`; no grouping, no symbol; `null` → empty cell);
timestamps → `yyyy-MM-dd HH:mm` in server local time. `<CUR>` below is the settings currency code (e.g. `USD`).

### 7.2 Sales (`sales-<yyyy-MM-dd>.csv`)

Columns, in order: `Order`, `Date`, `Status` (label: Completed, Pending, Cancelled, Refunded), `Payment method` (label:
Card, Cash, Bank transfer, E-wallet, Other), `Customer` (snapshot name, or `Walk-in` when null), `Items` (e.g.
`Ceramic Mug x2; Earl Grey 100g x1`), `Units`, `Subtotal (<CUR>)`, `Discount (<CUR>)`, `Net (<CUR>)`, `Tax (<CUR>)`,
`Total (<CUR>)`, `Cost (<CUR>)`, `Notes`. Rows: every sale matching the filters, in list sort order.

### 7.3 Reports (`report-<name>.csv`)

| Report | Columns | Rows |
| --- | --- | --- |
| `timeseries` | `Period`, `Period start`, `Revenue (<CUR>)`, `Sales`, `Units`, `AOV (<CUR>)` | one per bucket (automatic bucket, zero-filled). `Period` = `MMM d` (day), `Wk of MMM d` (week), `MMM yyyy` (month); `Period start` = bucket key |
| `products` | `Product`, `Revenue (<CUR>)`, `Units`, `Sales` | top 10 products |
| `categories` | `Category`, `Revenue (<CUR>)`, `Units`, `Share %` | all categories; share = `cat / revenue × 100` with 1 decimal, `0` when revenue is 0 |
| `customers` | `Customer`, `Revenue (<CUR>)`, `Sales` | top 10, walk-ins as `Walk-in customers` |
| `payments` | `Payment method`, `Revenue (<CUR>)`, `Sales` | methods present, as labels |

### 7.4 Products (`products.csv`)

`Name`, `SKU`, `Category`, `Default price (<CUR>)`, `Default cost (<CUR>)`, `Status` (`Active` / `Archived`), `Units sold`,
`Revenue (<CUR>)`. All products including archived, in `listProducts` order.

### 7.5 Customers (`customers.csv`)

`Name`, `Email`, `Phone`, `Completed sales`, `Revenue (<CUR>)`, `Customer since` (`yyyy-MM-dd` of `createdAt`). All
customers, in name order.

## 8. Demo data

`seedDemoData` generates about 13 months of believable coffee-shop sales. Output must be deterministic for a given
`seed`, `now` and `days` (row ids aside), so tests can pass a fixed `now` and a short `days`.

### 8.1 Parameters

| Option | Default |
| --- | --- |
| `seed` | `42` |
| `now` | `new Date()` |
| `days` | `400` |

PRNG: **mulberry32**, the standard public-domain 32-bit generator (increment constant `0x6D2B79F5`), returning floats in
`[0, 1)`. Below, `r()` is one draw and `pick(list)` = `list[floor(r() × list.length)]`. Draws happen exactly in the
order of the numbered steps.

### 8.2 Catalog (minor units)

| Category | Product | SKU | Price | Cost |
| --- | --- | --- | --- | --- |
| Coffee | House Blend Beans 1kg | COF-HB1 | 2800 | 1350 |
| Coffee | Single Origin Ethiopia 250g | COF-ETH | 1650 | 780 |
| Coffee | Decaf Colombia 500g | COF-DCF | 1900 | 920 |
| Coffee | Cold Brew Concentrate | COF-CBC | 1200 | 480 |
| Equipment | Pour-over Kettle | EQP-KTL | 6900 | 3400 |
| Equipment | Burr Grinder | EQP-GRD | 14900 | 8200 |
| Equipment | French Press 1L | EQP-FPR | 3900 | 1700 |
| Equipment | Digital Scale | EQP-SCL | 2900 | 1250 |
| Tea | Sencha Green 100g | TEA-SEN | 1400 | 560 |
| Tea | Earl Grey 100g | TEA-EGR | 1100 | 420 |
| Tea | Chamomile Blend | TEA-CHM | 950 | null |
| Merch | Ceramic Mug | MER-MUG | 1800 | 650 |
| Merch | Canvas Tote | MER-TOT | 2200 | 800 |
| Merch | Gift Card | MER-GFT | 5000 | null |
| Workshops | Home Barista Class | WKS-BAR | 8500 | 2500 |
| Workshops | Latte Art Session | WKS-LAT | 6500 | 2000 |

The two products without cost make some sales cost-unknown, which exercises the "profit unavailable" path.

Customers (20, in this order): Amelia Hart, Noah Patel, Sofia Rossi, Liam Chen, Maya Johnson, Ethan Brooks, Zara Ahmed,
Lucas Martin, Chloe Nguyen, Oliver Smith, Isla Murphy, Mateo Garcia, Hana Sato, Felix Wagner, Aria Kapoor, Leo Dubois,
Nora Lindqvist, Kai Tanaka, Ruby Evans, Omar Haddad.

### 8.3 Algorithm (one transaction)

1. Load existing SKUs and existing categories (keyed by lower-cased name).
2. For each catalog category in table order: reuse an existing category with the same name (case-insensitive; it stays
   non-demo), else insert it with `is_demo = 1`. For each of its products: always insert a new product with
   `is_demo = 1`, price and cost as defaults, and its SKU unless that SKU already exists (then `sku = null`).
3. Insert the 20 customers with `is_demo = 1`, email = lower-cased name with whitespace runs replaced by `.`, plus
   `@example.com` (e.g. `amelia.hart@example.com`), and `created_at = now − (days + 5) days`.
4. `today` = local start of day of `now`. For `d` from `days` down to `0`: `day = today − d` calendar days, and
   - growth = `1 + (days − d) / days` (gentle upward trend, ×1 to ×2);
   - weekend = `1.35` on Saturday and Sunday, else `1`;
   - seasonal = `1 + 0.25 × sin(2π × month / 12)`, month 1..12;
   - number of sales `n = max(0, round((1.2 + r() × 2.6) × growth × weekend × seasonal))`.
5. For each of the `n` sales:
   1. time = `day + (8 + r() × 11) hours` (08:00–19:00 local). Later than `now` → skip this sale (no further draws).
   2. Line count: `r() < 0.7` → 1; else `r() < 0.8` → 2; else 3.
   3. Each line: product = `pick(products)` (the 16 just inserted, in table order); quantity = `r() < 0.75` ? 1 :
      `1 + floor(r() × 4)`; unit price and cost = the product's defaults.
   4. Discount: `r() < 0.15` → `round(subtotal × (r() < 0.5 ? 0.10 : 0.05))`, else 0.
   5. Tax rate `1000` bp (10%). Totals via `computeSaleTotals`.
   6. Status from one `r()`: `< 0.86` completed, `< 0.92` pending, `< 0.96` refunded, else cancelled.
   7. Walk-in when `r() < 0.35` (customer and name null); else `pick(customers)` with its name snapshot.
   8. Payment = `pick(["card", "card", "card", "cash", "e_wallet", "bank_transfer"])` (card about 50%; `other` never).
   9. Insert the sale with `order_ref = "DEMO-"` + counter zero-padded to 5 digits (`DEMO-00001`, counting inserted
      sales), `occurred_at = created_at = round(time)`, `is_demo = 1`, `notes = null`; then its items with snapshots and
      positions.
6. Return `{ sales: <inserted count>, products: 16, customers: 20 }`.

Expected shape with defaults: about 1,600 sales (measured 1,599–1,607; the exact count depends on today's date); about 86% completed, 6% pending, 4% refunded, 4% cancelled;
about 36% walk-in; about 14% discounted; about 1.35 lines per sale. Demo data is never loaded automatically.

## 9. CLI scripts

Run with `tsx` through `package.json` scripts. They import services directly (the reason services avoid `server-only`)
and use the `@/` path alias from `tsconfig.json`, which `tsx` resolves. Each opens the database with
`openDatabase(dbPath())` (create + migrate) and closes it with `db.$client.close()` before exiting.

**Pitfall:** `tsx` runs these files as CommonJS, so top-level `await` fails to compile. Put async work in
`async function main()` and call `main().catch((e) => { console.error(e instanceof Error ? e.message : e); process.exit(1); })`.

| Script | Command | Behaviour | Output | Exit |
| --- | --- | --- | --- | --- |
| `db:migrate` | `tsx scripts/db-migrate.ts` | Open (create, migrate, settings row), close. | `Database ready and migrated: <file>` | 0 |
| `db:seed` | `tsx scripts/db-seed.ts [--force]` | If `hasData` and no `--force`: refuse. Else `seedDemoData`. | refuse, stderr: `Refusing to seed: <file> already contains data. Re-run with --force to add demo rows anyway.`; success: `Seeded demo data into <file>:` followed by the result object | 1 on refuse, else 0 |
| `db:demo:remove` | `tsx scripts/db-demo-remove.ts` | `removeDemoData`. | `Removed demo data:` followed by the count object | 0 |
| `db:backup` | `tsx scripts/db-backup.ts [dir]` | `backupTo(db, dir ?? backupsDir())`. Safe while the app runs. | `Backup written: <path>` | 0; 1 with the error message on failure |
| `db:restore` | `tsx scripts/db-restore.ts <file>` | Steps below. **Stop the app first**: a running server keeps the old file open. | see steps | 0; 1 on usage or any error |
| `db:generate` | `drizzle-kit generate` | Generate a migration after schema changes. | drizzle-kit output | — |

`db:restore` steps:

1. No argument → stderr `Usage: pnpm db:restore <path-to-backup.db>`, exit 1.
2. Resolve the path and run `verifyBackupFile`. On failure print the message, exit 1; nothing has changed.
3. If the target database exists: open it, `backupTo(current, <backupsDir>/pre-restore)`, close, print
   `Current database saved to <file>`.
4. `mkdir -p` the target directory; delete `<target>-wal` and `<target>-shm` if present (a stale WAL would corrupt the
   restored file); copy the backup over the target.
5. Open and close the target once so any newer migrations apply to the restored file.
6. Print `Restored <backup> -> <target>`.

## 10. Acceptance checks

- `POST /api/sales` with `{}` returns 400 with `error` and `issues`; with a malformed body it returns 400
  `Request body must be valid JSON`.
- An unknown id on any `[id]` route returns 404 with `<Entity> not found`.
- Archiving, renaming and every other mutation return the JSON documented above (never an empty body).
- Deleting a product, customer or in-use category with history returns 409 with the exact message and changes nothing.
- A sale that fails mid-write leaves zero new rows in any table.
- Sales paging counts sales, not lines: a 3-line sale counts once in `total`.
- `next build` completes without creating the database file.
- `pnpm db:seed` twice: the second run exits 1 with the refusal message.
- Exported CSVs start with a BOM, use CRLF, and prefix `'` to cells starting with `=`, `+`, `-`, `@`.

Traceability is in [10-Requirements-Traceability.md](10-Requirements-Traceability.md); build order is in
[../../BUILD_PLAN.md](../../BUILD_PLAN.md).
