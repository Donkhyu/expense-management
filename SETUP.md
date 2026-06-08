# Setup

Personal expense logging via iOS Shortcuts → Notion. Two shortcuts:
1. **Manual** — tap after a purchase, type amount/merchant (Home Screen / Lock Screen / Action Button).
2. **Alert parser** — copy a Hong Leong Bank transaction alert and run the shortcut; it extracts amount + merchant for you. Build guide: **`SHORTCUT.md`**.

Every transaction links to one **account** (e.g. your main bank balance or Ryt Bank savings), and a per-row formula keeps each account's live running balance (starting balance ± everything that moved). Transfers between accounts net to zero across your total wealth. Plus monthly reconciliation against the bank statement for whatever the alert parser misses.

> **Secrets:** never commit your Notion integration token. Store it only in the Shortcut on your phone. The only true secret is `<YOUR_TOKEN>`; the database and account IDs below are not secret (you still need the token to use them).

### Reference — this workspace's IDs

| Placeholder         | Value                                                              |
| ------------------- | ------------------------------------------------------------------ |
| `<YOUR_DB_ID>`      | `36ec4ed7327b8083a0dec2161698e9ff` (Transactions database)         |
| `<ACCOUNT_PAGE_ID>` | `afc516a99cdf4dc9b16d08a11ccc2ace` (🏦 Bank balance account row — the Shortcut's default) |
| `<RYT_ACCOUNT_PAGE_ID>` | `379c4ed7327b81759736d7f2c97d87fa` (🐷 Ryt Bank (Savings) account row) |
| `<BUDGETS_DB_ID>`   | `14b07e1858844f1bb0dba4383c3ae29e` (Budgets database)              |
| `<GOALS_DB_ID>`     | `c52f5e42b36248a69b7de903d2d09e55` (🏆 Savings Goals database)      |
| `<YOUR_TOKEN>`      | **secret** — store on-device only                                  |

The databases live under a **💰 Expense Management** page, which also serves as a dashboard (auto-balance callout + the **🎯 Budgets** board and the Transactions database inline).

---

## Phase 1 — Notion

The setup uses **four databases**: `Transactions` (one row per money movement — expense, income, or a transfer leg), `Accounts` (one row per account you hold), `Budgets` (per-category monthly caps — see **Budgets**), and `Savings Goals` (targets you fund by tagging deposits — see **Savings Goals**). Transactions link to an account via a Relation, and a rollup + formula on each account row turn that into a live balance.

### `Transactions` database

| Property      | Type             | Notes                                                                      |
| ------------- | ---------------- | -------------------------------------------------------------------------- |
| `Merchant`    | Title            | Row's primary field                                                        |
| `Date`        | Date             |                                                                            |
| `Amount`      | Number           | Number format set to **Ringgit** (RM)                                      |
| `Category`    | Select           | Food, Transport, Groceries, Bills, Entertainment, Shopping, Health, Other  |
| `Account`     | Relation         | → `Accounts`. Which account the money moved through. Shortcut defaults to Bank balance |
| `Notes`       | Text             | Corrections, context                                                       |
| `Type`        | Select           | `Expense` / `Income` / `Transfer Out` / `Transfer In`. Blank counts as Expense. Drives `Delta`'s sign |
| `Auto-logged` | Checkbox         | Distinguishes auto entries from manual                                     |
| `Delta`       | Formula          | `if(Type == "Income" or Type == "Transfer In", Amount, -Amount)` — signed balance effect, summed by each account's rollup |
| `Budget`      | Relation         | → `Budgets`. Links a transaction to its category budget (see **Budgets**). Set manually |
| `Goal`        | Relation         | → `Savings Goals`. Tag a deposit to a savings goal so it counts toward that goal (see **Savings goals**). Set manually |
| `Month`       | Formula          | `formatDate(Date, "YYYY-MM")` — the row's month, used for grouping and the budget reset |
| `ThisMonthExpense` | Formula     | Amount if the row is a *current-month* **expense** (income and transfers count `0`), else `0`. Summed by the Budget rollup so `Spent` auto-resets each month |

Keep `Category` as a **Select** (keeps the Shortcut JSON simple). `Account` is a **Relation**, not a Select — that's what powers the auto-updating balance below.

### `Accounts` database

One row per account. Currently `🏦 Bank balance` (your main bank, the Shortcut's default) and `🐷 Ryt Bank (Savings)`.

| Property           | Type      | Notes                                                            |
| ------------------ | --------- | ---------------------------------------------------------------- |
| `Account`          | Title     | The account name                                                 |
| `Starting balance` | Number    | The account's balance at the moment you started logging it       |
| `Expenses`         | Relation  | Auto-created reverse side of `Transactions.Account` (Notion auto-named it) |
| `Net change`       | Rollup    | Sum of `Delta` across that account's transactions (income/transfer-in +, expense/transfer-out −) |
| `Current balance`  | Formula   | `Starting balance + Net change` → the account's live running balance |

**How auto-balance works:** each transaction's `Account` relation routes its `Delta` into that account's `Net change` rollup, and `Current balance` recalculates instantly — no manual updating. The same machinery scales to any number of accounts: each row totals only its own transactions. Your **total net worth** is the sum of every account's `Current balance` — set the `Current balance` column footer to **Sum** in the Accounts table to show it.

> **The Shortcut still defaults to `🏦 Bank balance`** (no account picker — keeps logging to one tap). For a purchase or transfer on another account, open the row afterward and switch `Account`. New here? Set `Starting balance` on the Ryt Bank row (created at `0`) to your real savings balance.

### `Budgets` database

One row per spending category, each holding a monthly cap. A rollup totals only the **current month's** expenses linked to it, so budgets reset automatically on the 1st with no manual rollover.

| Property        | Type     | Notes                                                                    |
| --------------- | -------- | ------------------------------------------------------------------------ |
| `Category`      | Title    | The category name, e.g. `Food`, `Transport`, `Shopping`                  |
| `Monthly limit` | Number   | Ringgit (RM). Your cap for the month — **set this by hand per row**       |
| `Transactions`  | Relation | Reverse side of `Transactions.Budget` — the rows counted toward this budget |
| `Spent`         | Rollup   | Sum of `ThisMonthExpense` across related transactions (this month only)  |
| `Remaining`     | Formula  | `Monthly limit - Spent` → headroom left this month                       |

**How the monthly reset works:** `Spent` rolls up `Transactions.ThisMonthExpense`, which is `Amount` only when the row is an expense dated in the current month and `0` otherwise. When the month turns over, every prior row evaluates to `0`, so `Spent` falls back to `0` and `Remaining` returns to the full limit — no archiving or duplicating budget rows each month.

> **Linking is manual.** A transaction only counts toward a budget once its `Budget` relation is set. New rows (manual button *or* Shortcut) start unlinked, so `Spent` undercounts until you link them — see **Budgets & monthly tracking** below for the worklist view that makes this a quick habit.

### `🏆 Savings Goals` database

One row per savings goal (Emergency Fund, a trip, a big purchase). Progress is **tag-based**: any transaction whose `Goal` relation points here counts toward the goal, so a goal can draw from any account and you decide which deposits "count."

| Property         | Type     | Notes                                                                       |
| ---------------- | -------- | --------------------------------------------------------------------------- |
| `Goal`           | Title    | The goal name                                                               |
| `Target amount`  | Number   | Ringgit (RM). What you're aiming for                                        |
| `Start date`     | Date     | When you started saving — anchors the on-track/behind pace check            |
| `Target date`    | Date     | Deadline — drives `Months left` and `Status`                                |
| `Priority`       | Select   | High / Medium / Low (the board groups by this)                              |
| `Contributions`  | Relation | Reverse side of `Transactions.Goal` — every deposit tagged to this goal     |
| `Saved`          | Rollup   | Sum of `Delta` across contributions (deposits add, withdrawals subtract)    |
| `Progress`       | Formula  | `Saved ÷ Target amount` (0–1). Show it as a **bar/percent** in the UI for a progress bar |
| `Remaining`      | Formula  | `Target amount − Saved`, clamped at `0`                                     |
| `Months left`    | Formula  | Whole months from now to `Target date`                                      |
| `Monthly needed` | Formula  | `Remaining ÷ Months left` — what to set aside each month to land on time    |
| `Status`         | Formula  | ✅ Reached / 🟢 On track / 🔴 Behind / ⏰ Overdue, judged against the time elapsed vs. the target window |

**How it works:** to fund a goal, log the deposit as a normal transaction — usually a `Transfer In` (or `Income`) into your savings account — and set its `Goal` relation. `Saved` rolls up the signed `Delta`, so a later `Transfer Out` tagged to the same goal correctly *reduces* the saved amount. Because contributions are just tagged transactions, transfers between your own accounts still net to zero on net worth while also advancing the goal.

The **🎯 Progress** board groups goals by `Priority` and shows `Status`, `Saved`, `Progress` and `Monthly needed` on each card. (Notion can't group a board by a formula, so `Status` lives on the card rather than as the column.)

> **Two manual touches:** ① set each deposit's `Goal` (same habit as `Budget` — the Shortcut doesn't set it). ② One-time, open the `Progress` property and switch its display to **Bar** (as a percentage) for the visual progress bar.

### Create the integration

1. https://www.notion.so/my-integrations → **New integration** → Internal type, name `Expense Shortcut`.
2. Capabilities: Insert + Update + Read content.
3. Copy the Internal Integration Token (`secret_...` or `ntn_...`). Store in iOS Passwords / 1Password.

### Connect the databases to the integration

The integration needs access to **both** databases: it writes to `Transactions`, and the
`Account` relation points at a row in `Accounts` — if `Accounts` isn't shared, the API
returns `object_not_found` on the relation even though the row exists.

Easiest: open the **💰 Expense Management** page → top-right `⋯` → **Connections** →
add **Expense Shortcut**. Connecting at the parent page cascades to both child
databases. (Or connect `Transactions` and `Accounts` individually.)

### Grab the IDs you'll need in the Shortcut

- **`<YOUR_DB_ID>`** — open the `Transactions` DB as a full page in the browser. URL is `notion.so/<workspace>/<32-char-id>?v=...`. The 32-char hex string is the DB ID.
- **`<ACCOUNT_PAGE_ID>`** — open the `🏦 Bank balance` account row (the Shortcut's default) as a full page. The 32-char hex string in its URL is the page ID. This is what the `Account` relation points to. Each additional account (e.g. `🐷 Ryt Bank (Savings)`) has its own page ID — see the *Reference — this workspace's IDs* table near the top.

---

## Phase 2 — Shortcut #1: Manual

In the Shortcuts app, name it **Log Expense**.

> **Magic Variables:** every action's output is a "Magic Variable" — a coloured pill you drop into later actions. To insert one, tap inside a field and pick from the chip row above the keyboard. Rename each action's output (tap the pill at the top of the action) so it reads `Amount`, `Merchant`, etc. instead of `Provided Input`.

### Action sequence

1. **Ask for Input** — Prompt `Amount?`, **Input Type: Number**. Rename output → `Amount`.
2. **Ask for Input** — Prompt `Merchant?`, Input Type: Text. Rename output → `Merchant`.
3. **Choose from List** — items: `Food`, `Transport`, `Groceries`, `Bills`, `Entertainment`, `Shopping`, `Health`, `Other`. Rename output → `Category`.
   - **Dictionary** — map each category to an emoji (`Food : 🍔`, `Transport : 🚕`, `Groceries : 🛒`, `Bills : 💡`, `Entertainment : 🎬`, `Shopping : 🛍️`, `Health : 💊`, `Other : 📦`), then **Get Dictionary Value** → *Value* for *Key* = `Category` → rename → `Icon`. This becomes the row's page icon.
4. **Current Date**.
5. **Format Date** — Date Format: **Custom**, Format String: `yyyy-MM-dd`. Feed Current Date in. Rename output → `Today`. (Without this step the date arrives as `29/05/2026, 12:00`, which Notion rejects — it needs ISO 8601.)
6. **Text** — paste the JSON below, then replace each `[bracketed]` placeholder by inserting the matching Magic Variable. **Amount stays unquoted; everything else stays inside its quotes.** `Account` has no chooser — it's hardcoded to your one account, so paste your `<ACCOUNT_PAGE_ID>` once and forget it.

   ```json
   {
     "parent": { "database_id": "<YOUR_DB_ID>" },
     "icon": { "emoji": "[Icon]" },
     "properties": {
       "Merchant": {
         "title": [ { "text": { "content": "[Merchant]" } } ]
       },
       "Amount": { "number": [Amount] },
       "Date": { "date": { "start": "[Today]" } },
       "Category": { "select": { "name": "[Category]" } },
       "Account": { "relation": [ { "id": "<ACCOUNT_PAGE_ID>" } ] },
       "Type": { "select": { "name": "Expense" } },
       "Auto-logged": { "checkbox": false }
     }
   }
   ```

   > The `Account` relation takes the account **page** ID (with or without dashes), not the database ID. Linking it is what makes `Current balance` drop by the amount you just logged.

   > ⚠️ **Smart-quote trap.** iOS auto-replaces `"` with curly `" "`, which is not valid JSON. Before pasting: **Settings → General → Keyboard → turn off Smart Punctuation**. If you've already pasted, retype every `"` after disabling it.

7. **Get Contents of URL** — URL `https://api.notion.com/v1/pages`. Expand Show More:
   - **Method:** `POST`
   - **Headers:**
     - `Authorization` → `Bearer <YOUR_TOKEN>`
     - `Notion-Version` → `2022-06-28`
     - `Content-Type` → `application/json`
   - **Request Body:** **File** → insert the Text action's Magic Variable. (If File misbehaves, switch to **JSON** with the same Text variable as input.)
8. **Show Notification** — `Logged RM[Amount] at [Merchant]`.

### Test and debug

Run the shortcut with ▶︎. On success, a new row appears in Notion within ~1s and `Current balance` on the account drops by the amount.

If it fails, drop a **Quick Look** action between the Text action and `Get Contents of URL`, run again, and inspect the actual body. Common Notion responses and what they mean:

| Response | Cause |
| --- | --- |
| `invalid_json` with `"number": ,` | `Amount` Magic Variable didn't insert. Re-add it; confirm step 1 has Input Type Number. |
| `invalid_json` with curly `"…"` | Smart Punctuation is on. Disable and retype quotes. |
| `validation_error` mentioning `start` | Date is `29/05/2026, …` not `2026-05-29`. The `Format Date` step is missing or you inserted Current Date instead of `Today`. |
| `unauthorized` | Token wrong, or DB isn't connected to the integration (DB → ⋯ → Connections). |
| `object_not_found` | `database_id` typo, or DB not shared with the integration. |
| `validation_error` on the `Account` relation | The `<ACCOUNT_PAGE_ID>` is wrong, or it's the database ID instead of the account row's page ID. |
| `validation_error` on the `Category` select | The value doesn't exactly match an option in Notion (case-sensitive). |

### Place the shortcut

- **Home Screen:** long-press in the Shortcuts app → Share → Add to Home Screen.
- **Lock Screen widget:** Customize Lock Screen → Add Widget → Shortcuts → pick `Log Expense`.
- **Action Button** (iPhone 15 Pro / 16+): Settings → Action Button → Shortcut → `Log Expense`. Biggest QoL win — one press after paying.
- **Siri:** "Hey Siri, log expense" works automatically by name.

### Alternative: build the JSON with the Dictionary action

The Text-action approach above is recommended because the body is readable in one screen. If you prefer the typed Dictionary action, build this structure — at the top level use two keys of type `Dictionary` (`parent`, `properties`); inside `properties`, mirror the JSON above. `Merchant.title` requires an `Array` whose single item is a Dictionary, and `Account.relation` is an `Array` whose single item is a Dictionary with an `id` key. It works, it's just slow to enter and hard to debug.

---

## Phase 3 — Shortcut #2: Alert parser

This is the low-effort path: instead of typing amount + merchant, you copy a Hong Leong
Bank transaction alert and the shortcut parses both fields out of the text, then logs
with `Auto-logged: true`.

**Full build guide: [`SHORTCUT.md`](./SHORTCUT.md).** In short:

1. Copy the alert text from the HLB app's transaction detail (push banners can't be copied).
2. Run **Log Expense from Alert** → it runs `Match Text` regexes to pull `Amount` and
   `Merchant`, you tap a `Category`, and it POSTs the same body as Shortcut #1 (same
   hardcoded `Account` relation) with `"Auto-logged": true`.

> **iOS reality:** Shortcuts cannot auto-trigger on a third-party app's *push*
> notification, so this is started manually (Share Sheet / Back Tap / widget). A fully
> automatic "Is Notified" automation only works for apps that expose that trigger and
> isn't available for the HLB push alerts — hence the copy-and-run design.

---

## Adding funds (income)

Money coming *in* — your weekly allowance from family, your monthly internship
allowance, any deposit — is logged just like an expense, but with **`Type` = Income**:

| Property | Value |
| --- | --- |
| `Merchant` | Source, e.g. `Weekly allowance`, `Internship allowance` |
| `Amount` | The amount received, **positive** (`Type` flips the sign, not you) |
| `Type` | `Income` |
| `Date` | When it arrived |
| `Account` | 🏦 Bank balance (same relation as expenses) |

Because `Delta = if(Type == "Income", Amount, -Amount)`, an income row *adds* to
`Current balance`. The **By Category** board and the **Spending by Category** chart are
filtered to `Type != Income`, so allowances never distort your spending stats — and the
**💰 Income** view lists them on their own. Blank `Type` is treated as an expense, so
existing rows and the manual/alert shortcuts (which set `Type` = Expense) are unaffected.

> Recurring deposits like these are also good candidates for a **scheduled** Shortcut
> automation (time-of-day triggers run unattended on iOS) — see `SHORTCUT.md`.

## Transfers between accounts

Moving money between your own accounts — reloading an e-wallet, sweeping cash into Ryt savings — isn't spending or income, so it's logged as **two rows** that net to zero across your total wealth:

| Row | `Type` | `Account` | `Amount` | Effect |
| --- | --- | --- | --- | --- |
| Out of source | `Transfer Out` | the account money leaves | the amount, positive | `Delta` is **−** → source `Current balance` drops |
| Into destination | `Transfer In` | the account money enters | the same amount, positive | `Delta` is **+** → destination `Current balance` rises |

Example — moving RM200 from Bank to Ryt savings: one `Transfer Out` row on `🏦 Bank balance` and one `Transfer In` row on `🐷 Ryt Bank (Savings)`, both RM200. Bank drops 200, Ryt rises 200, total net worth unchanged.

Because `Delta = if(Type == "Income" or Type == "Transfer In", Amount, -Amount)`, the signs are automatic — always enter `Amount` as a positive number. Transfers are excluded from spending stats: `ThisMonthExpense` (so budgets) counts only `Expense` rows, and the **🗂️ By Category** board filters all transfer + income rows out. The **↔️ Transfers** view lists both legs together so you can eyeball that each pair matches.

> No Shortcut support for transfers — they're occasional, so add the two rows by hand with the **New** button (or duplicate an existing pair). If a spending **chart block** on the dashboard was set to `Type != Income`, widen its filter to also exclude `Transfer Out`/`Transfer In` so transfers don't show up as spend.

## Budgets & monthly tracking

The `Budgets` database (above) caps spending per category and resets every month on its own. Three views drive the workflow:

| View | Where | What it shows |
| --- | --- | --- |
| **🎯 Budgets** | Dashboard | Per-category `Monthly limit · Spent · Remaining`, sorted by `Spent`. Your at-a-glance "how am I doing this month". |
| **📆 Monthly** | Transactions | Date-sorted transactions. In the UI, **Group by → Month** for per-month subtotals (the API can't group by the `Month` formula, so set this grouping once by hand). |
| **🔗 To link** | Transactions | The manual-linking worklist: every row whose `Budget` relation is still empty. Empty = nothing to do. |

**Setup:** open **🎯 Budgets** and set a `Monthly limit` on each category row.

**Routine:** budgets only count transactions whose `Budget` is linked, and new rows (manual button or Shortcut) arrive unlinked. So make it a habit — open **🔗 To link**, set each row's `Budget` to its matching category, and the list empties. Until you do, `Spent` undercounts.

> This linking is deliberately manual. If clearing **🔗 To link** becomes a chore, the Shortcut can be extended to set the `Budget` relation at log time (it would need the budget row's page ID per category, mirroring how `Account` is hardcoded) — see `SHORTCUT.md`.

## Monthly reconciliation

Once a month, skim your bank statement against the Notion DB. Anything missing → log via Shortcut #1 with `(reconciled)` in `Notes`. This is the ~10-30% the alert parser won't catch (transfers, alerts you didn't copy, wording changes). Since the balance is driven entirely by logged expenses, reconciling also keeps `Current balance` honest — if it drifts from the real bank balance, a transaction was missed.
