# 03 — Domain Rules

This document defines the pure domain layer of Ledgerline: money arithmetic, date ranges, shared constants,
validation schemas, CSV encoding and the metric definitions behind every dashboard number. The `src/lib` modules have
no database or framework dependency (only `date-fns`, `zod` and `papaparse`), run in both browser and server, and must
be pinned by unit tests (see [07-Testing.md](07-Testing.md)). Table shapes are in [04-Data-Model.md](04-Data-Model.md);
HTTP contracts are in [05-Backend-and-API.md](05-Backend-and-API.md).

## 1. Module layout and global rules

| File | Responsibility |
| --- | --- |
| `src/lib/money.ts` | Minor-unit money, parsing, formatting, sale totals, percent change |
| `src/lib/date-range.ts` | Range presets, half-open ranges, previous period, buckets, labels |
| `src/lib/constants.ts` | Statuses, payment methods, labels, date formats, locales, page sizes |
| `src/lib/validation.ts` | Every Zod schema (wire and form) and the form-to-wire converters |
| `src/lib/csv.ts` | CSV encoding, formula-injection guard, download response, parsing |
| `src/server/services/analytics.ts` | Metric queries, dashboard composition, insights (server only) |
| `src/server/services/exports.ts` | CSV column sets for sales, reports, products, customers (server only) |

- DR-01. Money is always an integer number of **minor units** (cents). Never store, sum or compare floating-point
  major amounts. Conversion to major units happens only for display and CSV.
- DR-02. Tax rates are integers in **basis points** (1% = 100 bp, 10% = 1000 bp, 100% = 10000 bp).
- DR-03. Timestamps are integer epoch milliseconds. Calendar logic (days, weeks, months) uses the **local time zone
  of the machine** running the code. In a local-first setup the browser and the server share that machine.
- DR-04. Date ranges are half-open: `from <= t < to`.

## 2. Money module (`src/lib/money.ts`)

### 2.1 Constants and types

| Export | Value / shape | Notes |
| --- | --- | --- |
| `MINOR_PER_MAJOR` | `100` | Every supported currency has 2 fraction digits |
| `CURRENCIES` | `["USD", "EUR", "GBP", "CAD", "AUD", "NZD", "SGD", "CHF", "IDR", "INR"] as const` | Order shown in Settings |
| `CurrencyCode` | union of `CURRENCIES` | |
| `MAX_UNIT_PRICE_MINOR` | `1_000_000_000` | 10,000,000.00 major; also the cap for every minor amount on the wire |
| `MAX_QUANTITY` | `100_000` | Per line |
| `MAX_TAX_RATE_BP` | `10_000` | 100% |
| `LineInput` | `{ quantity: number; unitPriceMinor: number; unitCostMinor: number \| null }` | |
| `SaleTotals` | see 2.4 | |

The caps keep every intermediate value inside `Number.MAX_SAFE_INTEGER`. Mark the 2-decimal assumption with a code
comment: supporting 0- or 3-decimal currencies (JPY, KWD) would need a per-currency exponent stored with the data.

### 2.2 Parsing and string conversion

| Signature | Semantics |
| --- | --- |
| `parseMoney(input: string): number \| null` | Trim, remove **every** `,`, then require the whole string to match one or more digits, optionally followed by `.` and 0–2 digits (regex `^(\d+)(?:\.(\d{0,2}))?$`). Result = `integerPart × 100 + fraction`, where the fraction is right-padded with zeros to 2 digits, using integer arithmetic on the digit strings (never `parseFloat(x) * 100`). Return `null` if the pattern fails or the result is not a safe integer. |
| `toMajorString(minor: number): string` | Plain editable string, exactly 2 decimals, no grouping, no symbol; negative values get a leading `-`. `123456 → "1234.56"`, `5 → "0.05"`, `-150 → "-1.50"`. |
| `parsePercentToBp(input: string): number \| null` | Trim. Empty string → `0`. Otherwise the same digit pattern as `parseMoney` (no comma stripping): `whole × 100 + fraction`. Invalid → `null`. No upper bound here; schemas enforce `MAX_TAX_RATE_BP`. `"10" → 1000`, `"8.25" → 825`, `"x" → null`. |
| `bpToPercentString(bp: number): string` | `(bp / 100).toFixed(2)`, then strip trailing zeros and a dangling decimal point. `1000 → "10"`, `825 → "8.25"`, `1050 → "10.5"`, `0 → "0"`. |

`parseMoney` examples:

| Input | Result | Input | Result |
| --- | --- | --- | --- |
| `"12"` | 1200 | `""` | null |
| `"12.5"` | 1250 | `"abc"` | null |
| `"0.07"` | 7 | `"1.234"` | null (3 decimals) |
| `"1,234.56"` | 123456 | `"-5"` | null (no sign) |
| `"0.29"` | 29 (no float drift) | `"1e3"` | null |
| `"1.15"` | 115 | `"."`, `".5"` | null (leading digit required) |
| `" 12 "` | 1200 | `"1."` | 100 (empty fraction allowed) |

