# 07 — Testing

This is the test plan for Ledgerline. Every test listed here must be implemented. Together they are the
acceptance spec: a build is done only when every case passes with the values stated below. Domain rules live in
[03-Domain-Rules.md](03-Domain-Rules.md), the schema in [04-Data-Model.md](04-Data-Model.md), service signatures in
[05-Backend-and-API.md](05-Backend-and-API.md), and UI copy in [06-Frontend-and-Design.md](06-Frontend-and-Design.md).
[10-Requirements-Traceability.md](10-Requirements-Traceability.md) maps requirements to the test IDs below.

## 1. Testing rules

| ID | Rule |
| --- | --- |
| TR-1 | Tests never open `data/sales.db` or anything under `data/`. Unit tests use no database. Integration tests use a fresh temp database per test (section 3). E2E uses `.tmp/e2e/sales.db`, which is deleted at the start of every run. |
| TR-2 | The cases in this document are the acceptance spec. Do not weaken, skip, delete or loosen an assertion to make it pass. Fix the implementation instead. Adding further tests is allowed. |
| TR-3 | Tests call the real service and library functions against real SQLite. Do not mock the database, the services or Zod. |
| TR-4 | Server services must not import the `server-only` package. It throws outside a React Server environment, so Vitest would fail to load the services. |
| TR-5 | Tests that depend on local time build timestamps with local-time constructors (`new Date(y, m - 1, d, h)`), never UTC strings, so they pass in any time zone. |
| TR-6 | When output is long, redirect it to a file (for example `pnpm test > .tmp/vt.log 2>&1`) and read the summary lines. |

## 2. Tooling and configuration

### 2.1 Vitest (`vitest.config.mts`)

Use the `.mts` extension; it avoids Vite's CJS deprecation warning.

| Option | Value |
| --- | --- |
| `resolve.alias` | `@` → absolute path of `src` (resolved from `import.meta.dirname`) |
| `test.environment` | `"node"` |
| `test.include` | `["tests/unit/**/*.test.ts", "tests/integration/**/*.test.ts"]` |

No globals: tests import `describe`, `it`, `expect`, `afterEach` from `vitest`.

### 2.2 Playwright (`playwright.config.ts`)

| Option | Value |
| --- | --- |
| `testDir` | `"tests/e2e"` |
| `fullyParallel` | `false` |
| `workers` | `1` (one shared SQLite file; tests run in order) |
| `retries` | `1` when `process.env.CI` is set, else `0` |
| `reporter` | `"github"` on CI, else `"list"` |
| `expect.timeout` | `20_000` ms (the dev server compiles each route on first hit) |
| `timeout` | `60_000` ms per test |
| `use.baseURL` | `http://localhost:3200` |
| `use.trace` | `"retain-on-failure"` |
| `projects` | one project, `name: "chromium"`, `devices["Desktop Chrome"]` |
| `webServer.command` | delete `.tmp/e2e` recursively (with `force`, via a `node -e` one-liner using `fs.rmSync`), then `next dev -p 3200` |
| `webServer.url` | `http://localhost:3200/api/settings` (readiness probe) |
| `webServer.env` | `SALES_DB_PATH=.tmp/e2e/sales.db`, `NEXT_TELEMETRY_DISABLED=1` |
| `webServer.reuseExistingServer` | `false` |
| `webServer.timeout` | `180_000` ms |

Each run therefore starts from an empty database. The e2e file sets `test.describe.configure({ mode: "serial" })`
because the cases share that database and build on each other (section 6).

Before the first e2e run, install the browser with `pnpm test:e2e:install` (`playwright install chromium`).

### 2.3 Test file layout

| File | Kind | Cases |
| --- | --- | --- |
| `tests/helpers.ts` | shared helpers | — |
| `tests/unit/money.test.ts` | unit | 12 |
| `tests/unit/date-range.test.ts` | unit | 5 |
| `tests/unit/validation.test.ts` | unit | 12 (one `it.each` with 9 rows) |
| `tests/unit/csv.test.ts` | unit | 2 |
| `tests/integration/db.test.ts` | integration | 5 |
| `tests/integration/sales.test.ts` | integration | 11 |
| `tests/integration/analytics.test.ts` | integration | 7 |
| `tests/integration/catalog.test.ts` | integration | 5 |
| `tests/integration/exports.test.ts` | integration | 2 |
| `tests/e2e/app.spec.ts` | e2e | 5 |

Totals: 61 Vitest cases in 9 files (31 unit, 30 integration) and 5 Playwright cases.

