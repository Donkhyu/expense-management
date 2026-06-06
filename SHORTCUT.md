# iOS Shortcut — Log Expense from a Bank Alert

A single Shortcut that reads a **Hong Leong Bank** transaction alert, extracts the
**amount** and **merchant**, and posts a row to the Notion `Transactions` database. The
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
   with **Auto-logged ✓** and the balance drops instantly.

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

> **Note on transfers.** A transfer to your own e-wallet (like this Touch 'n Go top-up)
> isn't really *spending* — but it does leave your bank account, so logging it keeps the
> balance accurate. Pick category `Other`, or add a `Transfer` option to the `Category`
> select if you'd like to exclude these from spend analysis.

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
3. **Text** action containing nothing → rename → `Notes`. (Only the transfer branch
   overwrites it; the others leave it empty.)
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
   Then map the category to an emoji for the page icon:
   - **Dictionary** action:
     ```
     Food : 🍔
     Transport : 🚕
     Groceries : 🛒
     Bills : 💡
     Entertainment : 🎬
     Shopping : 🛍️
     Health : 💊
     Other : 📦
     ```
   - **Get Dictionary Value** → *Value* for *Key* = `Category` → rename → `Icon`.
   - (For an income shortcut, skip the lookup and just use `💰`.)
6. **Current Date** → **Format Date** (Custom, format string `yyyy-MM-dd`) → rename → `Today`.
   - Required: Notion needs ISO 8601 (`2026-05-31`), not `31/05/2026, 12:00`.
7. **Text** — paste the JSON below, then replace each `[bracket]` by inserting the
   matching Magic Variable. **`Amount` stays unquoted; everything else stays quoted.**
   ```json
   {
     "parent": { "database_id": "<YOUR_DB_ID>" },
     "icon": { "emoji": "[Icon]" },
     "properties": {
       "Merchant":    { "title":    [ { "text": { "content": "[Merchant]" } } ] },
       "Amount":      { "number":   [Amount] },
       "Date":        { "date":     { "start": "[Today]" } },
       "Category":    { "select":   { "name": "[Category]" } },
       "Account":     { "relation": [ { "id": "<ACCOUNT_PAGE_ID>" } ] },
       "Notes":       { "rich_text":[ { "text": { "content": "[Notes]" } } ] },
       "Type":        { "select":   { "name": "Expense" } },
       "Auto-logged": { "checkbox": true }
     }
   }
   ```
   > ⚠️ **Smart-quote trap.** Disable **Settings → General → Keyboard → Smart
   > Punctuation** first, or iOS turns `"` into curly `“ ”` and the JSON breaks.
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

The Notion-response table in `SETUP.md` covers the API errors (`unauthorized`,
`object_not_found`, `validation_error`, smart-quotes). Parser-specific issues:

| Symptom | Cause / fix |
| --- | --- |
| `Amount` empty | Alert text wasn't on the clipboard, or HLB changed the wording. Re-copy; check the regex against the new text. |
| `Merchant` empty | Neither template matched (new alert type). Add a `Match Text` branch for it. |
| Row logs but balance unchanged | `Account` relation missing/wrong, or the **Accounts** DB isn't shared with the integration (see `SETUP.md` → Connections). |
| `object_not_found` on the account | The Accounts DB / parent page isn't connected to `Expense Shortcut`. |