### 2.3 Formatting

`formatMoney(minor: number, currency: string, locale: string, opts?: { compact?: boolean }): string`

- Formats `minor / 100` with `Intl.NumberFormat(locale, { style: "currency", currency })`.
- With `compact: true` also pass `notation: "compact", maximumFractionDigits: 1` (chart axes, e.g. "$12.3K").
- Cache one `Intl.NumberFormat` per `locale|currency|compact` key in a module-level `Map`.
- Expected: `formatMoney(123456, "USD", "en-US") === "$1,234.56"`; `formatMoney(123456, "EUR", "de-DE")` matches
  `/1\.234,56\s€/` (the separator is a non-breaking space).

### 2.4 Sale totals

`computeSaleTotals(lines: LineInput[], discountMinor: number, taxRateBp: number): SaleTotals`

`SaleTotals` = `{ subtotalMinor, discountMinor, netMinor, taxMinor, totalMinor, costMinor: number | null, lines: { subtotalMinor, netMinor, costMinor: number | null }[] }`,
with `lines` in input order.

1. `lineSubtotal[i] = quantity[i] × unitPriceMinor[i]`; `subtotal = Σ lineSubtotal`.
2. If `discountMinor < 0` or `discountMinor > subtotal`, throw `RangeError("Discount must be between 0 and the subtotal")`.
3. `net = subtotal − discountMinor`. The discount is sale-level and applies **before** tax.
4. Allocate the discount to lines proportionally to line subtotal so that line nets sum exactly to `net`:
   - If `subtotal = 0`, every share is 0.
   - Every line except the last: `share[i] = floor(discount × lineSubtotal[i] / subtotal)`, computed with `BigInt`.
   - Last line: `share = discount − Σ previous shares` (the rounding remainder always lands on the last line).
   - `line.netMinor = lineSubtotal[i] − share[i]`.
5. `line.costMinor = unitCostMinor × quantity`, or `null` when `unitCostMinor` is `null`.
6. Tax = round-half-up of `net × taxRateBp / 10000` in `BigInt`. For non-negative `n` and positive `d`, round-half-up
   division is `(2n + d) / (2d)` with truncating integer division; here `n = net × taxRateBp`, `d = 10000`.
7. `total = net + tax`.
8. Sale `costMinor` = Σ line costs, or `null` if **any** line cost is `null`.

The server stores these values on the sale and its lines (see [04-Data-Model.md](04-Data-Model.md)); the sale form shows
the same computation live as a preview.

### 2.5 Percent change

`percentChange(current: number, previous: number): number | null` returns `null` when `previous === 0` (a change from zero
is not meaningful); otherwise `(current − previous) / |previous| × 100` as an unrounded float. Callers round for
display. `percentChange(150, 100) = 50`; `percentChange(5, 0) = null`.

### 2.6 Worked examples

| Lines (qty × price, cost) | Discount | Tax bp | Subtotal | Line nets | Net | Tax | Total | Cost |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 3 × 999, cost 400 | 0 | 1000 | 2997 | [2997] | 2997 | 300 (299.7) | 3297 | 1200 |
| 1 × 1005 | 0 | 500 | 1005 | [1005] | 1005 | 50 (50.25) | 1055 | null |
| 1 × 1010 | 0 | 500 | 1010 | [1010] | 1010 | 51 (50.5 rounds up) | 1061 | null |
| 1 × 1000, 2 × 1000 | 100 | 1000 | 3000 | [967, 1933] | 2900 | 290 | 3190 | null |
| 1 × 500 cost 100, 1 × 500 no cost | 0 | 0 | 1000 | [500, 500] | 1000 | 0 | 1000 | null |
| 1 × 100 | 101 | 0 | — | — | — | — | — | throws `RangeError` |

Row 4: first share `floor(100 × 1000 / 3000) = 33`, last line takes `100 − 33 = 67`.

## 3. Date-range module (`src/lib/date-range.ts`)

Use `date-fns` for all calendar arithmetic (`startOfDay`, `addDays`, `startOfMonth`, `subMonths`, `startOfYear`,
`startOfWeek`, `differenceInCalendarDays`, `eachDayOfInterval`, `format`). Never step days by adding `86_400_000` ms:
DST days last 23 or 25 hours.

### 3.1 Exports

