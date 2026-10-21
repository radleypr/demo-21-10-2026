# Ledgerline — Product Specification & Build Plan

> **What this repository is.** The complete specification and build plan for **Ledgerline**, a local-first sales
> dashboard.

## 1. Executive Summary

Ledgerline is a production-quality, single-user sales dashboard that runs entirely on one machine. You record sales,
products and customers, and it shows revenue, trends, rankings and insights. Data lives in a local SQLite file
(`data/sales.db`). There are no accounts, no cloud and no server to provision, and it needs no configuration:
`pnpm install && pnpm dev` is the whole setup.

The specs fix every decision that affects behaviour: stack and versions, data model, money and metric rules, API
contract, UI structure, design tokens, copy, and test cases. Nothing should need to be guessed. Where the specs
leave room (internal helper names, markup details), any choice is fine as long as every requirement and test passes.

## 2. What gets built

| Area | Capabilities |
| --- | --- |
| **Overview** | KPI cards (revenue, sales, AOV, units, net profit) with prior-period deltas and sparklines; revenue area chart with previous-period overlay; volume, product and category charts; insights; recent sales |
| **Sales** | server-side table: search, filters, sorting, paging, bulk delete, view/edit/duplicate/delete, CSV export of the filtered set |
| **Sale sheet** | add/edit/duplicate with catalog or new products, walk-in/existing/new customers, discount, tax, cost, live totals, inline validation |
| **Products & categories** | CRUD, archive/restore, SKU uniqueness, protected deletes, CSV |
| **Customers** | CRUD, purchase-history sheet, revenue and order counts, CSV |
| **Analytics** | trend metrics with chart-type switch, rankings, distributions, weekday pattern, top customers, status table, report CSVs |
| **Settings** | currency/locale/date format/default range/theme, DB location, online backups, demo data load/remove, erase-all |
| **Platform** | Next.js 16 App Router, React 19, TypeScript, Tailwind 4, shadcn/ui, TanStack Query/Table, Zustand, RHF + Zod, Recharts, Drizzle + better-sqlite3 |

The full functional specification is in [`docs/spec/01-Product-Spec.md`](docs/spec/01-Product-Spec.md).

## 3. Getting started

The public repository is the **starting guide**: it holds the specification and build plan, not the app. Create your
own **private working repository** from it, then build the app there, at the repository root and alongside these
documents. Keep the spec and the app in that one repository; do not create a separate repository for the code. The
public upstream stays documentation-only.

