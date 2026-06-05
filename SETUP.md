# Setup

Personal expense logging via iOS Shortcuts → Notion. Two shortcuts:
1. **Manual** — tap after a purchase, type amount/merchant (Home Screen / Lock Screen / Action Button).
2. **Alert parser** — copy a Hong Leong Bank transaction alert and run the shortcut; it extracts amount + merchant for you. Build guide: **`SHORTCUT.md`**.

Every expense links to a single **bank-balance account**, and a formula keeps a live running balance (starting balance − everything spent). Plus monthly reconciliation against the bank statement for whatever the alert parser misses.

> **Secrets:** never commit your Notion integration token. Store it only in the Shortcut on your phone. The only true secret is `<YOUR_TOKEN>`; the database and account IDs below are not secret (you still need the token to use them).

### Reference — this workspace's IDs

| Placeholder         | Value                                                              |
| ------------------- | ------------------------------------------------------------------ |
| `<YOUR_DB_ID>`      | `36ec4ed7327b8083a0dec2161698e9ff` (Expenses database)             |
| `<ACCOUNT_PAGE_ID>` | `afc516a99cdf4dc9b16d08a11ccc2ace` (🏦 Bank balance account row)    |
| `<YOUR_TOKEN>`      | **secret** — store on-device only                                  |

The databases live under a **💰 Expense Management** page, which also serves as a dashboard (auto-balance callout + both databases inline).

---

## Phase 1 — Notion

The setup uses **two databases**: `Expenses` (one row per purchase) and `Accounts` (a single row holding your bank balance). Expenses link to the account via a Relation, and a rollup + formula on the account turn that into a live balance.

### `Expenses` database

| Property      | Type             | Notes                                                                      |
| ------------- | ---------------- | -------------------------------------------------------------------------- |
| `Merchant`    | Title            | Row's primary field                                                        |
| `Date`        | Date             |                                                                            |
| `Amount`      | Number           | Number format set to **Ringgit** (RM)                                      |
| `Category`    | Select           | Food, Transport, Groceries, Bills, Entertainment, Shopping, Health, Other  |
| `Account`     | Relation         | → `Accounts`. Every expense points at the one bank-balance row             |
| `Notes`       | Text             | Corrections, context                                                       |
| `Type`        | Select           | `Expense` / `Income`. Blank counts as Expense. Drives `Delta`'s sign       |
| `Auto-logged` | Checkbox         | Distinguishes auto entries from manual                                     |
| `Delta`       | Formula          | `if(Type == "Income", Amount, -Amount)` — signed balance effect, summed by the account rollup |

Keep `Category` as a **Select** (keeps the Shortcut JSON simple). `Account` is a **Relation**, not a Select — that's what powers the auto-updating balance below.

### `Accounts` database

A single row (`🏦 Bank balance`) representing your bank account.

| Property           | Type      | Notes                                                            |
| ------------------ | --------- | ---------------------------------------------------------------- |
| `Account`          | Title     | The account name                                                 |
| `Starting balance` | Number    | Your bank balance at the moment you started logging              |
| `Expenses`         | Relation  | Auto-created reverse side of `Expenses.Account`                  |
| `Expenses total`   | Rollup    | Sum of `Delta` (or `Amount`) across all related expenses         |
| `Current balance`  | Formula   | `Starting balance + Expenses total` → the live running balance   |

**How auto-balance works:** the Shortcut hard-links every new expense to this one account row. The rollup re-totals spending and `Current balance` recalculates instantly — no manual updating. Because there's a single account, the Shortcut never has to ask which account to use.

> Tracking only the bank balance (not cash) is intentional — it keeps things to one account with zero per-expense choices.

### Create the integration

1. https://www.notion.so/my-integrations → **New integration** → Internal type, name `Expense Shortcut`.
2. Capabilities: Insert + Update + Read content.
3. Copy the Internal Integration Token (`secret_...` or `ntn_...`). Store in iOS Passwords / 1Password.

### Connect the databases to the integration

The integration needs access to **both** databases: it writes to `Expenses`, and the
`Account` relation points at a row in `Accounts` — if `Accounts` isn't shared, the API
returns `object_not_found` on the relation even though the row exists.

Easiest: open the **💰 Expense Management** page → top-right `⋯` → **Connections** →
add **Expense Shortcut**. Connecting at the parent page cascades to both child
databases. (Or connect `Expenses` and `Accounts` individually.)

### Grab the IDs you'll need in the Shortcut

- **`<YOUR_DB_ID>`** — open the `Expenses` DB as a full page in the browser. URL is `notion.so/<workspace>/<32-char-id>?v=...`. The 32-char hex string is the DB ID.
- **`<ACCOUNT_PAGE_ID>`** — open the single account row (`🏦 Bank balance`) as a full page. The 32-char hex string in its URL is the page ID. This is what the `Account` relation points to. (Concrete values are in the *Reference — this workspace's IDs* table near the top.)

---

## Phase 2 — Shortcut #1: Manual

In the Shortcuts app, name it **Log Expense**.

> **Magic Variables:** every action's output is a "Magic Variable" — a coloured pill you drop into later actions. To insert one, tap inside a field and pick from the chip row above the keyboard. Rename each action's output (tap the pill at the top of the action) so it reads `Amount`, `Merchant`, etc. instead of `Provided Input`.

### Action sequence

1. **Ask for Input** — Prompt `Amount?`, **Input Type: Number**. Rename output → `Amount`.
2. **Ask for Input** — Prompt `Merchant?`, Input Type: Text. Rename output → `Merchant`.
3. **Choose from List** — items: `Food`, `Transport`, `Groceries`, `Bills`, `Entertainment`, `Shopping`, `Health`, `Other`. Rename output → `Category`.
4. **Current Date**.
5. **Format Date** — Date Format: **Custom**, Format String: `yyyy-MM-dd`. Feed Current Date in. Rename output → `Today`. (Without this step the date arrives as `29/05/2026, 12:00`, which Notion rejects — it needs ISO 8601.)
6. **Text** — paste the JSON below, then replace each `[bracketed]` placeholder by inserting the matching Magic Variable. **Amount stays unquoted; everything else stays inside its quotes.** `Account` has no chooser — it's hardcoded to your one account, so paste your `<ACCOUNT_PAGE_ID>` once and forget it.

   ```json
   {
     "parent": { "database_id": "<YOUR_DB_ID>" },
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

## Monthly reconciliation

Once a month, skim your bank statement against the Notion DB. Anything missing → log via Shortcut #1 with `(reconciled)` in `Notes`. This is the ~10-30% the alert parser won't catch (transfers, alerts you didn't copy, wording changes). Since the balance is driven entirely by logged expenses, reconciling also keeps `Current balance` honest — if it drifts from the real bank balance, a transaction was missed.