| Export | Definition |
| --- | --- |
| `RANGE_PRESETS` | `["today", "7d", "30d", "thisMonth", "prevMonth", "thisYear", "custom"] as const` |
| `RangePreset` | union of `RANGE_PRESETS` |
| `PRESET_LABELS` | today "Today", 7d "Last 7 days", 30d "Last 30 days", thisMonth "This month", prevMonth "Previous month", thisYear "This year", custom "Custom range" |
| `DateRange` | `{ from: number; to: number }`, epoch ms, half-open |
| `RangeSelection` | `{ preset: RangePreset; customFrom?: string; customTo?: string }`; custom bounds are `yyyy-MM-dd` local days, both inclusive |
| `Bucket` | `"day" \| "week" \| "month"` |
| `parseDay(day: string): Date` | Split `yyyy-MM-dd` and build local midnight `new Date(y, m − 1, d)`. Never `new Date("2026-03-01")`, which parses as UTC |
| `resolveRange(sel: RangeSelection, now: Date = new Date()): DateRange` | 3.2 |
| `previousRange(range: DateRange): DateRange` | 3.3 |
| `autoBucket(range: DateRange): Bucket` | 3.4 |
| `dayKey(d: Date \| number): string` | `format(d, "yyyy-MM-dd")` |
| `bucketKeyForDay(day: string, bucket: Bucket): string` | 3.4 |
| `bucketKeys(range: DateRange, bucket: Bucket): string[]` | 3.4 |
| `formatBucketLabel(key: string, bucket: Bucket): string` | 3.4 |

### 3.2 Presets

Let `today = startOfDay(now)` and `tomorrow = addDays(today, 1)`.

| Preset | `from` | `to` |
| --- | --- | --- |
| `today` | `today` | `tomorrow` |
| `7d` | `addDays(tomorrow, −7)` (today and the 6 days before) | `tomorrow` |
| `30d` | `addDays(tomorrow, −30)` | `tomorrow` |
| `thisMonth` | `startOfMonth(now)` | `tomorrow` |
| `prevMonth` | `startOfMonth(subMonths(now, 1))` | `startOfMonth(now)` |
| `thisYear` | `startOfYear(now)` | `tomorrow` |
| `custom` | earlier of `parseDay(customFrom)` and `parseDay(customTo)` | `addDays(later, 1)` |

- Reversed custom bounds are swapped: `2026-01-10 → 2026-01-05` equals `2026-01-05 → 2026-01-10`.
- Custom with either bound missing falls back to `30d`.
- Example, `now = Sun 15 Mar 2026 14:30` local: today `[15 Mar, 16 Mar)`; 7d `[9 Mar, 16 Mar)`; 30d `[14 Feb, 16 Mar)`;
  thisMonth `[1 Mar, 16 Mar)`; prevMonth `[1 Feb, 1 Mar)`; thisYear `[1 Jan, 16 Mar)`; custom `2026-01-05..2026-01-10` →
  `[5 Jan, 11 Jan)`.

### 3.3 Previous period

`days = max(1, differenceInCalendarDays(to, from))`; previous = `[addDays(from, −days), from)`. Counting calendar days
(not milliseconds) keeps the window aligned across DST. Example: `[9 Mar, 16 Mar)` → `[2 Mar, 9 Mar)`. Every comparison
in the app (KPI deltas, dashed chart overlay, "Vs previous period" insight) uses this window.

### 3.4 Buckets

`autoBucket` with `n = differenceInCalendarDays(to, from)`:

| `n` | Bucket |
| --- | --- |
| `n <= 31` | `day` |
| `32 <= n <= 182` | `week` |
| `n >= 183` | `month` |

Examples: 30d → day; `[1 Dec 2025, 1 Mar 2026)` (90 days) → week; custom `2025-01-01..2025-12-31` → month.

`bucketKeyForDay(day, bucket)` returns the `yyyy-MM-dd` key of the bucket's first day:

- `day`: the day itself.
- `week`: `startOfWeek(day, { weekStartsOn: 1 })`. **Weeks start on Monday.**
- `month`: `startOfMonth(day)`, e.g. `2026-02-01`.

`bucketKeys(range, bucket)` walks every calendar day from `from` to `addDays(to, −1)` inclusive, maps each day to its
bucket key and appends the key when it differs from the last appended one. The list is ordered and gap-free so charts
can zero-fill. The first week or month key may lie before `from`.

| Range | Bucket | Keys |
| --- | --- | --- |
| `[1 Mar 2026, 4 Mar 2026)` | day | `2026-03-01, 2026-03-02, 2026-03-03` |
| `[1 Mar 2026, 10 Mar 2026)` (1 Mar is a Sunday) | week | `2026-02-23, 2026-03-02, 2026-03-09` |
| `[20 Jan 2026, 2 Mar 2026)` | month | `2026-01-01, 2026-02-01, 2026-03-01` |

`formatBucketLabel(key, bucket)` (date-fns default English locale):

