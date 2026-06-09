# 💰 Expense Management

A personal finance tracker built in **Notion**, logged from an **iOS Shortcut**. Modelled on
the *Finance Tracker* layout: money movements are split across **Expenses / Incomes /
Transfers**, which feed **Accounts** (live balances), **Categories** (auto-resetting monthly
budgets) and **Months** (monthly totals) — plus **Savings Goals**, **Recurring** and **Net
Worth** modules on top.

No spreadsheets, no manual math: log a purchase in one tap and every balance, budget and
total updates itself.

## How it works

```
🧾 Expenses ─┐                         ┌─ 🗂️ Categories  (budget usage, auto-resets monthly)
💵 Incomes  ─┼──→ 🏦 Accounts (Balance) ┤
↔️ Transfers ┘                         └─ 📅 Months      (per-month expense/income totals)
```

- **Log** an expense from your phone — the Shortcut writes a row to **Expenses** with the
  amount, account and category.
- **Balances** are rollups: each account totals its own expenses, income and transfers, so
  `Balance` is always live. Total net worth = sum of all account balances.
- **Budgets** count only the current month's expenses, so they reset on the 1st with no
  rollover.
- Everything is surfaced on one **💰 Finance** dashboard; the raw tables live under a nested
  **🗄️ Databases** page.

## Repo contents

| File | What's in it |
| --- | --- |
| [`SETUP.md`](./SETUP.md) | The Notion side — every database's schema and formulas, the workspace structure, the integration setup, and day-to-day workflows. Includes this workspace's database/page IDs. |
| [`SHORTCUT.md`](./SHORTCUT.md) | The iOS Shortcut build — parses a Hong Leong Bank alert (or manual input) and POSTs an expense to Notion, with the JSON payload and regexes. |

## Security

The only secret is the **Notion integration token** — it lives on the phone (in the
Shortcut), never in this repo. The database and page IDs in `SETUP.md` aren't secret on their
own (the token is still required to use them).