## 3. Test helpers (`tests/helpers.ts`)

| Export | Signature | Behaviour |
| --- | --- | --- |
| `tmpDir` | `tmpDir(): string` | Creates a directory with `fs.mkdtempSync(path.join(os.tmpdir(), "sales-dashboard-test-"))` and registers it for cleanup. |
| `testDb` | `testDb(file?: string): DB` | Default `file` is `path.join(tmpDir(), "test.db")`. Calls `openDatabase(file)` (creates directories, enables WAL and foreign keys, applies migrations, inserts default settings) and registers the handle for cleanup. |
| `at` | `at(y: number, m: number, d: number, h = 12): number` | Local-time epoch ms: `new Date(y, m - 1, d, h).getTime()`. Default hour is noon so a date never slips across a day boundary. |
| `sale` | `sale(db: DB, o?: Partial<SaleInput> & { qty?, price?, cost?, product?, category? }): SaleWithItems` | Calls `createSale` with the defaults below; remaining overrides are spread over the input. |

Default input built by `sale()`:

| Field | Default |
| --- | --- |
| `occurredAt` | `at(2026, 3, 10)` |
| `customer` | `{ mode: "walkin" }` |
| `paymentMethod` | `"card"` |
| `status` | `"completed"` |
| `items` | one line: `{ newProduct: { name: product, categoryName: category }, quantity: qty, unitPriceMinor: price, unitCostMinor: cost }` |
| `qty` / `price` / `cost` | `1` / `1000` / `null` |
| `product` / `category` | `"Widget"` / `"Gadgets"` |
| `discountMinor` / `taxRateBp` | `0` / `0` |

Cleanup: a module-level `afterEach` closes every registered DB whose `$client.open` is true, then removes every
registered temp directory with `fs.rmSync(dir, { recursive: true, force: true })`. Both registries are emptied each
time. Raw SQL in tests goes through `db.$client` (the better-sqlite3 handle).

Every `sale()` call creates a new product row, because it always uses `newProduct`. Tests that need repeated sales
of one product create the product first and pass `productId`.

## 4. Unit tests

All unit tests are pure: no database, no network, no real clock (dates are fixed).

### 4.1 `tests/unit/money.test.ts`

Functions under test from `src/lib/money.ts`: `parseMoney`, `toMajorString`, `parsePercentToBp`,
`computeSaleTotals`, `formatMoney`, `percentChange`.

| ID | Title | Inputs | Expected |
| --- | --- | --- | --- |
| T-U-01 | parseMoney parses major-unit strings into integer minor units | `"12"`, `"12.5"`, `"0.07"`, `"1,234.56"` | `1200`, `1250`, `7`, `123456` |
| T-U-02 | parseMoney avoids float drift | `"0.29"`, `"1.15"` | `29`, `115` (naive `0.29 * 100` gives 28.999…) |
| T-U-03 | parseMoney rejects invalid input | each of `""`, `"abc"`, `"1.234"`, `"-5"`, `"1e3"`, `"."` | `null` for every one |
| T-U-04 | toMajorString round-trips | `123456`, `5` | `"1234.56"`, `"0.05"` |
| T-U-05 | parsePercentToBp converts percent strings to basis points | `""`, `"10"`, `"8.25"`, `"x"` | `0`, `1000`, `825`, `null` |
| T-U-06 | computeSaleTotals computes subtotal, net, tax and total | lines `[{ quantity: 3, unitPriceMinor: 999, unitCostMinor: 400 }]`, discount `0`, tax `1000` bp | result contains `subtotalMinor 2997, discountMinor 0, netMinor 2997, taxMinor 300, totalMinor 3297, costMinor 1200` |
| T-U-07 | tax rounds half-up on the net amount | one line price `1005`, cost `null`, discount `0`, tax `500` bp; then price `1010` | `taxMinor` `50` (50.25 rounds down); then `51` (50.5 rounds up) |
| T-U-08 | discount applies before tax and is allocated across lines exactly | lines `1 × 1000` and `2 × 1000`, both cost `null`; discount `100`; tax `1000` bp | `netMinor 2900`, `taxMinor 290`, `totalMinor 3190`; sum of `lines[].netMinor` is `2900`; `lines.map(netMinor)` equals `[967, 1933]` |
| T-U-09 | unknown cost when any line has no cost | lines `1 × 500` cost `100` and `1 × 500` cost `null`, discount `0`, tax `0` | `costMinor` is `null` |
| T-U-10 | rejects a discount larger than the subtotal | one line `1 × 100`, cost `null`, discount `101`, tax `0` | throws `RangeError` |
| T-U-11 | formatMoney uses Intl for currency and locale | `(123456, "USD", "en-US")`; `(123456, "EUR", "de-DE")` | `"$1,234.56"`; matches `/1\.234,56\s€/` (the space may be a non-breaking space) |
| T-U-12 | percentChange returns null when the previous value is zero | `(150, 100)`; `(5, 0)` | `50`; `null` |