| Bucket | Output | Example |
| --- | --- | --- |
| day | `format(d, "MMM d")` | "Mar 9" |
| week | "Wk of " + `format(d, "MMM d")` | "Wk of Mar 2" |
| month | `format(d, "MMM yyyy")` | "Mar 2026" |

## 4. Constants (`src/lib/constants.ts`)

| Export | Values, in this order |
| --- | --- |
| `SALE_STATUSES` / `SaleStatus` | `completed`, `pending`, `cancelled`, `refunded` |
| `STATUS_LABELS` | Completed, Pending, Cancelled, Refunded |
| `PAYMENT_METHODS` / `PaymentMethod` | `card`, `cash`, `bank_transfer`, `e_wallet`, `other` |
| `PAYMENT_LABELS` | Card, Cash, Bank transfer, E-wallet, Other |
| `DATE_FORMATS` / `DateFormat` | `MMM d, yyyy`, `d MMM yyyy`, `yyyy-MM-dd`, `MM/dd/yyyy`, `dd/MM/yyyy` (date-fns patterns) |
| `LOCALES` | `en-US`, `en-GB`, `en-AU`, `de-DE`, `fr-FR`, `es-ES`, `id-ID`, `ja-JP` |
| `PAGE_SIZES` | `10`, `25`, `50`, `100` |

All arrays are `as const`; label maps are `Record<Union, string>`. The currency list (`CURRENCIES`) lives in the money
module. Settings defaults (USD, en-US, `MMM d, yyyy`, 30d) are in [04-Data-Model.md](04-Data-Model.md). The UI renders
dates with `format(ms, settings.dateFormat)` and date-times with the pattern `settings.dateFormat + ", HH:mm"`; plain
numbers use `Intl.NumberFormat(settings.locale)`.

## 5. Validation module (`src/lib/validation.ts`)

Use Zod 4 top-level helpers (`z.int()`, `z.email()`). There are two schema families:

- **Wire schemas** validate JSON sent to the API: money is integer minor units, time is epoch ms. The server
  re-validates every request with them; they are the source of truth.
- **Form schemas** validate React Hook Form state, where every input is the string the user typed. A converter turns
  validated form values into a wire object.

"Default message" means Zod's built-in message. Every other message below is exact UI copy.

### 5.1 Shared building blocks

| Name | Definition |
| --- | --- |
| `optionalText(max)` | string, trimmed, `max(max)`, optional, nullable; transform: empty string, `undefined` or `null` → `null` |
| `minor` | `z.int().min(0).max(MAX_UNIT_PRICE_MINOR)` (rejects `1.5`) |
| `moneyString(label, required = true)` | string refined: blank → valid only when not required; non-blank → `parseMoney(v) !== null`. Message: required "Enter a valid <label>"; optional "Enter a valid <label> or leave blank" |

### 5.2 Sale wire schemas

`saleItemInputSchema`:

| Field | Rule |
| --- | --- |
| `productId` | `string.min(1)`, nullable, optional |
| `newProduct` | nullable, optional object: `name` trimmed 1–120 (min message "Product name is required"), `categoryName` `optionalText(60)`, `sku` `optionalText(64)` |
| `quantity` | `z.int().min(1).max(MAX_QUANTITY)` |
| `unitPriceMinor` | `minor` |
| `unitCostMinor` | `minor`, nullable; key required; `null` = cost unknown |
| refine | exactly one of `productId` and `newProduct` is truthy; message "Choose an existing product or create a new one", path `productId` |

`customerInputSchema`, discriminated union on `mode`:

| `mode` | Fields |
| --- | --- |
| `walkin` | `name`: `optionalText(120)` |
| `existing` | `customerId`: `string.min(1)` |
| `new` | `name` trimmed 1–120 (min message "Customer name is required"); `email`: `z.email()` optional nullable, or `""` transformed to `null` (default message); `phone`: `optionalText(40)` |

`saleInputSchema` (type `SaleInput = z.output<typeof saleInputSchema>`):

| Field | Rule |
| --- | --- |
| `occurredAt` | `z.int().positive()`, epoch ms |
| `orderRef` | `optionalText(64)`; `null` means "generate one" on create and "keep the current one" on update |
| `customer` | `customerInputSchema` |
| `paymentMethod` | `z.enum(PAYMENT_METHODS)` |
| `status` | `z.enum(SALE_STATUSES)` |
| `items` | array of `saleItemInputSchema`, min 1 ("Add at least one item"), max 50 (default message) |
| `discountMinor` | `minor` |
| `taxRateBp` | `z.int().min(0).max(MAX_TAX_RATE_BP)` |
| `notes` | `optionalText(2000)` |
| superRefine | `discountMinor > Σ quantity × unitPriceMinor` → "Discount cannot exceed the subtotal", path `discountMinor` |

### 5.3 Query schemas

`salesQuerySchema` (type `SalesQuery`):

