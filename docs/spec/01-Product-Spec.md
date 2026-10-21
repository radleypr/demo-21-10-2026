# 01 — Product Specification

**Ledgerline** is a local-first, single-user sales dashboard. One person runs it on their own machine to record
sales, manage a small product catalog and customer list, and understand revenue, profit and trends. There is no
authentication, no cloud service, no FX conversion and no inventory tracking. All data lives in one SQLite file.

This document defines **what** the app must do: every page, control, message, default and edge case. Metric
formulas live in [03-Domain-Rules.md](03-Domain-Rules.md), the schema in [04-Data-Model.md](04-Data-Model.md), the
HTTP contract in [05-Backend-and-API.md](05-Backend-and-API.md), and visual design in
[06-Frontend-and-Design.md](06-Frontend-and-Design.md). Every requirement ID below is mapped to a test or check in
[10-Requirements-Traceability.md](10-Requirements-Traceability.md).

Conventions:

- Quoted strings in "double quotes" are exact UI copy. Use them character for character (including "…", "—",
  "–", "−", "·" and curly quotes where shown).
- `fmt.money` = currency formatting with the user's currency and locale; `fmt.number` = locale number
  formatting; `fmt.date` = the user's date format; `fmt.dateTime` = date format followed by ", HH:mm".
- "Toast" = a Sonner toast, bottom-right. "Success toast" / "Error toast" use `toast.success` / `toast.error`.
  When a description is listed, it is the toast's second line.

## 1. Scope

| In scope                                                         | Out of scope                                       |
| ---------------------------------------------------------------- | -------------------------------------------------- |
| Record, edit, duplicate, delete sales with multiple line items   | Multi-user, login, roles                           |
| Product catalog with categories, archive/restore                 | Inventory / stock levels                           |
| Optional customer records with purchase history                  | Partial refunds (refunds are always full)          |
| KPIs, charts, insights, rankings for any date range              | Currency conversion (currency is display-only)     |
| CSV exports, online backups, demo data, erase-all                | Remote access, sync, telemetry, external APIs      |
| Light/dark/system themes, keyboard and screen-reader support     | Zero- or three-decimal currencies (JPY, KWD, …)    |

## 2. Shared constants

| Constant          | Values (display label)                                                                                    |
| ----------------- | --------------------------------------------------------------------------------------------------------- |
| Sale status       | `completed` (Completed), `pending` (Pending), `cancelled` (Cancelled), `refunded` (Refunded)              |
| Payment method    | `card` (Card), `cash` (Cash), `bank_transfer` (Bank transfer), `e_wallet` (E-wallet), `other` (Other)     |
| Currency          | USD, EUR, GBP, CAD, AUD, NZD, SGD, CHF, IDR, INR (all two minor digits)                                    |
| Locale            | en-US, en-GB, en-AU, de-DE, fr-FR, es-ES, id-ID, ja-JP                                                     |
| Date format       | `MMM d, yyyy`, `d MMM yyyy`, `yyyy-MM-dd`, `MM/dd/yyyy`, `dd/MM/yyyy` (date-fns tokens)                  |
| Date presets      | `today` (Today), `7d` (Last 7 days), `30d` (Last 30 days), `thisMonth` (This month), `prevMonth` (Previous month), `thisYear` (This year), `custom` (Custom range) |
| Sales page sizes  | 10, 25, 50, 100 (default 25)                                                                               |
| Limits            | quantity 1–100,000; unit price/cost/discount 0–10,000,000.00; tax rate 0–100% (two decimals); 1–50 line items per sale |
| Default settings  | currency USD, locale en-US, date format `MMM d, yyyy`, default range `30d`, theme System                  |

Status badges: Completed = success tone, Pending = warning tone, Cancelled = muted, Refunded = destructive tone;
each badge has a small leading dot in the current colour.

## 3. App shell

### 3.1 Layout and navigation

| ID    | Requirement |
| ----- | ----------- |
| FR-01 | The root layout wraps every page in the shell: sidebar, sticky top bar, a single `<main id="main">` (max width 1400px, centred), one app-wide sale sheet (§5) and the toaster. Pages must not render their own `<main>`. |
| FR-02 | A "Skip to content" link is the first focusable element; it is visually hidden until focused and targets `#main`. |
| FR-03 | Sidebar (`aria-label="Main navigation"`) shows a brand header (a square "L" mark in the primary colour plus the word "Ledgerline", linking to `/`), then these items in this order with these lucide icons: Overview `/` (LayoutDashboard), Sales `/sales` (Receipt), Products `/products` (Package), Customers `/customers` (Users), Analytics `/analytics` (BarChart3), Settings `/settings` (Settings). |
| FR-04 | The active item is the one whose href equals the path for `/`, or whose href is a prefix of the path otherwise; it is highlighted and carries `aria-current="page"`. Unknown paths fall back to Overview. |
| FR-05 | The sidebar footer reads "Local data · single user" (hidden when collapsed). |
| FR-06 | Desktop (≥ 768px): the sidebar collapses to an icon rail (`collapsible="icon"`), with a tooltip of each label when collapsed, a drag rail, and the Ctrl/⌘+B shortcut. The open/collapsed state persists in the browser across reloads (default open). |
| FR-07 | Mobile (< 768px): the sidebar is an off-canvas drawer (18rem wide) opened by the top-bar toggle. Choosing any nav item or the brand link closes the drawer. |
| FR-08 | Browser tab title: "Ledgerline" on Overview; "<Page> · Ledgerline" elsewhere (Sales, Products, Customers, Analytics, Settings). |