### 4.2 `tests/unit/date-range.test.ts`

Functions under test from `src/lib/date-range.ts`: `resolveRange`, `previousRange`, `autoBucket`, `bucketKeys`.
Fixed clock: `now = new Date(2026, 2, 15, 14, 30)` (Sunday 15 March 2026, local time). `day(y, m, d)` below means
local midnight, `new Date(y, m - 1, d).getTime()`. A range is `{ from, to }`, half-open.

| ID | Title | Input | Expected |
| --- | --- | --- | --- |
| T-U-13 | presets resolve to half-open local-day intervals | `resolveRange({ preset }, now)` for each preset | `today` → `day(2026,3,15)`..`day(2026,3,16)`; `7d` → `day(2026,3,9)`..`day(2026,3,16)`; `30d` → `day(2026,2,14)`..`day(2026,3,16)`; `thisMonth` → `day(2026,3,1)`..`day(2026,3,16)`; `prevMonth` → `day(2026,2,1)`..`day(2026,3,1)`; `thisYear` → `day(2026,1,1)`..`day(2026,3,16)` |
| T-U-14 | custom ranges are inclusive days and tolerate reversed input | `{ preset: "custom", customFrom: "2026-01-05", customTo: "2026-01-10" }`; then the same with from and to swapped | both resolve to `day(2026,1,5)`..`day(2026,1,11)` |
| T-U-15 | previousRange is the equal-length window immediately before | `{ from: day(2026,3,9), to: day(2026,3,16) }` | `{ from: day(2026,3,2), to: day(2026,3,9) }` |
| T-U-16 | autoBucket chooses granularity by range length | resolved `30d` preset; `day(2025,12,1)`..`day(2026,3,1)`; resolved custom `2025-01-01`..`2025-12-31` | `"day"`; `"week"`; `"month"` |
| T-U-17 | bucketKeys zero-fills every key in the range | `day(2026,3,1)`..`day(2026,3,4)` with `"day"`; `day(2026,3,1)`..`day(2026,3,10)` with `"week"`; `day(2026,1,20)`..`day(2026,3,2)` with `"month"` | `["2026-03-01","2026-03-02","2026-03-03"]`; `["2026-02-23","2026-03-02","2026-03-09"]` (weeks start Monday; 1 March 2026 is a Sunday); `["2026-01-01","2026-02-01","2026-03-01"]` |

### 4.3 `tests/unit/validation.test.ts`

Schemas under test from `src/lib/validation.ts`: `saleFormSchema` (UI strings), `saleFormToInput` (form to
wire), `saleInputSchema` (server wire schema), type `SaleFormValues`.

Base form `form(patch)` (patch is shallow-merged over these values):

| Field | Value |
| --- | --- |
| `occurredAt` | `"2026-03-10T09:30"` |
| `orderRef`, `customerId`, `customerName`, `customerEmail`, `discount`, `notes` | `""` |
| `customerMode` | `"walkin"` |
| `paymentMethod` / `status` | `"card"` / `"completed"` |
| `items` | `[{ productMode: "new", productId: "", productName: "Mug", categoryName: "Merch", quantity: "2", unitPrice: "12.50", unitCost: "" }]` |
| `taxRate` | `"10"` |

| ID | Title | Steps | Expected |
| --- | --- | --- | --- |
| T-U-18 | accepts a valid form and converts it to minor units | `saleFormSchema.safeParse(form())`; `input = saleFormToInput(form())`; `saleInputSchema.safeParse(input)` | first parse succeeds; `input.items[0]` contains `quantity 2, unitPriceMinor 1250, unitCostMinor null, newProduct { name: "Mug", categoryName: "Merch" }`; `input.taxRateBp` is `1000`; `input.occurredAt` equals `new Date(2026, 2, 10, 9, 30).getTime()`; wire parse succeeds |

T-U-19 to T-U-27 are one `it.each` titled `rejects %s`. Each parses `form(patch)` with `saleFormSchema`, expects
`success === false`, and expects the list of issue paths (each `issue.path.join(".")`) to contain the listed path.
In the patches, "item" means the base item with the listed fields changed.