| Field | Rule | Default |
| --- | --- | --- |
| `q` | string, trimmed, max 100, optional | — |
| `status` | array of `SALE_STATUSES`, optional | — |
| `paymentMethod` | array of `PAYMENT_METHODS`, optional | — |
| `productId`, `categoryId` | string, optional | — |
| `from`, `to` | coerced integer epoch ms, optional | — |
| `sort` | `date` \| `amount` \| `quantity` \| `status` | `date` |
| `dir` | `asc` \| `desc` | `desc` |
| `page` | coerced integer ≥ 0 (zero-based) | `0` |
| `pageSize` | coerced integer 1–100 | `25` |

`parseSalesQuery(sp: URLSearchParams): SalesQuery` reads each key; an empty or missing value becomes `undefined`.
`status` and `paymentMethod` are comma-separated lists (`?status=completed,pending`), split and stripped of empty
entries. Invalid values throw `ZodError` (HTTP 400).

`rangeQuerySchema`: `{ from, to }` coerced integers; refine `to > from` ("Invalid date range"); refine
`to − from <= 11000 days` (`1000 × 86_400_000 × 11` ms, "Date range too large").

### 5.4 Sale form schemas (UI strings)

`saleFormItemSchema`:

| Field | Rule / message |
| --- | --- |
| `productMode` | `"existing" \| "new"` |
| `productId`, `productName`, `categoryName` | string (lengths are enforced by the wire schema: 120 / 60) |
| `quantity` | trimmed value matches `^\d+$` and is 1–100000: "Quantity must be a whole number from 1 to 100,000" (format the max with `toLocaleString("en-US")`) |
| `unitPrice` | `moneyString("price")` → "Enter a valid price"; then `parseMoney ≤ MAX_UNIT_PRICE_MINOR` → "Price is too large" |
| `unitCost` | `moneyString("cost", false)` → "Enter a valid cost or leave blank" |
| superRefine | `existing` without `productId` → "Select a product" (path `productId`); `new` with blank `productName` → "Product name is required" (path `productName`) |

`saleFormSchema` (type `SaleFormValues = z.input<typeof saleFormSchema>`):

| Field | Rule / message |
| --- | --- |
| `occurredAt` | matches `^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}$` (a `datetime-local` value): "Enter a valid date and time" |
| `orderRef` | string, max 64 |
| `customerMode` | `walkin` \| `existing` \| `new` |
| `customerId` | string |
| `customerName` | string, max 120 (walk-in name or new customer name) |
| `customerEmail` | `""` or `z.email("Enter a valid email")` |
| `paymentMethod`, `status` | enums as above |
| `items` | array of `saleFormItemSchema`, min 1: "Add at least one item" |
| `discount` | `moneyString("discount", false)` → "Enter a valid discount or leave blank" |
| `taxRate` | `parsePercentToBp` not `null` and ≤ 10000: "Tax rate must be between 0 and 100%" (blank = 0%) |
| `notes` | string, max 2000 |
| superRefine | `existing` without `customerId` → "Select a customer" (path `customerId`); `new` with blank name → "Customer name is required" (path `customerName`); discount above the form subtotal → "Discount cannot exceed the subtotal" (path `discount`). The form subtotal counts an unparsable price or quantity as 0 |

Issue paths are dotted, for example `items.0.quantity`, `items.0.unitPrice`, `items.0.productId`.

`saleFormToInput(f: SaleFormValues): SaleInput` (call only after the form validates):

| Wire field | Derived from |
| --- | --- |
| `occurredAt` | split `yyyy-MM-ddTHH:mm` into numbers, `new Date(y, mo − 1, d, h, mi).getTime()` (browser local time) |
| `orderRef` | trimmed; blank → `null` |
| `customer` | `existing` → `{ mode, customerId }`; `new` → `{ mode, name: trimmed, email: email or null, phone: null }`; `walkin` → `{ mode, name: trimmed or null }` |
| `items[i].productId` | `productId` when `existing`, else `null` |
| `items[i].newProduct` | when `new`: `{ name: trimmed, categoryName: trimmed or null, sku: null }`, else `null` |
| `items[i].quantity` | `parseInt(quantity, 10)` |
| `items[i].unitPriceMinor` | `parseMoney(unitPrice) ?? 0` |
| `items[i].unitCostMinor` | blank → `null`, else `parseMoney(unitCost)` |
| `discountMinor` | `parseMoney(discount or "0") ?? 0` |
| `taxRateBp` | `parsePercentToBp(taxRate) ?? 0` |
| `notes` | trimmed; blank → `null` |