1. Install the [prerequisites](#6-prerequisites).
2. Create a private working copy from this repository, then clone it:
   ```bash
   git clone https://github.com/<your-account>/<your-repo>.git ledgerline
   cd ledgerline
   export SPEC="$(pwd)"
   ```
   GitHub does not let you turn a public fork private, so either use **Import repository** to create a private copy,
   or clone the fork and push it to a new private repository. `SPEC` points at the same checkout you are building
   in: the specs and the app share one repository, and every path in the specs resolves from this root.
3. Read [`CONTRIBUTING.md`](CONTRIBUTING.md): reading order, conflict rule and implementation rules.
4. Work through [`BUILD_PLAN.md`](BUILD_PLAN.md) phase by phase from the repository root. Every phase ends with a
   **Gate** (commands and expected results); do not start the next phase until the gate passes.
5. Finish with the **Final acceptance** checklist at the end of `BUILD_PLAN.md` and write the build report described
   in `CONTRIBUTING.md`.

To resume interrupted work, re-run the gates in order and continue from the first phase whose gate does not pass.

**Upstream vs your working copy.** `MoonLabLtd/sales-dashboard-app` remains the documentation-only starting guide.
Application code, configuration and migrations are committed to your private working repository only; do not open
pull requests with app code against the public upstream.

**Effort and size.** The finished app is roughly 145 files: about 7–8k lines of application code, about 4k lines of
generated shadcn/ui primitives and about 1.6k lines of tests (61 Vitest cases, 5 Playwright tests). Dependencies take
about 1 GB.

## 4. Repository layout

The specification lives under `docs/spec/`. In your working repository the app is created beside it, at the repository
root, so the root of the finished repository holds both the documents below and the app's `package.json`, `src/`,
`scripts/`, `tests/`, `drizzle/` and configuration files from
[`02-Environment-and-Stack.md`](docs/spec/02-Environment-and-Stack.md) §5. The spec files under `docs/spec/` and the
governance documents stay in place; Phase 8 replaces only the root `README.md` with the app's user guide.

| Path | Purpose |
| --- | --- |
| `README.md` | Overview, getting started, prerequisites |
| `CONTRIBUTING.md` | Reading order, conflict rule, implementation rules, build report |
| `ARCHITECTURE.md` | Solution architecture: principles, layers, data spine, decisions (ADRs) |
| `BUILD_PLAN.md` | **The build plan.** Ordered phases with gates and the final acceptance checklist |
| `docs/spec/README.md` | Spec index: what each spec covers, which phase reads it, where to find a given fact |
| `docs/spec/01-Product-Spec.md` | Pages, flows, copy, functional (FR) and non-functional (NFR) requirements |
| `docs/spec/02-Environment-and-Stack.md` | Exact versions, bootstrap commands, every config file, project layout |
| `docs/spec/03-Domain-Rules.md` | Money, tax/discount math, statuses, metrics, date ranges, validation, CSV |
| `docs/spec/04-Data-Model.md` | Tables, constraints, indexes, DB path, migrations, backup/restore |
| `docs/spec/05-Backend-and-API.md` | Services, error model, the full HTTP API contract, demo seed, CLI scripts |
| `docs/spec/06-Frontend-and-Design.md` | Design tokens, shell, data layer, components, charts, accessibility |
| `docs/spec/07-Testing.md` | The test plan: every test case to implement, with expected values |
| `docs/spec/08-Operations.md` | Commands, environment variables, troubleshooting |
| `docs/spec/09-Gotchas.md` | Known pitfalls of this stack and how to avoid them. **Read before coding** |
| `docs/spec/10-Requirements-Traceability.md` | Every requirement → module → test or gate |

## 5. Success criteria

The build is done when all of these hold (details in `BUILD_PLAN.md` → Final acceptance):

| Metric | Target |
| --- | --- |
| Unit + integration tests | every case in [`07-Testing.md`](docs/spec/07-Testing.md) implemented; `pnpm check` exit 0 |
| Browser tests | `pnpm test:e2e`: all e2e cases pass |
| Production build | `pnpm build` exit 0, and the build does not create `data/` |
| Zero-config first run | deleting `data/` then `pnpm dev` creates and migrates the DB; empty state shown; no demo data |
| Metric correctness | the worked examples in [`03-Domain-Rules.md`](docs/spec/03-Domain-Rules.md) hold in the running app |
| Traceability | every FR/NFR in [`10-Requirements-Traceability.md`](docs/spec/10-Requirements-Traceability.md) is implemented and proved |

## 6. Prerequisites

Install these before starting the build.

| Requirement | Version | Why |
| --- | --- | --- |
| Git | any recent | create and clone a private working repository; the app is built there |
| Node.js | **24 LTS** (exactly major 24; the app's `engines` is `>=24 <25`) | runtime; `better-sqlite3` ships prebuilt binaries for it |
| pnpm | **11.x** (the app pins `pnpm@11.22.0`) | package manager; enabled through Corepack |
| C/C++ build tools | only if no prebuilt `better-sqlite3` binary matches your OS/CPU | compile the native SQLite driver |
| Internet access | during setup | `pnpm install` and a one-time Playwright Chromium download (~200 MB) |
| Disk space | ~2 GB free | `node_modules` ~1 GB, `.next` build ~0.5 GB, Chromium ~0.2 GB |
| Free ports | 3000 (dev), 3200 (e2e tests); the plan's gates also use 3100 and 3300 | local servers |

You do **not** need a database server, Docker, accounts, API keys or environment variables.

### 6.1 Install

**macOS** (Homebrew):

```bash
xcode-select --install          # build tools (skip if already installed)
brew install node@24 git
export PATH="$(brew --prefix node@24)/bin:$PATH"   # add to ~/.zshrc so every shell uses Node 24
corepack enable
```

**Linux** (via nvm):

```bash
sudo apt-get install -y git build-essential python3   # Debian/Ubuntu; use your distro's equivalent
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
nvm install 24 && nvm alias default 24
corepack enable
```

**Windows**: install Node.js 24 LTS from [nodejs.org](https://nodejs.org/) (or `winget install OpenJS.NodeJS.LTS`)
and Git for Windows. Then, in an admin terminal, run `corepack enable`. If `pnpm install` tries to compile
`better-sqlite3`, install "Visual Studio Build Tools" with the "Desktop development with C++" workload. WSL2 with the
Linux steps also works.

### 6.2 Verify

```bash
node -v        # v24.x.x
corepack pnpm -v   # 11.x  (or: pnpm -v)
git --version
```

If `node -v` shows a version other than 24, fix your `PATH` / nvm default **before** starting. A
different Node major is the most common cause of "Could not locate the bindings file" errors from
`better-sqlite3`. Every shell and tool that runs the build must see Node 24.