| ID | Name | Patch | Expected issue path |
| --- | --- | --- | --- |
| T-U-19 | zero quantity | `items: [item with quantity "0"]` | `items.0.quantity` |
| T-U-20 | fractional quantity | `items: [item with quantity "1.5"]` | `items.0.quantity` |
| T-U-21 | bad price | `items: [item with unitPrice "12.345"]` | `items.0.unitPrice` |
| T-U-22 | missing product | `items: [item with productMode "existing", productId ""]` | `items.0.productId` |
| T-U-23 | tax over 100% | `taxRate: "101"` | `taxRate` |
| T-U-24 | discount over subtotal | `discount: "30"` (subtotal is 2 × 12.50 = 25.00) | `discount` |
| T-U-25 | new customer without name | `customerMode: "new"` | `customerName` |
| T-U-26 | bad email | `customerMode: "new", customerName: "Ann", customerEmail: "nope"` | `customerEmail` |
| T-U-27 | no items | `items: []` | `items` |

| ID | Title | Steps | Expected |
| --- | --- | --- | --- |
| T-U-28 | wire schema rejects an item with both or neither product reference | `base = saleFormToInput(form())`; "both" is base with `items[0].productId = "p1"` added next to `newProduct`; "neither" is base with `items[0].newProduct = null` and no `productId` | `saleInputSchema.safeParse` fails for both |
| T-U-29 | wire schema rejects non-integer money | base with `discountMinor: 1.5` | parse fails |

### 4.4 `tests/unit/csv.test.ts`

Functions under test from `src/lib/csv.ts`: `toCsv`, `parseCsv`, `safeCell`.

| ID | Title | Steps | Expected |
| --- | --- | --- | --- |
| T-U-30 | writes a BOM, header row and CRLF lines, quoting when needed | `csv = toCsv(["Name","Note"], [["Mug", 'say "hi", ok'], ["Pot", "x"]])` | `csv` starts with `"\uFEFFName,Note\r\n"`; contains `"say ""hi"", ok"` (quoted, inner quotes doubled); `parseCsv(csv)` equals `[{ Name: "Mug", Note: 'say "hi", ok' }, { Name: "Pot", Note: "x" }]` |
| T-U-31 | neutralises spreadsheet formula injection | `safeCell` on `"=SUM(A1)"`, `"@cmd"`, `"-1+1"`, `-5`, `null` | `"'=SUM(A1)"`, `"'@cmd"`, `"'-1+1"`, `-5` (numbers pass through unchanged), `""` |

## 5. Integration tests

Each case opens its own database with `testDb()` unless stated otherwise. `march` means
`{ from: at(2026, 3, 1, 0), to: at(2026, 4, 1, 0) }`. Error assertions use `AppError` from `src/server/errors.ts`
(carries an HTTP `status`) and `ZodError` from `zod`. The status codes and messages shown in brackets are the
required behaviour; the tests assert at least the error class or the quoted pattern.

### 5.1 `tests/integration/db.test.ts`

Under test: `openDatabase` (`src/db/client.ts`), `getSettings` / `updateSettings` (`src/server/services/settings.ts`),
`backupTo` / `verifyBackupFile` (`src/server/services/maintenance.ts`), `getSale`.

| ID | Title | Setup and steps | Expected |
| --- | --- | --- | --- |
| T-I-01 | first open creates the data directory, the file, all tables and default settings | `file = <tmpDir>/nested/dir/sales.db` (directories do not exist yet); `testDb(file)` | file exists; `sqlite_master` table names include `sales`, `sale_items`, `products`, `customers`, `categories`, `settings`; `pragma journal_mode` (simple) is `"wal"`; `pragma foreign_keys` (simple) is `1`; `getSettings(db)` contains `currency "USD", locale "en-US", defaultRange "30d"` |
| T-I-02 | reopening is idempotent: no migrations re-run, data kept | `db1 = testDb(file)`; `s = sale(db1)`; `updateSettings(db1, { currency: "EUR", locale: "de-DE", dateFormat: "dd/MM/yyyy", defaultRange: "7d" })`; close `db1`; `db2 = testDb(file)` on the same file (simulates a restart) | opening does not throw; `getSale(db2, s.id).totalMinor` is `1000`; `getSettings(db2)` contains `currency "EUR", defaultRange "7d"` |
| T-I-03 | foreign keys and CHECK constraints are enforced by the database | raw insert into `sale_items (id, sale_id, product_name, quantity, unit_price_minor, line_subtotal_minor, line_net_minor, position)` with values `('x','missing','P',1,1,1,1,0)`; then `s = sale(db)` and raw `UPDATE sales SET status = 'bogus' WHERE id = ?` | insert throws matching `/FOREIGN KEY/`; update throws matching `/CHECK/` |
| T-I-04 | a clear error when the database cannot be opened | `openDatabase(<path of an existing directory>)` | throws matching `/Cannot open SQLite database/` |
| T-I-05 | a consistent, verifiable backup of a live database | `sale(db)`; `file = await backupTo(db, tmpDir())`; `verifyBackupFile(file)`; `copy = testDb(file)` | `verifyBackupFile` does not throw; `SELECT count(*) AS n FROM sales` on `copy` returns `{ n: 1 }` |