Example: occurredAt `"2026-03-10T09:30"`, walk-in, one item `{ productMode: "new", productName: "Mug", categoryName: "Merch", quantity: "2", unitPrice: "12.50", unitCost: "" }`,
taxRate `"10"` → item `{ quantity: 2, unitPriceMinor: 1250, unitCostMinor: null, newProduct: { name: "Mug", categoryName: "Merch", sku: null } }`,
`taxRateBp: 1000`, `occurredAt = new Date(2026, 2, 10, 9, 30).getTime()`; the result passes `saleInputSchema`. With
that item, discount `"30"` fails (subtotal 25.00), quantity `"0"` or `"1.5"` fails, price `"12.345"` fails, taxRate
`"101"` fails.

### 5.5 Catalog, customer and settings schemas

| Schema | Fields |
| --- | --- |
| `productInputSchema` (wire, type `ProductInput`) | `name` trimmed 1–120 ("Name is required"); `sku` `optionalText(64)`; `categoryId` `string.min(1)`, nullable, optional, absent → `null`; `description` `optionalText(1000)`; `defaultPriceMinor` `minor`; `defaultCostMinor` `minor`, nullable, optional, absent → `null`; `archived` boolean, optional (absent = `false`) |
| `categoryInputSchema` (wire) | `name` trimmed 1–60 ("Name is required") |
| `customerRecordSchema` (wire, type `CustomerRecordInput`) | `name` trimmed 1–120 ("Name is required"); `email` `z.email("Enter a valid email")` optional nullable, or `""` → `null`; `phone` `optionalText(40)`; `notes` `optionalText(2000)` |
| `settingsInputSchema` (wire, type `SettingsInput`) | `currency` enum `CURRENCIES`; `locale` enum `LOCALES`; `dateFormat` enum `DATE_FORMATS`; `defaultRange` enum `RANGE_PRESETS` excluding `custom` |
| `productFormSchema` (form, type `ProductFormValues`) | `name` trimmed 1–120 ("Name is required"); `sku` max 64; `categoryId` string (empty = none); `description` max 1000; `price` `moneyString("price")` then "Price is too large"; `cost` `moneyString("cost", false)` |
| `customerFormSchema` (form, type `CustomerFormValues`) | `name` trimmed 1–120 ("Name is required"); `email` `""` or `z.email("Enter a valid email")`; `phone` max 40; `notes` max 2000 |

`productFormToInput(f)`: `name` trimmed; `sku`, `description` trimmed or `null`; `categoryId` or `null`;
`defaultPriceMinor = parseMoney(price) ?? 0`; `defaultCostMinor` = blank → `null`, else `parseMoney(cost)`. Customer
form values are posted unchanged; `customerRecordSchema` turns blank strings into `null`.

## 6. CSV rules (`src/lib/csv.ts`)

| Signature | Semantics |
| --- | --- |
| `safeCell(v: unknown): string \| number` | `null`/`undefined` → `""`. Numbers pass through unchanged (`-5` stays `-5`). Any other value is stringified; if it starts with `=`, `+`, `-`, `@`, tab or CR, prefix a single quote: `"=SUM(A1)" → "'=SUM(A1)"`, `"@cmd" → "'@cmd"`, `"-1+1" → "'-1+1"` |
| `toCsv(headers: string[], rows: unknown[][]): string` | Prefix a UTF-8 BOM (U+FEFF, so Excel detects UTF-8). Serialise with `Papa.unparse({ fields: headers, data })`, every data cell passed through `safeCell`, option `newline: "\r\n"`. Comma delimiter; a field is double-quoted when it contains a comma, double quote, CR or LF, or starts or ends with a space; inner quotes are doubled. No trailing newline |
| `csvResponse(filename: string, csv: string): Response` | Headers `Content-Type: text/csv; charset=utf-8`; `Content-Disposition: attachment; filename="<name>"` where every character outside `[A-Za-z0-9_.-]` becomes `_`; `Cache-Control: no-store` |
| `parseCsv(text: string): Record<string, string>[]` | Strip a leading BOM, parse with a header row and `skipEmptyLines: true`. Tests use it to round-trip exports |

Example: `toCsv(["Name", "Note"], [["Mug", 'say "hi", ok'], ["Pot", "x"]])` starts with `\uFEFFName,Note\r\n` and
contains `"say ""hi"", ok"`; `parseCsv` of it returns the two rows unchanged.

### 6.1 Export column sets (`src/server/services/exports.ts`)

Money cells are **major units** via `toMajorString(minor)` (e.g. `1234.50`, no symbol, no grouping); a `null` amount
is an empty cell. `<CUR>` is the currency code from settings. Timestamps are `yyyy-MM-dd HH:mm` in local time.