### 3.2 Top bar

| ID    | Requirement |
| ----- | ----------- |
| FR-09 | Sticky, 56px high, translucent blurred background, bottom border. Left to right: sidebar toggle (`aria-label="Toggle navigation"`), vertical separator, the current page label as the page's only `<h1>`. Right group: date range picker (FR-11), search button, theme toggle, Add sale button. |
| FR-10 | Search button: ≥ 1024px shows an outline button "Search…" with a "⌘K" key hint; below 1024px an icon-only button with `aria-label="Search"`. Both open the command palette (§3.4). |
| FR-11 | The global date range picker appears only on Overview and Analytics. ≥ 640px it sits in the top bar; below 640px it moves to a full-width second row under the header, and those two pages add top padding so content is not covered. |
| FR-12 | Theme toggle: icon button `aria-label="Change theme"` (sun in light, moon in dark) opening a radio menu Light / Dark / System. Theme is stored in the browser (default System) and switches without transition flash. |
| FR-13 | "Add sale" primary button with a plus icon. Below 640px the label is visually hidden (still in the accessible name). Opens the sale sheet in add mode. |

### 3.3 Global date range

| ID    | Requirement |
| ----- | ----------- |
| FR-14 | The trigger is an outline button with a calendar icon, the label, and a chevron; `aria-label="Date range: <label>"`. The label is the preset label, or for custom ranges "<fmt.date(from)> – <fmt.date(last day)>". |
| FR-15 | The popover shows a preset list (all presets except Custom; the active one highlighted) and a two-month range calendar. Clicking a preset applies it and closes. In the calendar, days after today are disabled; "Apply range" (disabled until a start day is picked) applies a custom range (a single picked day means a one-day range); "Cancel" closes without change. Opening the popover pre-selects the current custom range, if any. |
| FR-16 | Preset semantics (local time, half-open `[from, to)`): Today = today; Last 7/30 days = the last 7/30 calendar days including today; This month = 1st of month to end of today; Previous month = full previous calendar month; This year = 1 Jan to end of today; Custom = inclusive calendar days, swapped if reversed. |
| FR-17 | The user's selection persists in the browser. When there is no stored selection, the default range from Settings applies. Changing the default range in Settings clears the stored selection so the new default takes effect immediately. |
| FR-18 | "Today" is re-evaluated at least once per minute so a dashboard left open across midnight rolls over. |
| FR-19 | Every comparison uses the **previous period of equal length** (same number of calendar days) immediately before the range. |

### 3.4 Command palette

| ID    | Requirement |
| ----- | ----------- |
| FR-20 | ⌘K (macOS) or Ctrl+K toggles the palette from anywhere; the browser default is prevented. Dialog title "Search", description "Jump to a page, find a sale or run an action". Input placeholder "Search pages, order IDs, products, customers…". Empty result text "No results.". |
| FR-21 | When the trimmed input has ≥ 2 characters, a "Sales" group shows first: an item "Search all sales for “<term>”" that navigates to `/sales?q=<term>`, then up to 6 matching sales (order ref in mono, first product name, " · <customer>" when present, total right-aligned). Selecting a sale opens it in the sale sheet (edit mode). |
| FR-22 | An "Actions" group with "Add sale" (opens the sale sheet), and a "Pages" group listing the six nav items with their icons. |
| FR-23 | Running any item closes the palette and clears the input. |

## 4. Overview (`/`)

Uses the global date range. All data comes from one dashboard request plus one recent-sales request.

### 4.1 KPI cards