### 5.2 `tests/integration/sales.test.ts`

Under test from `src/server/services/sales.ts`: `createSale`, `updateSale`, `getSale`, `deleteSales`, `listSales`;
plus `createProduct` / `listProducts` (catalog), `createCustomer` / `getCustomerDetail` (customers) and
`salesQuerySchema` (validation). `q(o)` means `salesQuerySchema.parse(o)`, which applies the defaults
`sort "date"`, `dir "desc"`, `page 0`, `pageSize 25`.

| ID | Title | Setup and steps | Expected |
| --- | --- | --- | --- |
| T-I-06 | create stores snapshots, computed totals and an order reference | `createSale` with `occurredAt at(2026,3,10)`, `customer { mode: "new", name: "Ada" }`, `paymentMethod "cash"`, `status "completed"`, items `{ newProduct { name "Kettle", categoryName "Equipment" }, quantity 2, unitPriceMinor 2500, unitCostMinor 1000 }` and `{ newProduct { name "Beans", categoryName "Coffee" }, quantity 1, unitPriceMinor 1000, unitCostMinor 400 }`, `discountMinor 600`, `taxRateBp 1000` | `orderRef` matches `/^ORD-[0-9A-F]{8}$/`; sale contains `subtotalMinor 6000, discountMinor 600, netMinor 5400, taxMinor 540, totalMinor 5940, costMinor 2400, customerName "Ada"`; items as `[productName, categoryName, lineNetMinor]` equal `[["Kettle","Equipment",4500], ["Beans","Coffee",900]]`; product "Kettle" in `listProducts(db)` has `defaultPriceMinor 2500, defaultCostMinor 1000` |
| T-I-07 | historical product name and price survive a product edit | `p = createProduct(db, { name: "Mug", defaultPriceMinor: 1200 })`; `createSale` (walk-in, card, completed, `at(2026,3,1)`) with item `{ productId: p.id, quantity 1, unitPriceMinor 1200, unitCostMinor null }`; raw `UPDATE products SET name = 'Big Mug', default_price_minor = 9999` for `p.id` | `getSale(...).items[0]` contains `productName "Mug", unitPriceMinor 1200` |
| T-I-08 | update replaces items and recomputes totals | `s = sale(db, { qty: 1, price: 1000 })`; `updateSale(db, s.id, …)` with the same `occurredAt` and `orderRef`, walk-in, card, `status "refunded"`, one item `{ productId: s.items[0].productId, quantity 3, unitPriceMinor 700, unitCostMinor null }`, discount `0`, tax `0` | result contains `status "refunded", netMinor 2100, orderRef s.orderRef`; `items` has length `1` |
| T-I-09 | duplicate order references and invalid input are rejected server-side | `sale(db, { orderRef: "A-1" })`, then again with `"A-1"`; `sale(db, { qty: 0 })`; `sale(db, { discountMinor: 5000, price: 1000 })` | second `A-1` throws `AppError` [409, `Order reference "A-1" is already used`]; `qty 0` throws `ZodError`; discount over subtotal throws `ZodError` |
| T-I-10 | a failing sale is atomic and leaves no partial rows | `createSale` with `customer { mode: "new", name: "Temp" }` and items `{ newProduct { name "Ok" }, 1 × 100 }` and `{ productId "does-not-exist", 1 × 100 }` | throws matching `/no longer exists/` [400, `Selected product no longer exists`]; count of `sales` + `products` + `customers` rows is `0` (the new customer and new product were rolled back) |
| T-I-11 | delete removes sales and their items | `a = sale(db)`, `b = sale(db)`; `deleteSales(db, [a.id, b.id])` | returns `2`; `SELECT count(*) AS n FROM sale_items` is `{ n: 0 }` |
| T-I-12 | sales link to customers and appear in customer history | `c = createCustomer(db, { name: "Bea" })`; `sale` with `customer { mode: "existing", customerId: c.id }`, price `1500`; another with price `500`, `status "refunded"`; `getCustomerDetail(db, c.id)` | `history` has length `2` (every status is listed) |