| Export | File name | Columns | Rows |
| --- | --- | --- | --- |
| Sales | `sales-<yyyy-MM-dd>.csv` (today) | Order, Date, Status, Payment method, Customer, Items, Units, Subtotal (<CUR>), Discount (<CUR>), Net (<CUR>), Tax (<CUR>), Total (<CUR>), Cost (<CUR>), Notes | All sales matching the list filters (same `SalesQuery`, ignoring `page`/`pageSize`), ordered by the list sort column and direction, then `occurred_at` desc. Status and Payment method use labels. Customer = snapshot name, or "Walk-in" when `null`. Items = `<product_name> x<quantity>` per line joined with `; `. Units = Σ quantity |
| Report timeseries | `report-timeseries.csv` | Period, Period start, Revenue (<CUR>), Sales, Units, AOV (<CUR>) | Dashboard `series` with the automatic bucket; Period = `formatBucketLabel`; Period start = bucket key |
| Report products | `report-products.csv` | Product, Revenue (<CUR>), Units, Sales | Top 10 product ranking |
| Report categories | `report-categories.csv` | Category, Revenue (<CUR>), Units, Share % | Category breakdown; Share % = `(revenue / total revenue × 100).toFixed(1)`, or `"0"` when total revenue is 0 |
| Report customers | `report-customers.csv` | Customer, Revenue (<CUR>), Sales | Top 10 customer ranking |
| Report payments | `report-payments.csv` | Payment method, Revenue (<CUR>), Sales | Payment breakdown, method labels |
| Products | `products.csv` | Name, SKU, Category, Default price (<CUR>), Default cost (<CUR>), Status, Units sold, Revenue (<CUR>) | All products including archived; Status "Active" or "Archived"; Units sold and Revenue from completed sales; active first, then name case-insensitive |
| Customers | `customers.csv` | Name, Email, Phone, Completed sales, Revenue (<CUR>), Customer since | All customers by name case-insensitive; Customer since = `created_at` as `yyyy-MM-dd` |

`REPORTS = ["timeseries", "products", "categories", "customers", "payments"] as const` (type `ReportName`). Reports
take a range validated by `rangeQuerySchema`. Headers are written literally, e.g. `Revenue (USD)`.

## 7. Metric definitions (`src/server/services/analytics.ts`)

Put this table as a block comment at the top of the analytics module. It is the single source of truth.

| Metric | Definition |
| --- | --- |
| Revenue | Σ `net_minor` (subtotal − discount, **tax excluded**) of **completed** sales whose `occurred_at` is in range |
| Sales count | Number of completed sales in range |
| AOV | `Math.round(revenue / salesCount)`; 0 when `salesCount = 0` |
| Units sold | Σ `sale_items.quantity` of completed sales |
| Cost | Σ `cost_minor` of completed sales (`null` ignored by the sum) |
| Costed count | Completed sales with non-null `cost_minor` |
| Net profit | `revenue − cost` only when `salesCount > 0` and `costedCount = salesCount`; otherwise `null` |
| Pending | Not yet realised. Excluded from every metric above; reported as Σ net and count |
| Refunded | Full refund of the whole sale. Excluded from revenue. No negative entry is ever written, so a refund cannot be subtracted twice. Reported as Σ net and count. Partial refunds are not modelled |
| Cancelled | Never happened. Excluded everywhere except the status breakdown and the cancelled count |
| Previous period | Same metrics over `previousRange(range)` |
| Change | `percentChange(current, previous)` |

Day grouping happens in SQL in local time: `date(occurred_at / 1000, 'unixepoch', 'localtime')`. Weekday uses
`CAST(strftime('%w', occurred_at / 1000, 'unixepoch', 'localtime') AS INTEGER)` (0 = Sunday).

### 7.1 Service functions

All take `(db: DB, r: DateRange, …)` and filter `occurred_at >= r.from AND occurred_at < r.to`.

| Function | Returns | Rules |
| --- | --- | --- |
| `kpis(db, r): Kpis` | `{ revenueMinor, salesCount, aovMinor, units, costMinor, costedCount, profitMinor: number \| null, pendingMinor, pendingCount, refundedMinor, refundedCount, cancelledCount }` | One conditional-aggregate query over `sales` plus one for units; every sum `coalesce`s to 0 |
| `series(db, r, bucket): SeriesPoint[]` | `{ key, revenueMinor, orders, units, aovMinor }[]` | Completed only. Aggregate per local day in SQL, roll up with `bucketKeyForDay`, zero-fill in `bucketKeys` order. `aovMinor = Math.round(revenue / orders)`, 0 when no orders |
| `productRanking(db, r, limit = 10)` | `{ productId, name, revenueMinor, units, orders }[]` | Completed only. Group by `coalesce(product_id, product_name)`; name from the line snapshot; revenue = Σ `line_net_minor`; orders = distinct sales; order by revenue desc, units desc |
| `categoryBreakdown(db, r)` | `{ name, revenueMinor, units }[]` | Completed only. Group by snapshot `category_name`, `null` → "Uncategorized"; revenue desc |
| `customerRanking(db, r, limit = 10)` | `{ customerId, name, revenueMinor, orders }[]` | Completed only. All walk-ins (`customer_id` null) form one row named "Walk-in customers"; registered customers use their current name, falling back to the sale snapshot; revenue = Σ sale `net_minor`; revenue desc |
| `paymentBreakdown(db, r)` | `{ method, revenueMinor, orders }[]` | Completed only; revenue desc |
| `statusBreakdown(db, r)` | `{ status, netMinor, orders }[]` | **All** statuses present in range |
| `weekdayBreakdown(db, r)` | 7 rows `{ dow, revenueMinor, orders }` | Completed only; `dow` 0–6, Sunday first; missing days zero-filled |
| `dashboard(db, range, bucketOverride?)` | `Dashboard` | 7.2 |

