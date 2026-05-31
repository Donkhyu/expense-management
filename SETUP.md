# Setup

Personal expense logging via iOS Shortcuts → Notion. Two shortcuts:
1. **Manual** — tap after a purchase (Home Screen / Lock Screen / Action Button).
2. **Auto** — Personal Automation triggered by a bank app's transaction notification.

Every expense links to a single **bank-balance account**, and a formula keeps a live running balance (starting balance − everything spent). Plus monthly reconciliation against the bank statement for whatever the auto path misses.

> **Secrets:** never commit your Notion integration token. Store it only in the Shortcut on your phone. Placeholders below: `<YOUR_TOKEN>`, `<YOUR_DB_ID>`, `<ACCOUNT_PAGE_ID>`.

---

## Phase 1 — Notion

The setup uses **two databases**: `Expenses` (one row per purchase) and `Accounts` (a single row holding your bank balance). Expenses link to the account via a Relation, and a rollup + formula on the account turn that into a live balance.

### `Expenses` database

| Property      | Type             | Notes                                                                      |
| ------------- | ---------------- | -------------------------------------------------------------------------- |
| `Merchant`    | Title            | Row's primary field                                                        |
| `Date`        | Date             |                                                                            |
| `Amount`      | Number           | Plain number; MYR implicit                                                 |
| `Category`    | Select           | Food, Transport, Groceries, Bills, Entertainment, Shopping, Health, Other  |
| `Account`     | Relation         | → `Accounts`. Every expense points at the one bank-balance row             |
| `Notes`       | Text             | Corrections, context                                                       |
| `Auto-logged` | Checkbox         | Distinguishes auto entries from manual                                     |
| `Delta`       | Formula          | `-Amount` — the signed effect on the balance, summed by the account rollup |

Keep `Category` as a **Select** (keeps the Shortcut JSON simple). `Account` is a **Relation**, not a Select — that's what powers the auto-updating balance below.

### `Accounts` database

A single row (e.g. `💰 Main balance`) representing your bank account.

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

### Connect the database to the integration

Open the **`Expenses`** DB → top-right `⋯` → **Connections** → Connect to → "Expense Shortcut".

### Grab the IDs you'll need in the Shortcut

- **`<YOUR_DB_ID>`** — open the `Expenses` DB as a full page in the browser. URL is `notion.so/<workspace>/<32-char-id>?v=...`. The 32-char hex string is the DB ID.
- **`<ACCOUNT_PAGE_ID>`** — open the single account row (`Main balance`) as a full page. The 32-char hex string in its URL is the page ID. This is what the `Account` relation points to.

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

## Phase 3 — Shortcut #2: Auto (notification-triggered)

In Shortcuts → **Automation** tab → **+** → **New Automation**:

1. Trigger: **App** → bank app (MAE for Maybank, CIMB OCTO for CIMB) → **Is Notified**.
   - If "Is Notified" isn't available on your iOS, fall back to a Share Sheet shortcut: long-press the bank notification → Share → run shortcut.
2. **Run Immediately** on; **Notify When Run** off.
3. The trigger provides a `Notification` variable with the body text. Add:
   - **Match Text** — regex for amount, e.g. `RM\s?([0-9,]+\.[0-9]{2})` — capture Group 1 → `Amount`.
   - **Match Text** — regex for merchant — bank-specific; tune against real notifications.
   - **If** `Amount` is empty → exit (notification wasn't a transaction).
4. **Choose from List** → Category (set this manually; auto-categorization is brittle).
5. Same Notion POST as Shortcut #1, with the same hardcoded `Account` relation, but with `"Auto-logged": true`.

**Tuning the regex:** trigger one small real transaction first, screenshot the actual notification, and write the regex against that exact wording. Don't trust generic patterns.

---

## Monthly reconciliation

Once a month, skim your bank statement against the Notion DB. Anything missing → log via Shortcut #1 with `(reconciled)` in `Notes`. This is the ~10-30% the auto path won't catch (transfers, missed notifications, wording changes). Since the balance is driven entirely by logged expenses, reconciling also keeps `Current balance` honest — if it drifts from the real bank balance, a transaction was missed.