Seed for T-I-13 to T-I-16 (one fresh DB per case, built by a local `seed()` function using `sale()`):

| Ref | `occurredAt` | Product | Qty × price | Status | Payment | Customer | Total |
| --- | --- | --- | --- | --- | --- | --- | --- |
| R-1 | `at(2026,3,1)` | Alpha | 1 × 500 | completed | cash | walk-in | 500 |
| R-2 | `at(2026,3,5)` | Beta | 1 × 900 | pending | card | walk-in | 900 |
| R-3 | `at(2026,3,9)` | Alpha | 7 × 100 | completed | card | new, name `Zed 100%`, phone `null` | 700 |
| R-4 | `at(2026,4,1)` | Gamma | 1 × 300 | refunded | card | walk-in | 300 |

| ID | Title | Steps | Expected |
| --- | --- | --- | --- |
| T-I-13 | filters by half-open date range | `listSales(db, q({ from: at(2026,3,1,0), to: at(2026,3,9,0) }))` | sorted order refs are `["R-1","R-2"]` (R-3 at noon on 9 March is outside) |
| T-I-14 | search covers order ref, product and customer; LIKE wildcards are literal | `q: "alpha"`; `q: "R-4"`; `q: "100%"`; `q: "%"` | `total` `2` (case-insensitive product match); `rows[0].orderRef` `"R-4"`; refs `["R-3"]`; `total` `1` (`%` matches only the literal character in "Zed 100%") |
| T-I-15 | filters by status and payment method | `status: ["pending","refunded"]`; `paymentMethod: ["cash"]` | `total` `2`; refs `["R-1"]` |
| T-I-16 | sorts and pages on the server | `sort "amount", dir "desc"`; `sort "quantity", dir "desc"`; `page 1, pageSize 3` (default date desc) | refs `["R-2","R-3","R-1","R-4"]` (amount sorts by `totalMinor`); first row contains `orderRef "R-3", units 7`; result contains `total 4` and its rows are `["R-1"]` |

### 5.3 `tests/integration/analytics.test.ts`

Under test from `src/server/services/analytics.ts`: `kpis(db, range)`, `series(db, range, bucket)`,
`dashboard(db, range)`. Constant `MIN_SALES_FOR_INSIGHTS = 3`.

| ID | Title | Setup and steps | Expected |
| --- | --- | --- | --- |
| T-I-17 | revenue counts only completed sales, net of discount, excluding tax | `sale` price `10000`, `discountMinor 1000`, `taxRateBp 1000` (net 9000, tax 900); price `5000` `pending`; price `7000` `cancelled`; price `3000` `refunded`; `kpis(db, march)` | contains `revenueMinor 9000, salesCount 1, aovMinor 9000, units 1, pendingMinor 5000, pendingCount 1, refundedMinor 3000, refundedCount 1, cancelledCount 1` |
| T-I-18 | a refund removes the sale from revenue exactly once | `s = sale` price `4000`; check; raw `UPDATE sales SET status = 'refunded'` for `s.id`; check again | first `revenueMinor 4000`; then `revenueMinor 0, salesCount 0, refundedMinor 4000` (no negative double count) |
| T-I-19 | profit only when every completed sale has a known cost | `sale` price `1000` cost `600`; then `sale` price `1000` cost `null` product `"Mystery"` | `profitMinor` `400`; then `null` |
| T-I-20 | half-open range boundaries | sales at `at(2026,3,1,0)` (included), `at(2026,4,1,0)` (excluded), `at(2026,2,28,23)` | `kpis(db, march).salesCount` is `1` |
| T-I-21 | time series zero-fills and buckets by local day | sales at `at(2026,3,2,23)` price `100`, `at(2026,3,2,1)` price `200`, `at(2026,3,4,9)` price `300`; `series(db, { from: at(2026,3,1,0), to: at(2026,3,5,0) }, "day")` | `[key, revenueMinor, orders]` equals `[["2026-03-01",0,0], ["2026-03-02",300,2], ["2026-03-03",0,0], ["2026-03-04",300,1]]` |
| T-I-22 | rankings of products, categories and payment methods use completed sales | `p = createProduct(db, { name: "Grinder", defaultPriceMinor: 5000 })` (no category); `createSale` at `at(2026,3,3)`, walk-in, card, completed, `2 × 5000` of `p`; the same but `cancelled` with `9 × 5000`; `sale` product `"Beans"` category `"Coffee"` price `1500` payment `"cash"`; `d = dashboard(db, march)` | `d.products` as `[name, revenueMinor, units]` equals `[["Grinder",10000,2], ["Beans",1500,1]]`; `d.categories` names equal `["Uncategorized","Coffee"]`; card entry in `d.payments` has `revenueMinor 10000`; `d.kpis.current.revenueMinor` is `11500`; `d.series` has length `31` |
| T-I-23 | compares with the previous period; withholds insights when data is thin | `sale` at `at(2026,2,10)` price `1000`; `sale` at `at(2026,3,10)` price `1500`; `d = dashboard(db, march)`; then add sales at `at(2026,3,11)` and `at(2026,3,12)`, price `1500` each; `d2 = dashboard(db, march)` | `d.kpis.previous.revenueMinor` is `1000`; `d.insights` equals `[]` (1 completed sale is below 3); in `d2.insights` the item with `id "vs-prev"` has `value "+350.0%"` (4500 vs 1000) |

