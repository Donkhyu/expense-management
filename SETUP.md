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
| `Account`     | Select   | Debit, Cash                                                          |
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

In the Shortcuts app, name it **Log Expense**.

> **Magic Variables:** every action's output is a "Magic Variable" — a coloured pill you drop into later actions. To insert one, tap inside a field and pick from the chip row above the keyboard. Rename each action's output (tap the pill at the top of the action) so it reads `Amount`, `Merchant`, etc. instead of `Provided Input`.

### Action sequence

1. **Ask for Input** — Prompt `Amount?`, **Input Type: Number**. Rename output → `Amount`.
2. **Ask for Input** — Prompt `Merchant?`, Input Type: Text. Rename output → `Merchant`.
3. **Choose from List** — items: `Food`, `Transport`, `Groceries`, `Bills`, `Entertainment`, `Shopping`, `Health`, `Other`. Rename output → `Category`.
4. **Choose from List** — items: `Debit`, `Cash`. Rename output → `Account`.
5. **Current Date**.
6. **Format Date** — Date Format: **Custom**, Format String: `yyyy-MM-dd`. Feed Current Date in. Rename output → `Today`. (Without this step the date arrives as `29/05/2026, 12:00`, which Notion rejects — it needs ISO 8601.)
7. **Text** — paste the JSON below, then replace each `[bracketed]` placeholder by inserting the matching Magic Variable. **Amount stays unquoted; everything else stays inside its quotes.**

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
       "Account": { "select": { "name": "[Account]" } },
       "Auto-logged": { "checkbox": false }
     }
   }
   ```

   > ⚠️ **Smart-quote trap.** iOS auto-replaces `"` with curly `" "`, which is not valid JSON. Before pasting: **Settings → General → Keyboard → turn off Smart Punctuation**. If you've already pasted, retype every `"` after disabling it.

8. **Get Contents of URL** — URL `https://api.notion.com/v1/pages`. Expand Show More:
   - **Method:** `POST`
   - **Headers:**
     - `Authorization` → `Bearer <YOUR_TOKEN>`
     - `Notion-Version` → `2022-06-28`
     - `Content-Type` → `application/json`
   - **Request Body:** **File** → insert the Text action's Magic Variable. (If File misbehaves, switch to **JSON** with the same Text variable as input.)
9. **Show Notification** — `Logged RM[Amount] at [Merchant]`.

### Test and debug

Run the shortcut with ▶︎. On success, a new row appears in Notion within ~1s.

If it fails, drop a **Quick Look** action between the Text action and `Get Contents of URL`, run again, and inspect the actual body. Common Notion responses and what they mean:

| Response | Cause |
| --- | --- |
| `invalid_json` with `"number": ,` | `Amount` Magic Variable didn't insert. Re-add it; confirm step 1 has Input Type Number. |
| `invalid_json` with curly `"…"` | Smart Punctuation is on. Disable and retype quotes. |
| `validation_error` mentioning `start` | Date is `29/05/2026, …` not `2026-05-29`. The `Format Date` step is missing or you inserted Current Date instead of `Today`. |
| `unauthorized` | Token wrong, or DB isn't connected to the integration (DB → ⋯ → Connections). |
| `object_not_found` | `database_id` typo, or DB not shared with the integration. |
| `validation_error` on a `select` property | The value doesn't exactly match an option in Notion (case-sensitive). |

### Place the shortcut

- **Home Screen:** long-press in the Shortcuts app → Share → Add to Home Screen.
- **Lock Screen widget:** Customize Lock Screen → Add Widget → Shortcuts → pick `Log Expense`.
- **Action Button** (iPhone 15 Pro / 16+): Settings → Action Button → Shortcut → `Log Expense`. Biggest QoL win — one press after paying.
- **Siri:** "Hey Siri, log expense" works automatically by name.

### Alternative: build the JSON with the Dictionary action

