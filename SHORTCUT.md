# iOS Shortcut — Log Expense from a Bank Alert

A single Shortcut that reads a **Hong Leong Bank** transaction alert, extracts the
**amount** and **merchant**, and posts a row to the Notion `Expenses` database. The
account balance updates automatically (see `SETUP.md` for the Notion side).

> The Notion **token** is the only real secret — keep it on-device. The non-secret
> database and account IDs for this workspace are listed in `SETUP.md`.

---

## How you use it

1. You get an HLB transaction push (card / Apple Pay, or QR payment).
2. Copy the alert text — open the transaction in the HLB app where the text is
   selectable, and copy it. (iOS can't copy directly from a push banner.)
3. Run **Log Expense from Alert** (Share Sheet, Home Screen, Lock Screen widget, or
   Back Tap).
4. It auto-fills **Amount** + **Merchant**; you tap a **Category**; it logs the row
   and the account balance drops instantly.

> **Why manual?** iOS Shortcuts cannot auto-trigger on a *third-party* app's push
> notification (like the HLB app). There is no "when notification arrives" trigger
> for push, so the run is started by you — but all the typing is automated away.

---

## Alert templates it understands

| Type             | Example alert text                                                                 | Merchant          | Amount  |
| ---------------- | ---------------------------------------------------------------------------------- | ----------------- | ------- |
| Card / Apple Pay | `Your card ending 8233 has been debited for MYR23.80 at BOOST JUICE TRX using Apple Pay.` | `BOOST JUICE TRX` | `23.80` |
| QR payment       | `Your QR payment of RM3.50 to WOF SUBANG ELITE is SUCCESSFUL.`                      | `WOF SUBANG ELITE`| `3.50`  |
| Transfer         | `Your transfer of RM20.00 to DEON HIU JIU WE at TOUCH 'n GO is SUCCESSFUL`          | `TOUCH 'n GO` (destination; recipient → Notes) | `20.00` |

If HLB sends still other wordings (refunds, reversals), add another branch in step 4 —
match on a keyword unique to that template, then extract with its own regex.

> **Note on transfers.** This parser logs **expenses**. A bank *transfer* (e.g. a Touch 'n Go
> top-up) belongs in the **Transfers** database as a single row with a `From Account` /
> `To Account` — log those by hand. If you'd rather capture a top-up quickly here, pick
> `Other`; just know it lands in Expenses, not Transfers.

---

## Build the Shortcut

In the Shortcuts app, create a new shortcut named **Log Expense from Alert**.

> **Magic Variables:** each action's output is a coloured pill you drop into later
> actions. Rename the relevant outputs (`Amount`, `Merchant`, `Category`, `Today`) so
> they're easy to find.

1. **Get Clipboard.** Save as `AlertText`.
   - *Prefer a prompt instead?* Use **Ask for Text**, Default Answer = `Clipboard`.
2. **Match Text** on `AlertText` — Amount:
   ```
   (?:MYR|RM)\s*([0-9,]+\.[0-9]{2})
   ```
   → **Get Group at Index** `1` → **Replace Text** (find `,`, replace empty) → rename → `Amount`.
3. *(Optional, unused.)* The `Expenses` DB has no Notes field, so you can skip a `Notes`
   action. The transfer branch below still extracts the recipient, but there's nowhere to
   store it — log transfers in the Transfers DB instead (see the note above).
4. **Branch by template** to fill `Merchant` (and `Notes` for transfers). Build this as
   nested **If / Otherwise** actions on `AlertText`, in this order — a transfer also
   contains "to … is", so it must be tested *before* the QR pattern:

   | Test: `AlertText` **contains** | Merchant — `Match Text` → Group 1 | Also set |
   | ------------------------------ | --------------------------------- | -------- |
   | `transfer of`                  | `at (.+?) is` → destination       | `Notes` ← `transfer of .+? to (.+?) at` (recipient) |
   | `QR payment`                   | `to (.+?) is`                     | —        |
   | *(otherwise — card)*           | `at (.+?) using`                  | —        |

   So: `If contains "transfer of"` → … `Otherwise` → `If contains "QR payment"` → … `Otherwise` → card. Each branch ends with **Get Group at Index** `1` → set `Merchant`.
5. **Choose from Menu** — `Food`, `Transport`, `Groceries`, `Bills`, `Entertainment`,
   `Shopping`, `Health`, `Other` → rename → `Category`. (See *Auto-category* below to
   skip this for known merchants.)
   Then map the category two ways — to its **Notion page ID** (the `Category` relation needs
   the row's ID, not its name) and to an emoji (page icon):
   - **Dictionary** — category → **page ID** (from the *Category row IDs* table in `SETUP.md`):
     ```
     Food : 37ac4ed7327b81739afafe3f06554a01
     Transport : 37ac4ed7327b81d898dcce506b54a166
     Groceries : 37ac4ed7327b81db8d71f20999a7edca
     Bills : 37ac4ed7327b8110b9f3deaff7017823
     Entertainment : 37ac4ed7327b81b99919ffef9d357820
     Shopping : 37ac4ed7327b8186a8f3d24635b376b6
     Health : 37ac4ed7327b81b5885ecb120ef45b7f
     Other : 37ac4ed7327b81b9bccfeb50d139c1dd
     ```
     **Get Dictionary Value** → *Value* for *Key* = `Category` → rename → `CategoryId`.
   - **Dictionary** — category → emoji (`Food : 🍔`, `Transport : 🚕`, `Groceries : 🛒`, `Bills : 💡`, `Entertainment : 🎬`, `Shopping : 🛍️`, `Health : 💊`, `Other : 📦`) → **Get Dictionary Value** for `Category` → rename → `Icon`.
6. **Current Date** → **Format Date** (Custom, format string `yyyy-MM-dd`) → rename → `Today`.
   - Required: Notion needs ISO 8601 (`2026-05-31`), not `31/05/2026, 12:00`.
7. **Text** — paste the JSON below, then replace each `[bracket]` by inserting the
   matching Magic Variable. **`Amount` stays unquoted; everything else stays quoted.**
   ```json
   {
     "parent": { "database_id": "<EXPENSES_DB_ID>" },
     "icon": { "emoji": "[Icon]" },
     "properties": {
       "Expense":  { "title":    [ { "text": { "content": "[Merchant]" } } ] },
       "Amount":   { "number":   [Amount] },
       "Date":     { "date":     { "start": "[Today]" } },
       "Account":  { "relation": [ { "id": "<BANK_ACCOUNT_ID>" } ] },
       "Category": { "relation": [ { "id": "[CategoryId]" } ] }
     }
   }
   ```
   > ⚠️ **Smart-quote trap.** Disable **Settings → General → Keyboard → Smart
   > Punctuation** first, or iOS turns `"` into curly `“ ”` and the JSON breaks.
   > **Relations take page IDs, not names.** `Account` is hardcoded to Bank
   > (`<BANK_ACCOUNT_ID>`); `Category` uses the `[CategoryId]` from the page-ID Dictionary
   > above (change `Account` in Notion for a non-default account). The `Month` relation —
   > and, for a savings deposit, a `Goal` on a Transfer — are set later in Notion. The
   > Expenses DB has no Notes field, so transfer recipients aren't stored here.
8. **Get Contents of URL** — `https://api.notion.com/v1/pages`:
   - **Method:** `POST`
   - **Headers:** `Authorization` → `Bearer <YOUR_TOKEN>`, `Notion-Version` → `2022-06-28`,
     `Content-Type` → `application/json`
   - **Request Body:** **JSON** (or **File**) → insert the Text action's variable.
9. **Show Notification** — `Logged RM[Amount] at [Merchant]`.

---

## Regex reference

| Field           | Pattern                          | Then                          |
| --------------- | -------------------------------- | ----------------------------- |
| Amount              | `(?:MYR\|RM)\s*([0-9,]+\.[0-9]{2})` | Group 1 → strip `,` → Number  |
| Merchant (card)     | `at (.+?) using`                 | Group 1                       |
| Merchant (QR)       | `to (.+?) is`                    | Group 1                       |
| Merchant (transfer) | `at (.+?) is`                    | Group 1 (destination)         |
| Notes (transfer)    | `transfer of .+? to (.+?) at`    | Group 1 (recipient)           |

iOS **Match Text** uses ICU regex and supports capture groups via **Get Group at Index**.

---

## Optional: auto-category

To skip the menu for known merchants, add a **Text** + **If `AlertText` contains …**
chain (or a **Dictionary** keyword map) before step 5, e.g.:

| Keyword in alert | Category |
| ---------------- | -------- |
| `BOOST JUICE`    | Food     |
| `GRAB`           | Transport|
| `SHELL`, `PETRON`| Transport|
| `WATSON`, `GUARDIAN` | Health |

Fall back to **Choose from Menu** only when no keyword matches. Keep the map short —
brittle auto-categorisation is worse than one tap.

---

## Placement & triggers

- **Share Sheet:** shortcut settings → enable *Show in Share Sheet* (accept Text).
- **Home Screen:** long-press in Shortcuts → Share → Add to Home Screen.
- **Lock Screen widget:** Customize Lock Screen → Add Widget → Shortcuts.
- **Back Tap:** Settings → Accessibility → Touch → Back Tap → Double Tap → this shortcut.
- **Siri:** "Hey Siri, log expense from alert" works by name.

---

## Troubleshooting

Common Notion API errors: `unauthorized` (token wrong or DB not connected to the
integration), `object_not_found` (database/parent page not shared), `validation_error`
(bad relation page ID, or a date that isn't ISO `yyyy-MM-dd`), plus the smart-quote trap
above. Parser-specific issues:

| Symptom | Cause / fix |
| --- | --- |
| `Amount` empty | Alert text wasn't on the clipboard, or HLB changed the wording. Re-copy; check the regex against the new text. |
| `Merchant` empty | Neither template matched (new alert type). Add a `Match Text` branch for it. |
| Row logs but balance unchanged | `Account` relation missing/wrong, or the **Accounts** DB isn't shared with the integration (see `SETUP.md` → Connections). |
| `object_not_found` on the account | The Accounts DB / parent page isn't connected to `Expense Shortcut`. |