### 5.4 `tests/integration/catalog.test.ts`

Under test from `src/server/services/catalog.ts`, `customers.ts`, `maintenance.ts` and `src/server/demo-seed.ts`.

| ID | Title | Setup and steps | Expected |
| --- | --- | --- | --- |
| T-I-24 | categories: create, rename, case-insensitive unique, protected when in use | `c = createCategory(db, { name: "Tea" })`; `createCategory(db, { name: "tea" })`; `renameCategory(db, c.id, { name: "Teas" })`; `createProduct(db, { name: "Sencha", categoryId: c.id, defaultPriceMinor: 900 })`; `deleteCategory(db, c.id)` | duplicate throws `AppError` [409, `Category "tea" already exists`]; `listCategories(db)[0]` contains `name "Teas", productCount 1`; delete throws matching `/in use\|used/i` [409, `This category is used by 1 product(s). Reassign them first.`] |
| T-I-25 | unique SKUs and price validation | `createProduct` `{ name "A", sku "SKU-1", defaultPriceMinor 100 }`; then `{ name "B", sku "SKU-1", defaultPriceMinor 100 }`; then `{ name "C", defaultPriceMinor -1 }` | second throws `AppError` [409, `SKU "SKU-1" is already used by another product`]; third throws (Zod validation) |
| T-I-26 | products with sales history are archived, not deleted | `s = sale(db, { product: "Kept" })`, `pid = s.items[0].productId`; `deleteProduct(db, pid)`; `setProductArchived(db, pid, true)`; then `unused = createProduct(db, { name: "Unused", defaultPriceMinor: 100 })`, `updateProduct(db, unused.id, { name: "Unused 2", defaultPriceMinor: 200 })`, `deleteProduct(db, unused.id)` | delete of `pid` throws `AppError` [409, `This product appears in past sales. Archive it instead to keep history intact.`]; `listProducts(db)` does not contain `pid`; `listProducts(db, { includeArchived: true })` contains it with `archived true`; `getSale(db, s.id).items[0].productName` is `"Kept"`; `unused` is absent even with `includeArchived: true` |
| T-I-27 | customers with sales cannot be deleted | `c = createCustomer(db, { name: "Cy" })`; `sale` with `customer { mode: "existing", customerId: c.id }`; `deleteCustomer(db, c.id)` | throws `AppError` [409, `This customer has 1 sale(s). Customers with history cannot be deleted.`] |
| T-I-28 | demo data is tagged and removable without touching real records | `seedDemoData(db, { now: new Date(2026, 2, 15), days: 30 })`; `before = stats(db)`; `real = sale(db, { product: "Real product" })`; `removeDemoData(db)`; `after = stats(db)` | `hasData(db)` is `true`; `before.demoSales === before.sales` and `before.sales > 0`; `after` contains `sales 1, demoSales 0, demoRows 0`; `getSale(db, real.id).items[0].productName` is `"Real product"` |

`stats(db)` returns `{ sales, demoSales, products, customers, categories, demoRows }`, where `demoRows` is the sum
of `is_demo = 1` rows across sales, products, customers and categories.

### 5.5 `tests/integration/exports.test.ts`

Under test from `src/server/services/exports.ts`: `salesCsv(db, query, currency)` and
`reportCsv(db, report, range, currency)`; results are read back with `parseCsv`.

