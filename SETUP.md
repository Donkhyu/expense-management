# Setup

Personal finance tracker in Notion, structured after the **Finance Tracker** layout: money
movements are split across three databases — **Expenses**, **Incomes**, **Transfers** —
which feed **Accounts** (live balances), **Categories** (per-category monthly budgets) and
**Months** (monthly totals). Logging is via iOS Shortcuts → Notion (see **`SHORTCUT.md`**),
plus monthly reconciliation against the bank statement.

> **Secrets:** never commit your Notion integration token. Store it only in the Shortcut on
> your phone. The only true secret is `<YOUR_TOKEN>`; the database/page IDs below are not
> secret (you still need the token to use them).

### Reference — this workspace's IDs

| Placeholder            | Value                                                              |
| ---------------------- | ------------------------------------------------------------------ |
| `<EXPENSES_DB_ID>`     | `7f0939c9cdf94904ae2d06b8a7d5160a` (🧾 Expenses database)          |
| `<INCOMES_DB_ID>`      | `f132396ae1934038a094d9b944e9720c` (💵 Incomes database)           |
| `<TRANSFERS_DB_ID>`    | `290c437a6bb045b09e886f79b9e0ba89` (🔁 Transfers database)         |
| `<CATEGORIES_DB_ID>`   | `9598b4c01bd64e25b3fe2f3e98f08127` (🗂️ Categories database)        |
| `<MONTHS_DB_ID>`       | `01df7527292c496999c1c1127ac1af39` (📅 Months database)            |
| `<ACCOUNTS_DB_ID>`     | `834988dff9744f8ebf9f590eb38fcab1` (🏦 Accounts database)          |
| `<BANK_ACCOUNT_ID>`    | `afc516a99cdf4dc9b16d08a11ccc2ace` (🏦 Bank balance — Shortcut default) |
| `<RYT_ACCOUNT_ID>`     | `379c4ed7327b81759736d7f2c97d87fa` (🐷 Ryt Bank (Savings))         |
| `<GOALS_DB_ID>`        | `c52f5e42b36248a69b7de903d2d09e55` (🏆 Savings Goals database)     |
| `<RECURRING_DB_ID>`    | `4409c590ebb84edc9f73c809ec482c3c` (🔁 Recurring database)         |
| `<NETWORTH_DB_ID>`     | `6f4475655a3445d48af77c5c6846211e` (📈 Net Worth database)         |
| `<DASHBOARD_PAGE_ID>`  | `37ac4ed7327b81c3b247ef68c93848b8` (💰 Finance dashboard)          |
| `<YOUR_TOKEN>`         | **secret** — store on-device only                                 |