### 7.2 Dashboard composition

`dashboard` returns `{ range, previousRange, bucket, kpis: { current, previous }, series, previousSeries, daily, products, categories, customers, payments, statuses, weekdays, insights }`:

- `bucket = bucketOverride ?? autoBucket(range)`.
- `series` and `previousSeries` use that bucket over the range and the previous range.
- `daily` is the day-bucket series of the range (equal to `series` when the bucket is `day`).
- `products` and `customers` are the top 10.
- Export `type Dashboard = ReturnType<typeof dashboard>` for the client.

### 7.3 Insights

`MIN_SALES_FOR_INSIGHTS = 3`. When the current completed `salesCount` is below 3, `insights` is `[]` and the UI shows
"Not enough data yet" with "Insights appear once there are at least 3 completed sales in the period."

`Insight = { id: string; title: string; value: string | number; detail?: string; tone?: "positive" | "negative" | "neutral"; money?: boolean }`.
Money values travel as minor units and the client formats them. Otherwise emit, in this order:

| id | Condition | title | value | detail | tone |
| --- | --- | --- | --- | --- | --- |
| `best-product` | `products[0]` exists | "Best-selling product" | name of the top product by revenue | `"<units> units"` | none (`money: false`) |
| `top-category` | `categories[0]` exists | "Top category" | category name | `"<share>% of revenue"`, share = `Math.round(catRevenue / revenue × 100)`, 0 when revenue is 0 | none |
| `best-day` | the daily point with the highest revenue (first wins ties) has revenue > 0 | "Best day" | its `yyyy-MM-dd` key; the client renders it with the date-format setting | revenue in minor units as a string, with `money: true` | none |
| `trend` | `daily.length >= 4` and the change is not `null` | "Trend within period" | "Up <n>%" when change ≥ 0, else "Down <n>%"; `n = Math.abs(change).toFixed(0)` | "Second half vs first half" | > 2 positive, < −2 negative, else neutral |
| `vs-prev` | always | "Vs previous period" | "No prior sales" when the change is `null`, else sign + `change.toFixed(1)` + "%", sign `+` when ≥ 0 (so "+0.0%") | "Revenue change" | `null` or 0 neutral, > 0 positive, < 0 negative |

Trend change: `half = floor(daily.length / 2)`; `a` = Σ revenue of the first `half` days; `b` = Σ revenue of the last
`half` days (the middle day is skipped when the length is odd); change = `percentChange(b, a)`. Vs-prev change =
`percentChange(current.revenueMinor, previous.revenueMinor)`.

### 7.4 Display rules tied to metrics

- KPI change badge: `percentChange(current, previous)` shown with one decimal and `+` only when > 0 (`+12.3%`,
  `-4.0%`); when `null`, show "No prior-period data".
- Net profit card: when `profitMinor` is `null`, value "—" with hint "<N> sales missing cost" where
  `N = salesCount − costedCount`, or "No completed sales" when `salesCount = 0`. When known, hint "Revenue minus
  product cost"; the comparison uses `previous.profitMinor ?? 0`.
- Note line under the KPIs, shown only when the pending or refunded count is > 0: "Revenue counts completed sales
  only (net of discounts, excluding tax)." followed by "<money> pending in <n> sales." when pending > 0 and
  "<money> refunded across <n> sales." when refunded > 0.

### 7.5 Worked metric example

Range has four sales: A completed, net 10000, cost 4000, 2 units; B completed, net 5000, cost `null`, 1 unit;
C pending, net 3000; D refunded, net 2000. The previous period has one completed sale, net 6000.

| Metric | Value |
| --- | --- |
| Revenue | 15000 |
| Sales count | 2 |
| AOV | 7500 |
| Units | 3 |
| Cost / costed count | 4000 / 1 |
| Net profit | `null` (card hint "1 sales missing cost") |
| Pending | 3000 in 1 sale |
| Refunded | 2000 in 1 sale |
| Revenue change | 150 (badge "+150.0%") |
| Insights | `[]` (only 2 completed sales) |