| ID    | Requirement |
| ----- | ----------- |
| FR-24 | A "Key metrics" section of five cards in this order: "Total revenue", "Sales", "Avg. order value", "Units sold", "Net profit". Grid: 1 column, 2 at ≥ 640px, 3 at ≥ 1024px, 5 at ≥ 1280px. While loading, five skeleton cards. |
| FR-25 | Each card shows the label, the value (money values via `fmt.money`, counts via `fmt.number`), and a change badge vs the previous period: arrow up-right with success tone when change > +0.05%, arrow down-right with destructive tone when < −0.05%, a flat dash with muted tone otherwise; text "+12.3%" / "-4.0%" (one decimal, explicit "+" for positive). Hidden text "vs previous period" follows for screen readers; the line has the tooltip "Compared with the previous period of equal length". |
| FR-26 | When the previous value is 0, no badge is shown; instead the line reads "No prior-period data" (or the card's hint, see FR-28). |
| FR-27 | Revenue, Sales, Avg. order value and Units sold show a 72×28px sparkline of the current-period series (same buckets as the main chart) when the series has ≥ 2 points. Net profit has no sparkline. |
| FR-28 | Net profit: when profit is known (every completed sale in the period has full cost data), shows the value and change, with hint "Revenue minus product cost" (shown only when there is no prior-period value). Otherwise the value is "—" and the hint line reads "<N> sales missing cost", or "No completed sales" when the period has no completed sales. |
| FR-29 | Under the cards, when the period has pending or refunded sales, a small note: "Revenue counts completed sales only (net of discounts, excluding tax)." followed by "<money> pending in <n> sales." (if any pending) and "<money> refunded across <n> sales." (if any refunded). |

### 4.2 Charts and insights

| ID    | Requirement |
| ----- | ----------- |
| FR-30 | When the period has **no sales of any status**, all charts and insights are replaced by one card with the empty state "No sales in this period" / "Record a sale or pick a wider date range. You can also load demo data from Settings." and two buttons: "Add sale" (opens the sheet) and "Open settings" (links to `/settings`). The Recent sales card still renders below. |
| FR-31 | "Revenue over time" (2/3 width at ≥ 1024px): area chart of completed revenue per bucket, 288px tall, with the previous period overlaid as a dashed muted line aligned by bucket index. Description "By <day|week|month>, dashed line is the previous period". Buckets: day when the range is ≤ 31 days, week (Monday start) when ≤ 182 days, month otherwise. Every bucket in the range appears, zero-filled. X labels: day "MMM d", week "Wk of MMM d", month "MMM yyyy". Y axis uses compact money. Tooltip shows "Revenue" and "Previous period" with full money values. |
| FR-32 | "Insights" card (1/3 width), description "What stands out this period". A definition list of rows (label left, value right, optional detail line below the value). When there are fewer than 3 completed sales: empty state with a lightbulb icon, "Not enough data yet" / "Insights appear once there are at least 3 completed sales in the period." |
| FR-33 | Insight rows, in order, each only when its condition holds: (1) "Best-selling product" — top product by completed revenue (ties by units), detail "<units> units"; (2) "Top category" — top category by revenue, detail "<share>% of revenue" (share rounded to whole percent); (3) "Best day" — the calendar day with the highest completed revenue, shown with `fmt.date`, detail its revenue in money; only when that revenue > 0; (4) "Trend within period" — only when the period has ≥ 4 days and the first half has revenue: compare the sum of the last ⌊n/2⌋ days with the first ⌊n/2⌋ days; value "Up 12%" / "Down 7%" (whole percent), detail "Second half vs first half", tone positive when change > 2%, negative when < −2%, neutral otherwise; (5) "Vs previous period" — always: "+8.4%" / "-3.1%" (one decimal) or "No prior sales" when previous revenue is 0, detail "Revenue change", tone by sign. Positive tone = success colour with a trending-up icon; negative = destructive colour with trending-down icon. |
| FR-34 | Row of three cards (stacked below 1024px): "Sales volume" — bar chart of completed sales count per bucket, description "Completed sales per <bucket>"; "Revenue by product" — horizontal bar chart of the top 6 products, description "Top 6 products"; "Sales by category" — donut, description "Share of revenue". The latter two show the small empty state "No completed sales" when there is nothing to plot. |
| FR-35 | Donut charts (here and on Analytics): inner radius 62%, centre text = compact total plus a caption ("Revenue" on Overview, "Total" elsewhere), and a legend listing each slice with colour swatch, name and percentage of total (one decimal). More than 6 slices collapse into the top 5 plus "Other". Slice colours cycle through chart-1…chart-5 then the muted chart colour. |
| FR-36 | Horizontal ranking charts truncate labels longer than 18 characters to 17 + "…" and show full values in the tooltip. |

### 4.3 Recent sales

| ID    | Requirement |
| ----- | ----------- |
| FR-37 | "Recent sales" card, description "Latest transactions in this period, any status", header action "View all" → `/sales`. Shows the 8 most recent sales in the range (date descending), any status. |
| FR-38 | Columns: Date (`fmt.date`), Order (mono), Product (first product name, then " +N" when there are more lines), Customer (hidden < 768px; "Walk-in" in muted text when none), Qty (units), Total, Payment (hidden < 1024px), Status (badge), row actions (§6.4). |
| FR-39 | States: 5 skeleton rows while loading; error state with retry; empty state "No sales in this period". |

## 5. Sale sheet (add / edit / duplicate)

One right-side sheet instance serves the whole app (opened from the top bar, palette, tables, customer history).
Full width on mobile, max 36rem (`sm:max-w-xl`) on larger screens. The body scrolls; the totals panel and
footer stay pinned at the bottom.

### 5.1 Modes

| Mode      | Title            | Description                                                         | Initial values                                                                                         | Submit label    |
| --------- | ---------------- | ------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ | --------------- |
| Add       | "Add sale"       | "Record a transaction. Amounts are in your reporting currency."     | Blank form (5.2 defaults)                                                                              | "Record sale"   |
| Edit      | "Edit sale"      | "Changes update every report that includes this sale."              | All fields from the sale                                                                               | "Save changes"  |
| Duplicate | "Duplicate sale" | "Record a transaction. Amounts are in your reporting currency."     | Fields from the source sale, but date/time = now, order reference blank, status = Completed            | "Record sale"   |

| ID    | Requirement |
| ----- | ----------- |
| FR-40 | Edit and Duplicate load the source sale first; meanwhile three skeleton blocks show. If loading fails, the server message is shown in destructive text instead of the form. |
| FR-41 | Prefill mapping: lines with a product link use "Catalog product" mode with that product; lines without one use "New product" mode with the stored product and category name. A sale with a customer link uses "Existing" with that customer; otherwise "Walk-in" with the stored name on receipt. Prices, costs and discount show as plain "1234.50" strings (cost blank when unknown; discount blank when zero); tax rate shows without trailing zeros ("7.25", "10", "0"). |
| FR-42 | The form fully resets whenever the sheet opens for a different sale or mode. Closing via Cancel, Escape or the overlay discards changes. |

### 5.2 Fields (in visual order)

| Field                | Control                                                                          | Default            | Validation (inline message)                                                                                     |
| -------------------- | -------------------------------------------------------------------------------- | ------------------ | ---------------------------------------------------------------------------------------------------------------- |
| Date & time          | `datetime-local`, label "Date & time"                                             | now (local, minute) | required format → "Enter a valid date and time". Interpreted in the browser's time zone.                         |
| Order reference      | text, label "Order reference", placeholder "Auto-generated if blank"              | blank              | max 64. Blank → the server assigns `ORD-` + 8 uppercase hex chars. Must be unique (server, see 5.4).           |
| Items                | fieldset legend "Items"; one bordered group per line, `aria-label="Item <n>"`     | one empty line     | at least one line → "Add at least one item"; max 50 ("Add item" disabled at 50)                                  |
| ↳ Product source     | toggle "Catalog product" / "New product" (`aria-label="Product source"`)          | Catalog product    | —                                                                                                                |
| ↳ Product (catalog)  | select, label "Product", placeholder "Select a product" ("Loading products…" while loading) | none   | required → "Select a product"                                                                                     |
| ↳ Product name (new) | text, label "Product name"                                                        | blank              | required → "Product name is required"; max 120. Saving creates a catalog product with this line's price/cost as defaults. |
| ↳ Category (new)     | text, label "Category", placeholder "e.g. Coffee"                                 | blank              | optional, max 60. Matched to an existing category by name, or created.                                           |
| ↳ Qty                | text (`inputMode=numeric`), label "Qty"                                           | "1"                | whole number 1–100000 → "Quantity must be a whole number from 1 to 100,000"                                      |
| ↳ Unit price         | text (`inputMode=decimal`), label "Unit price", placeholder "0.00"                | blank              | required money → "Enter a valid price"; > 10,000,000.00 → "Price is too large"                                   |
| ↳ Unit cost          | text (`inputMode=decimal`), label "Unit cost", placeholder "Optional"             | blank              | optional money → "Enter a valid cost or leave blank". Blank = unknown cost.                                      |
| Customer             | fieldset legend "Customer"; toggle "Walk-in" / "Existing" / "New" (`aria-label="Customer type"`) | Walk-in | —                                                                                                    |
| ↳ Walk-in            | text, label "Name on receipt (optional)", placeholder "Walk-in customer"          | blank              | max 120                                                                                                          |
| ↳ Existing           | select (visually-hidden label "Customer"), placeholder "Select a customer" ("Loading customers…") | none | required → "Select a customer"                                                                       |
| ↳ New                | text "Name" + email "Email (optional)"                                            | blank              | name required → "Customer name is required" (max 120); email blank or valid → "Enter a valid email"              |
| Payment method       | select, label "Payment method"                                                    | Card               | one of the five methods                                                                                          |
| Status               | select, label "Status"                                                            | Completed          | one of the four statuses                                                                                         |
| Discount             | money input with the currency code as prefix addon, label "Discount", placeholder "0.00" | blank (= 0) | optional money → "Enter a valid discount or leave blank"; > subtotal → "Discount cannot exceed the subtotal" |
| Tax rate             | text with "%" suffix addon, label "Tax rate"                                      | "0"                | 0–100 with ≤ 2 decimals (blank = 0) → "Tax rate must be between 0 and 100%"                                      |
| Notes                | textarea (2 rows), label "Notes"                                                  | blank              | max 2000                                                                                                         |

Money inputs accept digits with an optional "." and up to two decimals; thousands commas are ignored ("1,234.5"
→ 1234.50). Negative numbers, letters and three decimals are invalid.

| ID    | Requirement |
| ----- | ----------- |
| FR-43 | Picking a catalog product fills that line's unit price with the product's default price (and re-validates it) and its unit cost with the default cost, or blank when the product has no cost. Switching a line to "New product" keeps qty/price/cost. |
| FR-44 | The catalog select lists active products only, plus the currently selected product even if archived (labelled "<name> (archived)"). When there are no options it shows "No products yet — choose “New product”." |
| FR-45 | Each line has a remove button (trash icon, `aria-label="Remove item <n>"`) only when there is more than one line. "Add item" (outline, plus icon) appends an empty line. |
| FR-46 | Layout: date/ref, payment/status and discount/tax pairs are two columns ≥ 640px; qty/price/cost are always three columns; new-product name/category and new-customer name/email are two columns ≥ 640px. |
| FR-47 | Validation runs on blur and then on change (`mode: "onTouched"`), and on submit for all fields. Invalid fields get `aria-invalid` and a message below. The server re-validates everything with the same rules. |

### 5.3 Live totals

| ID    | Requirement |
| ----- | ----------- |
| FR-48 | A panel pinned above the footer (`aria-live="polite"`) shows "Subtotal", "Discount" (as "−<money>"), "Tax", and "Total" (bold, larger), updating on every keystroke. |
| FR-49 | Preview rules: invalid or blank quantities/prices count as 0; the discount is clamped to the subtotal; the tax rate is clamped to 100%; tax = round half up of (subtotal − discount) × rate; total = subtotal − discount + tax. The preview must equal what the server stores for valid input (see [03-Domain-Rules.md](03-Domain-Rules.md)). |

### 5.4 Submit

| ID    | Requirement |
| ----- | ----------- |
| FR-50 | Footer: "Cancel" (ghost, closes) and the submit button. While saving, the submit button is disabled and shows a spinner; a second submit while pending is ignored. |
| FR-51 | Add/Duplicate create a new sale; Edit replaces the sale (header and all lines) in one transaction. New customers and new products/categories typed in the form are created in the same transaction. |
| FR-52 | On success: toast "Sale recorded" (add/duplicate) or "Sale updated" (edit) with description "<orderRef> · <fmt.money(total)>"; every data query except settings is invalidated; the sheet closes. |
| FR-53 | On failure: toast "Could not save sale" with the server message as description (e.g. 'Order reference "X" is already used', "Selected product no longer exists", "Selected customer no longer exists"). The sheet stays open with the user's input. |

## 6. Sales (`/sales`)

### 6.1 Toolbar and filters

| ID    | Requirement |
| ----- | ----------- |
| FR-54 | Search input (search icon addon, `aria-label="Search sales"`, placeholder "Order ID, product or customer"), debounced 250ms, trimmed, max 100 characters. Matches order reference, receipt/customer name, or any line's product name, case-insensitive substring. "%" and "_" are matched literally. |
| FR-55 | `?q=<term>` in the URL pre-fills the search; later changes to `q` (from the palette) replace the search text. |
| FR-56 | Filters, in order: date select (`aria-label="Date range"`): "All time" (default) plus every preset except Custom, resolved at selection time; "Status" multi-select; "Payment" multi-select (checkbox menus that stay open, showing a count badge when any are selected); category select (`aria-label="Category"`, "All categories" + categories); product select (`aria-label="Product"`, "All products" + all products including archived). A product or category filter matches sales with at least one such line. |
| FR-57 | "Clear filters" (ghost, X icon) appears when any search or filter is active and resets all of them. |
| FR-58 | Right side: "Export CSV" (outline, download icon) and, below 640px only, an "Add" button opening the sale sheet. |
| FR-59 | Any change to search or filters resets to the first page and clears the row selection. |

### 6.2 Table

| ID    | Requirement |
| ----- | ----------- |
| FR-60 | Columns: select checkbox; Date (`fmt.dateTime`, sortable); Order (mono); Product (first product, then " +N more" muted; "—" if none; truncated); Customer ("Walk-in" muted when none; truncated); Qty (units, right, sortable); Total (right, sortable); Payment; Status (badge plus a small uppercase "Demo" tag for demo rows; sortable); actions. |
| FR-61 | Sorting is server-side by `date`, `amount` (total incl. tax), `quantity` (units) or `status` (alphabetical key). Default date descending. First click on a column sorts descending, then toggles; sorting cannot be removed. Ties break by date descending, then id descending. Changing sort returns to page 1. Sort headers show an up/down/neutral arrow and set `aria-sort` on the header cell. |
| FR-62 | Pagination is server-side. Footer `nav` (`aria-label="Pagination"`): "<first>–<last> of <total>" (total locale-formatted), "Rows" select (`aria-label="Rows per page"`, 10/25/50/100; the choice persists in the browser and resets to page 1), "Page <n> of <pages>", previous/next icon buttons (`aria-label="Previous page"` / `"Next page"`) disabled at the ends. The footer is hidden when there are no results. |
| FR-63 | While refetching, the previous page stays visible at reduced opacity and the table has `aria-busy`. Initial load shows 8 skeleton rows. Errors show the error state with "Try again". |
| FR-64 | Empty states: no filters → "No sales yet" / "Record your first sale, or load demo data from Settings." with "Add sale"; with filters → "No sales match these filters" / "Try removing a filter or widening the date range." with "Clear filters". |

### 6.3 Selection and bulk delete

| ID    | Requirement |
| ----- | ----------- |
| FR-65 | Header checkbox `aria-label="Select all on this page"` (checked, indeterminate or unchecked for the current page); row checkboxes `aria-label="Select <orderRef>"`. Selected rows are highlighted. |
| FR-66 | When ≥ 1 row is selected, a status bar (`role="status"`) shows "<n> selected", "Clear selection" and a destructive "Delete selected". |
| FR-67 | Confirm dialog "Delete <n> sales?" / "These sales and their line items are removed from every report. This cannot be undone." buttons "Cancel" / "Delete sales". Success toast "<deleted> sales deleted" and selection cleared; error toast "Could not delete sales" + message. |

### 6.4 Row actions (Sales table and Overview recent sales)

| ID    | Requirement |
| ----- | ----------- |
| FR-68 | Icon button (`aria-label="Actions for <orderRef>"`) opening a menu: "View", "Edit", "Duplicate", separator, "Delete" (destructive). |
| FR-69 | View opens a dialog: title = order ref (mono); description "<fmt.dateTime> · <customer name or 'Walk-in customer'>"; status badge, payment label and a "Demo" tag when applicable; a lines table (Item with category below in muted text, Qty, Price, Line subtotal); then Subtotal, Discount ("−<money>"), "Tax (<rate>%)", Total; then notes if any. Skeleton while loading. |
| FR-70 | Edit / Duplicate open the sale sheet in that mode (§5). |
| FR-71 | Delete opens a confirm: "Delete sale <orderRef>?" / "This permanently removes the sale and its line items from every report. To keep history, mark it Cancelled or Refunded instead." buttons "Keep sale" / "Delete sale". Success toast "Sale <orderRef> deleted"; error toast "Could not delete sale" + message. |

### 6.5 Export

| ID    | Requirement |
| ----- | ----------- |
| FR-72 | "Export CSV" downloads exactly the currently filtered and sorted set (all pages, not just the visible page) as `sales-<yyyy-MM-dd>.csv`. Column layout is defined in [05-Backend-and-API.md](05-Backend-and-API.md). |

## 7. Products (`/products`)

Two tabs: "Products" (Package icon, default) and "Categories" (Tags icon).

### 7.1 Products tab

| ID    | Requirement |
| ----- | ----------- |
| FR-73 | Toolbar: search (`aria-label="Search products"`, placeholder "Name or SKU", debounced 250ms, substring on name or SKU); "Show archived" checkbox (default off); right side "Export CSV" (downloads `products.csv`, all products) and "Add product". |
| FR-74 | Columns: Product (name, "Archived" outline badge, "Demo" tag); SKU (mono, "—" when none, hidden < 768px); Category ("Uncategorized" muted when none); Price (default price); Cost (hidden < 1024px, "—" when unknown); Units sold; Revenue; actions. Units and revenue count completed sales only, all time. Archived rows use muted text. |
| FR-75 | Sort: active before archived, then name case-insensitive ascending. |
| FR-76 | Footnote: "Units and revenue count completed sales only. Sales keep a snapshot of the product name, price and cost, so editing a product never rewrites history." |
| FR-77 | Row actions (`aria-label="Actions for <name>"`): "Edit"; "Archive" / "Restore" (toast "<name> archived" / "<name> restored"; error toast with the message); separator; "Delete" (destructive). Archived products disappear from the sale sheet's catalog select but keep their history. |
| FR-78 | Delete confirm: "Delete <name>?" / "Products that appear in sales cannot be deleted; archive them instead to hide them from new sales." button "Delete product". Success toast "Product deleted". If the product has sales the server answers 409 and the error toast "Could not delete product" shows "This product appears in past sales. Archive it instead to keep history intact." |
| FR-79 | Add/Edit dialog: title "Add product" / "Edit product", description "Default price and cost prefill new sales; past sales are unchanged." Fields: "Name" (autofocus, required "Name is required", max 120); "SKU" (placeholder "Optional", max 64, unique: 'SKU "<sku>" is already used by another product'); "Category" select ("Uncategorized" + categories sorted by name); "Default price" (required money: "Enter a valid price" / "Price is too large"); "Unit cost" (placeholder "Optional", "Enter a valid cost or leave blank"); "Description" (textarea, max 1000). Buttons "Cancel" / "Add product" or "Save changes". Toasts "Product added" / "Product updated"; error "Could not save product" + message. |
| FR-80 | Empty states: no search → "No products yet" / "Add products to your catalog, or create them on the fly while recording a sale." with "Add product"; with search → "No products match your search". Loading: 6 skeleton rows. Error: error state with retry. |

### 7.2 Categories tab

| ID    | Requirement |
| ----- | ----------- |
| FR-81 | Add form: input (`aria-label` and placeholder "New category name", max 60) and "Add" button disabled when blank or saving. Success toast "Category added" and the input clears. Duplicate name: error toast 'Category "<name>" already exists'. |
| FR-82 | List (sorted by name, case-insensitive): name, "<n> product" / "<n> products", rename icon button (`aria-label="Rename <name>"`), delete icon button (`aria-label="Delete <name>"`). |
| FR-83 | Inline rename: the row becomes an input (`aria-label="Category name"`, autofocus, max 60) with "Save" (disabled when blank) and "Cancel"; Escape cancels. Toast "Category renamed"; duplicate names fail with the same message as FR-81. |
| FR-84 | Delete confirm: "Delete category <name>?" / "Only categories with no products can be deleted. Move or delete its products first." button "Delete category". Toast "Category deleted"; when in use, error toast "Could not delete category" with "This category is used by <n> product(s). Reassign them first." |
| FR-85 | Empty: "No categories yet" / "Categories group products in reports." Footnote: "Renaming a category does not change the category recorded on past sales." |

## 8. Customers (`/customers`)

| ID    | Requirement |
| ----- | ----------- |
| FR-86 | Toolbar: search (`aria-label="Search customers"`, placeholder "Name, email or phone", debounced 250ms, substring on name, email or phone); right side "Export CSV" (`customers.csv`) and "Add customer". |
| FR-87 | Columns: Customer (name as a button that opens the history sheet, plus "Demo" tag); Email (hidden < 768px, "—"); Phone (hidden < 1024px, "—"); Orders; Revenue; Last sale (hidden < 640px, `fmt.date` or "—"); actions. Orders and revenue count completed sales (net, excluding tax); Last sale is the latest sale of any status. Sorted by name case-insensitive. Footnote "Orders and revenue count completed sales only." Loading: 6 skeleton rows; error state with retry. |
| FR-88 | Row actions (`aria-label="Actions for <name>"`): "View history", "Edit", separator, "Delete" (destructive). |
| FR-89 | History sheet (right, max 32rem): title = name; description = email and phone joined by " · ", or "Purchase history". Three stat tiles "Revenue", "Orders", "Avg. order" ("—" when no orders) computed from completed sales in the list; notes block when present; then the sale list (newest first, at most 200): order ref, "<fmt.date> · <payment label>", status badge, total. Clicking a row opens that sale in the sale sheet (edit). No sales: "No sales recorded for this customer yet." Footnote "Revenue excludes tax and counts completed sales only. Showing up to 200 most recent sales." |
| FR-90 | Add/Edit dialog: "Add customer" / "Edit customer", description "Contact details are optional and stay on this machine." Fields "Name" (autofocus, "Name is required", max 120), "Email" (placeholder "Optional", "Enter a valid email"), "Phone" (`type=tel`, "Optional", max 40), "Notes" (3 rows, max 2000). Buttons "Cancel" / "Add customer" or "Save changes". Toasts "Customer added" / "Customer updated"; error "Could not save customer". |
| FR-91 | Delete confirm: "Delete <name>?" / "Customers with recorded sales cannot be deleted, so their history stays intact." button "Delete customer". Toast "Customer deleted"; when blocked, error toast "Could not delete customer" with "This customer has <n> sale(s). Customers with history cannot be deleted." |
| FR-92 | Empty states: "No customers yet" / "Customers are optional. Walk-in sales are recorded without a customer record." with "Add customer"; with search "No customers match your search". |

## 9. Analytics (`/analytics`)

Uses the global date range. All revenue figures are completed sales, net of discounts, excluding tax.

| ID    | Requirement |
| ----- | ----------- |
| FR-93 | Header row: note "Revenue = completed sales, net of discounts, excluding tax. All figures in <currency code>." and an "Export report" menu (download icon; disabled until the range is known) with label "CSV for the selected range" and items "Revenue over time", "Product ranking", "Categories", "Customers", "Payment methods" downloading `report-<id>.csv` (ids `timeseries`, `products`, `categories`, `customers`, `payments`) for the current range. |
| FR-94 | "Trends" card, description "Dashed line shows the previous period of equal length.". Controls: metric tabs "Revenue" / "Sales" / "Avg. order" / "Units" (default Revenue; session-only); grouping select (`aria-label="Grouping"`) "Auto" / "Daily" / "Weekly" / "Monthly" (default Auto = FR-31 rule; session-only); chart type toggle (`aria-label="Chart type"`) area / bar / line icon buttons (`aria-label` "Area chart" / "Bar chart" / "Line chart"; persisted in the browser; default area). Chart 288px tall with previous-period overlay (dashed for area/line; muted bars side by side for bar). Money metrics use money formatting; counts use number formatting. |
| FR-95 | "Product ranking": toggle "Revenue" / "Units" (default Revenue); horizontal bars of the top 10 products by the chosen measure; description "Top products by revenue" / "Top products by units sold". |
| FR-96 | "Revenue by category": donut, description "Share of completed revenue". |
| FR-97 | "Payment methods": donut of revenue by method (display labels), description "Revenue by how customers paid". |
| FR-98 | "Day of week": horizontal bars of completed revenue per weekday in local time, always seven rows Mon, Tue, Wed, Thu, Fri, Sat, Sun (zero-filled), description "Revenue by weekday (local time)". |
| FR-99 | "Top customers" (full width), description "Walk-in sales are grouped together": table Customer, Orders, Avg. order, Revenue; top 10 by revenue; all sales without a customer link form one row "Walk-in customers" in muted text. |
| FR-100 | "Sales by status" (full width), description "Only completed sales count toward revenue. Refunds are full refunds and remove the sale from revenue.": table Status (badge), Sales, Net amount — one row per status present in the range. |
| FR-101 | Charts with nothing to plot show "No completed sales in this period." A failed dashboard request replaces the page with the error state; loading shows skeletons sized like each chart. |

## 10. Settings (`/settings`)

Single column, max width 48rem. Three cards in order: Preferences, Data & storage, Export.

### 10.1 Preferences

| ID     | Requirement |
| ------ | ----------- |
| FR-102 | Card "Preferences", description "Changes are saved immediately and apply across the app." Each row: label, optional help text, select (full width on mobile, 14rem from 640px). Selects are disabled while a save is in flight. 5 skeleton rows while loading; error state with retry. |
| FR-103 | "Reporting currency" — the 10 currency codes; help "Amounts are displayed in this currency. Existing values are not converted — no exchange rates are applied." |
| FR-104 | "Number locale" — the 8 locales; help "Controls number and currency formatting, e.g. <1234.5 formatted as currency in the current locale and currency>". |
| FR-105 | "Date format" — options labelled with a sample date (14 March 2026, 15:30) rendered in each format. |
| FR-106 | "Default date range" — all presets except Custom; help "Used when the app opens." Saving it also resets the current global selection (FR-17). |
| FR-107 | "Theme" — Light / Dark / System; help "Stored in this browser." Changes apply instantly without a server call and without a toast. |
| FR-108 | Every server-backed change saves the full settings object immediately, updates the cached settings (all money/date formatting re-renders), and shows toast "Settings saved"; on failure an error toast with the message. |

### 10.2 Data & storage

| ID     | Requirement |
| ------ | ----------- |
| FR-109 | Card "Data & storage", description "Everything is stored locally in a single SQLite file. Nothing leaves this machine." A definition list: "Database file" (absolute path, mono, wraps), "Backups folder" (absolute path), "Records" ("<n> sales · <n> products · <n> customers · <n> categories"). |
| FR-110 | Backups section, heading "Backups", text "Uses SQLite's online backup API, so it is safe while the app is running.", button "Back up now" (database-backup icon). Success toast "Backup created: <file>"; the list refreshes. Up to 5 most recent backups (newest first): file name (mono), "<size in KB, no decimals> KB · <fmt.dateTime>". Instruction text: "To restore, stop the app and run `pnpm db:restore <backup-file>`. The current database is saved to `backups/pre-restore` first." |
| FR-111 | Demo data section, heading "Demo data". With no demo rows: text "Load about a year of sample sales to explore the dashboard. Demo rows are tagged and removable." and button "Load demo data" (sparkles icon) which runs immediately; toast "Demo data added". With demo rows: text "<n> demo sales are loaded. Demo rows are tagged and can be removed without touching your own records." and button "Remove demo data" which opens a confirm "Remove demo data?" / "All demo sales are deleted. Demo products or customers used by your own sales are kept." button "Remove demo data"; toast "Demo data removed". Demo data is never loaded automatically. |
| FR-112 | Danger zone (destructive-tinted border), heading "Erase all data", text "Deletes every sale, product, customer and category. Settings are kept. Back up first.", destructive button "Erase all data". The confirm dialog "Erase all data?" / "This permanently deletes all sales, products, customers and categories. It cannot be undone." contains an input labelled "Type DELETE ALL DATA to confirm" (`autocomplete=off`). The "Erase everything" button stays disabled until the input equals `DELETE ALL DATA` exactly (case-sensitive). Success toast "All data erased"; closing the dialog clears the input. The server independently rejects any other phrase with 'Type "DELETE ALL DATA" to confirm'. |
| FR-113 | All actions in this card are disabled while any of them is running; each success invalidates every data query and refreshes the counts; errors show an error toast with the message. |

### 10.3 Export

| ID     | Requirement |
| ------ | ----------- |
| FR-114 | Card "Export", description "CSV files open in any spreadsheet. Filtered exports are available on the Sales and Analytics pages." Buttons "All sales", "Products", "Customers" (download icons) download the unfiltered CSVs. |

## 11. Shared states and behaviour

| ID     | Requirement |
| ------ | ----------- |
| FR-115 | Error state (any failed data view): warning-triangle icon, title "Something went wrong", description = server message (or "Something went wrong"), outline button "Try again" that refetches. |
| FR-116 | Route-level loading fallback: five KPI-sized skeletons and one 320px skeleton, `aria-busy="true"`, `aria-label="Loading"`. |
| FR-117 | After any successful write (sale, product, category, customer, demo, erase, backup), every data query except settings is invalidated, so all pages show fresh numbers without a reload. |
| FR-118 | Confirm dialogs are alert dialogs: Cancel closes, the destructive action is disabled while pending, Escape cancels. |
| FR-119 | Server validation errors return the first field message; the client shows it in an error toast. Requests failing without a JSON body show "Request failed (<status>)". |
| FR-120 | Demo rows are tagged everywhere they appear (Sales table, sale view dialog, Products, Customers) and demo sales use `DEMO-00001`-style order references. |

## 12. Non-functional requirements

| ID     | Area             | Requirement |
| ------ | ---------------- | ----------- |
| NFR-01 | Zero-config      | `pnpm install && pnpm dev` on a clean checkout runs the app with no environment variables; the first request creates `data/sales.db` and applies migrations. The app starts empty. |
| NFR-02 | Persistence      | All business data and server-side settings persist in one SQLite file (WAL mode) and survive restarts and new production builds. `pnpm build` never opens the database. Browser-only state (theme, date selection, sidebar state, chart type, page size) persists in localStorage/cookies. |
| NFR-03 | Integrity        | The server is the single source of truth; every write is validated server-side and runs in a transaction. Money is integer minor units end to end; no floating-point money in storage or aggregation. |
| NFR-04 | Privacy / local  | No network calls except to the app's own origin. No telemetry, analytics, CDNs or web fonts loaded at runtime (fonts are self-hosted). Single user, no auth; intended for localhost only. |
| NFR-05 | No FX            | Currency is a display setting only. Changing it never converts or rewrites stored amounts, and the UI says so (FR-103). |
| NFR-06 | Accessibility    | WCAG 2.2 AA: every control has a visible or accessible label; full keyboard operation (palette, menus, dialogs, sheet, tables, toggles); visible focus rings; `aria-invalid` and messages on invalid fields; one `h1` per page; one `main`; status/live regions for selection count and totals; colour is never the only status signal (badges carry text); reduced motion respected. |
| NFR-07 | Responsive       | Usable from 360px wide, verified at 390×844, with no horizontal page overflow (tables may scroll inside their card). Breakpoints: 640px (sm), 768px (md, mobile drawer), 1024px (lg), 1280px (xl). |
| NFR-08 | Themes           | Light and dark themes with system default; no flash of the wrong theme on load; charts and badges readable in both. |
| NFR-09 | Visual quality   | Restrained, premium analytics look (Geist Sans/Mono, cool neutrals, one indigo accent, 5-colour chart palette, tabular numbers for figures). Not a generic admin template. Details in [06-Frontend-and-Design.md](06-Frontend-and-Design.md). |
| NFR-10 | Performance      | With ~13 months of demo data, dashboard and sales-list requests should feel instant locally (target: under 300ms; a goal, not a gate); pagination, sorting and filtering are server-side; aggregates are computed in SQL; tables keep the previous page visible while fetching. |
| NFR-11 | Feedback         | Every data view has loading (skeleton), empty and error (retry) states. Every mutation gives a success or error toast. Double submits are impossible. |
| NFR-12 | Time             | All date bucketing and ranges use the local time zone of the machine (browser and server are the same machine). Periods are half-open `[from, to)`. |
| NFR-13 | Quality gates    | `pnpm check` (typecheck, lint, format check, unit and integration tests) and `pnpm test:e2e` pass; `pnpm build && pnpm start` works. See [07-Testing.md](07-Testing.md) and [08-Operations.md](08-Operations.md). |

## 13. Acceptance summary

1. From a clean clone, `pnpm install && pnpm dev` serves an empty, working dashboard at http://localhost:3000.
2. A sale recorded through the sheet appears in Sales, Overview KPIs/charts/recent sales, Products totals and
   Customers totals, and survives a reload.
3. Submitting an incomplete sale shows inline messages and creates nothing.
4. Below 768px the sidebar is a drawer; at 390px wide no page scrolls horizontally.
5. The metric rules in [03-Domain-Rules.md](03-Domain-Rules.md) hold for every figure on every page.
6. Every FR and NFR above is covered in [10-Requirements-Traceability.md](10-Requirements-Traceability.md). Build
   order is in [BUILD_PLAN.md](../../BUILD_PLAN.md), structure in [ARCHITECTURE.md](../../ARCHITECTURE.md), working rules in
   [CONTRIBUTING.md](../../CONTRIBUTING.md); pitfalls in [09-Gotchas.md](09-Gotchas.md); stack in
   [02-Environment-and-Stack.md](02-Environment-and-Stack.md).