Category row IDs (for the Shortcut's `Category` relation):

| Category | Page ID |
| --- | --- |
| Food | `37ac4ed7327b81739afafe3f06554a01` |
| Transport | `37ac4ed7327b81d898dcce506b54a166` |
| Groceries | `37ac4ed7327b81db8d71f20999a7edca` |
| Bills | `37ac4ed7327b8110b9f3deaff7017823` |
| Entertainment | `37ac4ed7327b81b99919ffef9d357820` |
| Shopping | `37ac4ed7327b8186a8f3d24635b376b6` |
| Health | `37ac4ed7327b81b5885ecb120ef45b7f` |
| Other | `37ac4ed7327b81b9bccfeb50d139c1dd` |

Everything lives under the **💰 Expense Management** page; the day-to-day dashboard is the
**💰 Finance** sub-page (linked views of every database).

---

## Architecture

Six interlinked databases. A money movement is **one row in exactly one** of Expenses /
Incomes / Transfers. Those feed the balances, budgets and monthly totals automatically.

```
Expenses ─┐                    ┌─ Categories (Expense This Month → Usage %)
Incomes  ─┼─→ Accounts (Balance)│
Transfers─┘                    └─ Months (Expense / Income totals)
```

### 🧾 Expenses

| Property | Type | Notes |
| --- | --- | --- |
| `Expense` | Title | What it was / merchant |
| `Amount` | Number (RM) | Positive |
| `Date` | Date | |
| `Account` | Relation → Accounts | Which account paid. Shortcut defaults to Bank |
| `Category` | Relation → Categories | Drives the category budget |
| `Month` | Relation → Months | Links the row to its month (set manually or by template) |
| `Expense This Month` | Formula | `Amount` if `Date` is in the current month, else `0` — the budget engine |

### 💵 Incomes

| Property | Type | Notes |
| --- | --- | --- |
| `Income` | Title | Source description |
| `Amount` | Number (RM) | Positive |
| `Date` | Date | |
| `Account` | Relation → Accounts | Where it landed |
| `Month` | Relation → Months | |
| `Source` | Select | Salary / Bonus / Interest / Gift / Other |

### 🔁 Transfers

One row per transfer (not two). It moves money **out of** one account and **into** another.

| Property | Type | Notes |
| --- | --- | --- |
| `Transfer` | Title | Description |
| `Amount` | Number (RM) | Positive |
| `Date` | Date | |
| `From Account` | Relation → Accounts | Money leaves here (counts as `TransferOut`) |
| `To Account` | Relation → Accounts | Money arrives here (counts as `TransferIn`) |

### 🏦 Accounts

| Property | Type | Notes |
| --- | --- | --- |
| `Account` | Title | Account name |
| `Starting balance` | Number (RM) | The account's opening/real-current balance |
| `Expense Items` / `Income Items` / `Transfers Out` / `Transfers In` | Relations | Reverse sides of the movement DBs |
| `Total Expenses` / `Total Income` / `Total TransferOut` / `Total TransferIn` | Rollups | Sum of `Amount` over each relation |
| `Balance` | Formula | `Starting balance + Total Income + Total TransferIn − Total Expenses − Total TransferOut` |

**How balances work:** every movement row points at an account; the four rollups re-total it
and `Balance` recomputes instantly. **Total net worth** = sum of every account's `Balance`
(set the `Balance` column footer to **Sum**). Accounts: `🏦 Bank balance` (Shortcut default)
and `🐷 Ryt Bank (Savings)`, opening balance RM100.

### 🗂️ Categories

| Property | Type | Notes |
| --- | --- | --- |
| `Category` | Title | Food, Transport, Groceries, Bills, Entertainment, Shopping, Health, Other |
| `Monthly Budget` | Number (RM) | Your cap — **set per row** |
| `Expenses` | Relation | Reverse of `Expenses.Category` |
| `Expense This Month` | Rollup | Sum of `Expenses.Expense This Month` (current month only) |
| `Usage` | Formula | `Expense This Month ÷ Monthly Budget` (0–1) — show as a **bar** in the UI |

Budgets reset automatically: `Expense This Month` only counts current-month expenses, so on
the 1st every category falls back to `0` with no rollover.

### 📅 Months

| Property | Type | Notes |
| --- | --- | --- |
| `Name` | Title | e.g. `June 2026` |
| `Month Start` | Date | First of the month (sort / `This Month`) |
| `Expense` / `Income` | Rollups | Totals of the Expenses / Incomes linked to this month |
| `This Month` | Formula | `true` when `Month Start` is the current month |

One row per month (`June 2026` seeded). Link each expense/income's `Month` relation to roll
its amount into that month's totals.

---

## Extra modules (beyond the base template)

These three are ours, kept from the previous build — the stock Finance Tracker doesn't have them.

- **🏆 Savings Goals** — `Target amount`, `Start/Target date`, `Saved` (rolls up the
  **Transfers** tagged to the goal via the `Funding` relation / `Goal` on Transfers), plus
  `Progress`, `Remaining`, `Monthly needed`, `Status`. Fund a goal by tagging a Transfer's
  `Goal`. *(Show `Progress` as a bar in the UI.)*
- **🔁 Recurring** — reference list of subscriptions/bills with `Monthly equivalent` &
  `Annual cost`. Paying one still needs a real Expenses row.
- **📈 Net Worth** — manual monthly snapshots (`Total`) + a `Trend` line chart.

---

## 💰 Finance dashboard

The **💰 Finance** page holds linked views of every database: **🏦 Accounts**, **🗂️ Categories
& Budgets**, **🧾 Expenses**, **💵 Incomes**, **🔁 Transfers**, **📅 Months**.

UI-only finishes the API can't do:
- Add quick-add **buttons** (New Expense / New Income / New Transfer) at the top.
- Arrange the views into columns.
- Set `Usage` (Categories) and `Progress` (Goals) display to **Bar**.
- Set each account's `Starting balance` to its **real current balance** (Bank is at a
  placeholder RM880 — the old per-transaction history was deleted, so correct it).

---

## Integration & Shortcut

### Create the integration
1. https://www.notion.so/my-integrations → **New integration** → Internal, name `Expense Shortcut`.
2. Capabilities: Insert + Update + Read content.
3. Copy the token (`secret_…` / `ntn_…`); store on-device only.

### Connect the databases
Open the **💰 Expense Management** page → `⋯` → **Connections** → add **Expense Shortcut**.
Connecting the parent cascades to the child databases (Expenses, Accounts, Categories, …).
The Shortcut writes to **Expenses** and references rows in **Accounts** and **Categories**,
so all three must be shared or the relation returns `object_not_found`.

### Logging
The iOS Shortcut creates a row in the **Expenses** DB — title `Expense`, plus `Amount`,
`Date`, an `Account` relation (hardcoded to Bank) and a `Category` relation (mapped from the
chosen category name to its page ID via a Dictionary). Full build + JSON: **`SHORTCUT.md`**.
The `Month` relation is set manually (or via a Notion template button) — it only affects the
Months DB totals, not budgets.

---

## Day-to-day workflows

**Add an expense** → Shortcut (or **New** in Expenses). Set `Category`; optionally link `Month`.

**Add income** → a row in **Incomes** with `Amount` (positive), `Account`, `Source`. It lifts
that account's `Balance` via `Total Income`.

**Transfer between accounts** → one row in **Transfers**: `From Account`, `To Account`,
`Amount`. Source `Balance` drops, destination rises, net worth unchanged. To credit a savings
goal, set the transfer's `Goal`.

**Budgets** → set `Monthly Budget` per row in **Categories**; `Usage` tracks the current
month and resets on the 1st. Link each expense's `Category` (the Shortcut does this) so it counts.

**Monthly reconciliation** → once a month, skim the bank statement; log anything missing.
Because `Balance` is driven entirely by logged rows, drift from the real balance means a
movement was missed. Adjust an account's `Starting balance` when you reconcile.