| ID | Title | Setup and steps | Expected |
| --- | --- | --- | --- |
| T-I-29 | sales export respects filters and writes major-unit amounts | `sale` `{ orderRef "E-1", price 1250, taxRateBp 1000, notes "=HYPERLINK(1)" }`; `sale` `{ orderRef "E-2", price 999, status "pending" }`; `parseCsv(salesCsv(db, salesQuerySchema.parse({ status: ["completed"] }), "USD"))` | exactly `1` row, containing `Order "E-1"`, `Status "Completed"`, `"Net (USD)" "12.50"`, `"Tax (USD)" "1.25"`, `"Total (USD)" "13.75"`, `Notes "'=HYPERLINK(1)"` |
| T-I-30 | analytics reports export for a date range | `sale` at `at(2026,3,2)` price `500` product `"Tea"`; range `at(2026,3,1,0)`..`at(2026,3,4,0)`; `reportCsv(db, "timeseries", range, "USD")`; `reportCsv(db, "products", range, "USD")` | timeseries has `3` rows (one per day, zero-filled); first products row contains `Product "Tea"` |

## 6. End-to-end tests (`tests/e2e/app.spec.ts`)

Run in serial mode against the empty `.tmp/e2e` database, in the order below; later cases rely on the state left by
earlier ones. "Heading" means the level-1 heading inside the `banner` landmark (the app header).

| ID | Title | Steps | Expected |
| --- | --- | --- | --- |
| T-E-01 | first run shows an empty, working dashboard | go to `/` | heading text is `Overview`; text `No sales in this period` is visible (first match) |
| T-E-02 | sidebar navigates between every section | go to `/`; locate the navigation landmark named `Main navigation` (or the element with `data-sidebar="sidebar"`); click each link by exact name in the order `Sales`, `Products`, `Customers`, `Analytics`, `Settings`, `Overview` | after each click the heading text equals the link name |
| T-E-03 | validation blocks an incomplete sale | go to `/sales`; click the first `Add sale` button; in the form landmark named `Add sale`, click `Record sale` without filling anything; then click `Cancel` | an element with role `alert` is visible inside the form; the form is still visible (nothing saved); after Cancel the text `No sales yet` is visible |
| T-E-04 | a recorded sale shows up everywhere and survives a reload | go to `/sales`; click `Add sale`; in the form choose radio `New product`; fill label `Product name` = `E2E Widget`, `Category` = `Testing`, `Qty` = `3`, `Unit price` = `12.50`; click `Record sale` | toast `Sale recorded` visible; form hidden; a table row containing `E2E Widget` is visible and contains `$37.50`; after `page.reload()` the row is still visible; on `/` the text `$37.50` is visible (first match); on `/products` a row containing `E2E Widget` is visible |
| T-E-05 | mobile layout uses a navigation drawer | set viewport `390 × 844`; go to `/` | link `Products` (exact) is hidden; after clicking button `Toggle navigation` and then link `Products`, heading is `Products`; `document.documentElement.scrollWidth - window.innerWidth` is `<= 0` (no horizontal overflow) |

These cases fix the following UI contract (details in [06-Frontend-and-Design.md](06-Frontend-and-Design.md)):
accessible names `Main navigation`, `Add sale` (button and form), `Record sale`, `Cancel`, `New product`,
`Toggle navigation`; field labels `Product name`, `Category`, `Qty`, `Unit price`; field errors rendered with
`role="alert"`; empty-state copy `No sales in this period` and `No sales yet`; toast `Sale recorded`; sale form
defaults (walk-in customer, card, completed, tax 0) so that 3 × 12.50 totals `$37.50`.

## 7. Commands and pass criteria

| Command | Runs | Pass criterion |
| --- | --- | --- |
| `pnpm test` | `vitest run` | 61 passed, 0 failed, 0 skipped |
| `pnpm test:e2e` | `playwright test` (starts its own dev server on port 3200) | 5 passed, 0 failed |
| `pnpm check` | `pnpm typecheck && pnpm lint && pnpm format:check && pnpm test` | exits 0 |
| `pnpm test:e2e:install` | `playwright install chromium` | one-time setup before the first e2e run |

`pnpm typecheck` is `next typegen && tsc --noEmit`. The work is complete only when `pnpm test`, `pnpm test:e2e` and
`pnpm check` are all green on the same tree. Track progress in [../../BUILD_PLAN.md](../../BUILD_PLAN.md) and follow
[../../CONTRIBUTING.md](../../CONTRIBUTING.md).