The Text-action approach above is recommended because the body is readable in one screen. If you prefer the typed Dictionary action, build this structure — at the top level use two keys of type `Dictionary` (`parent`, `properties`); inside `properties`, six `Dictionary` keys whose contents mirror the JSON above. `Merchant.title` requires an `Array` whose single item is a Dictionary. It works, it's just slow to enter and hard to debug.

---

## Phase 3 — Shortcut #2: Auto (HLB notification-triggered)

Reference notification (Hong Leong Bank `Transaction Notice`):

```
Your card ending 8233 has been debited for MYR6.60 at ECO MART - RESI 121 using Apple Pay. Please call us if you did not make this transaction.
```

Whether the transaction was Apple Pay or a direct card swipe, the money comes from the same HLB debit account, so `Account` is hardcoded to `Debit` — no branching needed.

### Build the shortcut

In Shortcuts: long-press `Log Expense` → **Duplicate** → rename to `Log Expense (Auto - HLB)`.

In the duplicate, delete:
- Ask for Input "Amount?"
- Ask for Input "Merchant?"
- Choose from List → Account

Then add, above the Text (JSON) action:

1. **Receive input** at the start — Shortcut Input type **Text**. Rename → `NotificationText`.
2. **Match Text** on `NotificationText` with regex:
   ```
   MYR\s?([0-9,]+\.[0-9]{2})
   ```
   Set **Group Index = 1**. Rename → `Amount`.
3. **If** `Amount` does not have any value → **Stop Shortcut**. End If. (Filters out non-transaction notifications.)
4. **Match Text** on `NotificationText` with regex:
   ```
   at (.+?)(?: using Apple Pay|\. Please)
   ```
   Group Index = 1. Rename → `Merchant`. The alternation handles both Apple Pay and direct-card wording; if HLB changes the wording, this is the line to retune.
5. **Text** action: `Debit`. Rename → `Account`.

In the existing Text (JSON) action, flip:

```json
"Auto-logged": { "checkbox": false }
```

to

```json
"Auto-logged": { "checkbox": true }
```

Everything else (Choose from List → Category, Current Date, Format Date, Get Contents of URL, Show Notification) stays as-is — the existing `[Amount]`, `[Merchant]`, `[Today]`, `[Category]`, `[Account]` pills automatically pick up the new sources.

### Create the Personal Automation

Shortcuts → **Automation** tab → **+** → **New Automation** → scroll to **App** → tap → search **HLB Connect** (or whichever HLB app actually posts the `Transaction Notice` — verify by long-pressing a real notification) → Next → choose **Is Notified** (iOS 26 may also show this as "Sends Notification" or "Notification Received").

- **Action:** Run Shortcut → `Log Expense (Auto - HLB)`.
- **Run Immediately:** ON.
- **Notify When Run:** ON initially (so you can see when it fires); turn off once you trust it.

If the HLB app doesn't offer any notification-related trigger, that bank app delivers via a system service Shortcuts can't intercept — fall back to a Share Sheet shortcut (long-press the notification → Share → Run Shortcut).

### Test

1. Trigger one small Apple Pay transaction (RM1–5).
2. Expect: HLB notification → Shortcuts run notification → Category prompt → row in Notion with `Account: Debit`, `Auto-logged: true`.
3. Do one direct card transaction (online purchase, or contactless physical card without Apple Pay). If the merchant comes back empty, the wording after the merchant isn't `. Please call us…` — capture that notification, update the alternation in step 4.

### Caveats

- **SMS-only transactions are missed.** Some HLB transactions (ATM withdrawals, certain transfers) only send SMS, and iOS doesn't expose SMS to Shortcuts. Monthly reconciliation catches these.
- **iOS occasionally pauses Personal Automations** that don't run for a while. Check the Automation tab if entries stop appearing.
- **Notification copy is brittle.** If HLB refreshes their app's wording, the regexes break. Easy to re-tune when it happens.

---

## Monthly reconciliation

Once a month, skim your bank statement against the Notion DB. Anything missing → log via Shortcut #1 with `(reconciled)` in `Notes`. This is the ~10-30% the auto path won't catch (cash, transfers, missed notifications, wording changes).
