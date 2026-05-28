# Setup

Personal expense logging via iOS Shortcuts → Notion. Two shortcuts:
1. **Manual** — tap after a purchase (Home Screen / Lock Screen / Action Button).
2. **Auto** — Personal Automation triggered by a bank app's transaction notification.

Plus monthly reconciliation against the bank statement for whatever the auto path misses.

> **Secrets:** never commit your Notion integration token or database ID. Store them only in the Shortcut on your phone. Placeholders below: `<YOUR_TOKEN>`, `<YOUR_DB_ID>`.

---

## Phase 1 — Notion

### Create the database

| Property      | Type     | Notes                                                                |
| ------------- | -------- | -------------------------------------------------------------------- |
| `Merchant`    | Title    | Row's primary field                                                  |
| `Date`        | Date     |                                                                      |
| `Amount`      | Number   | Plain number; MYR implicit                                           |
| `Category`    | Select   | Food, Transport, Groceries, Bills, Entertainment, Shopping, Health, Other |
| `Account`     | Select   | Apple Pay, Maybank, CIMB, Cash, Other                                |
| `Notes`       | Text     | Corrections, context                                                 |
| `Auto-logged` | Checkbox | Distinguishes auto entries from manual                               |

Keep `Category` and `Account` as **Select**, not Relation — keeps the Shortcut JSON simple.

### Create the integration

1. https://www.notion.so/my-integrations → **New integration** → Internal type, name `Expense Shortcut`.
2. Capabilities: Insert + Update + Read content.
3. Copy the Internal Integration Token (`secret_...` or `ntn_...`). Store in iOS Passwords / 1Password.

### Connect the database to the integration

Open the DB → top-right `⋯` → **Connections** → Connect to → "Expense Shortcut".

### Grab the database ID

Open the DB as a full page in the browser. URL is `notion.so/<workspace>/<32-char-id>?v=...`. The 32-char hex string is the DB ID.

---

## Phase 2 — Shortcut #1: Manual

In the Shortcuts app, name it **Log Expense**. Action sequence:

1. **Ask for Input** — "Amount?" — Number — save as `Amount`.
2. **Ask for Input** — "Merchant?" — Text — save as `Merchant`.
3. **Choose from List** — Category options (matching the Notion Select) — save as `Category`.
4. **Choose from List** — Account options — save as `Account`.
5. **Current Date** → **Format Date** → ISO 8601, format `yyyy-MM-dd` — save as `Today`.
6. **Dictionary**:

```
parent:
  database_id: <YOUR_DB_ID>
properties:
  Merchant:
    title:
      - text:
          content: [Merchant]
  Amount:
    number: [Amount]
  Date:
    date:
      start: [Today]
  Category:
    select:
      name: [Category]
  Account:
    select:
      name: [Account]
  Auto-logged:
    checkbox: false
```

7. **Get Contents of URL**:
   - URL: `https://api.notion.com/v1/pages`
   - Method: `POST`
   - Headers:
     - `Authorization: Bearer <YOUR_TOKEN>`
     - `Notion-Version: 2022-06-28`
     - `Content-Type: application/json`
   - Request Body: **JSON** → reference the Dictionary.
8. **Show Notification** — "Logged RM[Amount] at [Merchant]".

Add to Home Screen, Lock Screen widget, and (iPhone 15 Pro / 16+) the Action Button.

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
5. Same Notion POST as Shortcut #1, but with `Auto-logged: true` and `Account` hardcoded to that bank.

**Tuning the regex:** trigger one small real transaction first, screenshot the actual notification, and write the regex against that exact wording. Don't trust generic patterns.

---

## Monthly reconciliation

Once a month, skim your bank statement against the Notion DB. Anything missing → log via Shortcut #1 with `(reconciled)` in `Notes`. This is the ~10-30% the auto path won't catch (cash, transfers, missed notifications, wording changes).
