---
name: spencio
description: Read and change the user's Spencio expense tracker data through the Spencio MCP tools (transactions, accounts and credit cards, transfers, loans, goals, budgets, subscriptions and bills, categories, tags, merchant rules, SMS tracking). Use when the user asks about their spending, income, budgets, subscriptions or Spencio transactions, or wants to add, edit, recategorize or delete them.
when_to_use: >-
  Questions like how much did I spend on food last month, compare this month
  with last month, which subscriptions do I pay for, am I over budget; or
  changes like move all my Uber rides to Transport, add Rs 4,380 at Keells to
  groceries.
allowed-tools: mcp__plugin_spencio_spencio__list_transactions mcp__plugin_spencio_spencio__get_transaction mcp__plugin_spencio_spencio__list_accounts mcp__plugin_spencio_spencio__list_categories mcp__plugin_spencio_spencio__get_budgets mcp__plugin_spencio_spencio__spending_summary mcp__plugin_spencio_spencio__compare_periods mcp__plugin_spencio_spencio__find_recurring mcp__plugin_spencio_spencio__get_account_details mcp__plugin_spencio_spencio__list_loans mcp__plugin_spencio_spencio__list_goals mcp__plugin_spencio_spencio__list_bills mcp__plugin_spencio_spencio__get_preferences mcp__plugin_spencio_spencio__list_tags mcp__plugin_spencio_spencio__list_merchant_rules mcp__plugin_spencio_spencio__list_sms_review mcp__plugin_spencio_spencio__list_sms_senders
---

# Spencio

Spencio is a personal expense tracker. Its data lives only on the user's phone. The `spencio` MCP server forwards each tool call to the phone, which answers from its own database, so the Spencio app must be open on the phone with **MCP mode** on.

## Before anything else

- If a tool says **"Spencio isn't reachable"**, stop and ask the user to open the Spencio app on their phone and keep MCP mode on (Settings › AI assistants). Then retry once. Don't loop.
- If the server isn't connected, or a call comes back unauthorized, the user has to connect it: run `/mcp`, choose `plugin:spencio:spencio`, sign in in the browser, and type the 6-digit code the phone shows. MCP mode must be started on the phone first; the code expires after 5 minutes and works once.
- If a tool says the user hasn't agreed to share data yet, ask them to accept the consent sheet on the phone, then retry.
- If it says Spencio is locked, ask them to unlock the app.

## Data you get back

- **Money is integer minor units plus a currency code.** `{"amount_minor": 125000, "currency": "LKR"}` is Rs 1,250.00. Divide by 10^decimals when you show it (2 for most currencies, 0 for JPY). Never add amounts in different currencies together; report them per currency.
- Dates are `YYYY-MM-DD` in the phone's local time.
- Account and category ids are integers; transaction ids are strings. Get them from `list_accounts`, `list_categories` and `list_transactions`, never guess them.
- Notes are left out of every answer unless the user turned on Share notes on the phone. If a note is missing, say it isn't shared; don't assume there is none.
- Searching by `text` matches the transaction title (merchant or description).

## Reading

Start with the cheapest call that answers the question:

| Question | Tool |
|---|---|
| Totals: "how much on X", "biggest categories", "spend by month/merchant/account" | `spending_summary` with `group_by` (`category`, `merchant`, `month`, `account`) and `kind` (`expense` default, or `income`) |
| "This month vs last month" | `compare_periods` |
| "Am I on budget?" | `get_budgets` |
| Subscriptions, bills, repeating charges | `find_recurring` (tracked ones and ones detected in the last 14 months) |
| Specific transactions | `list_transactions` with filters (`from`, `to`, `category_ids`, `account_ids`, `text`, `min_amount_minor`, `max_amount_minor`, `kind`) |
| Account balances | `list_accounts` |
| Category names and ids | `list_categories` (a parent category includes its subcategories when you filter by it) |

- Prefer `spending_summary` over listing rows and adding them yourself: the phone computes the totals and leaves out transfers, loans and balance corrections.
- `list_transactions` returns up to `limit` rows (max 200, default 50) and a `next_cursor`. Only fetch more pages if the answer needs them.
- Resolve relative dates ("last month", "this year") to exact `from`/`to` dates using today's date, and say which range you used.

## Changing data

Every change can be undone by the user on the phone, and they see each one live. Still, be deliberate:

- **Confirm before writing** unless the user's request already says exactly what to do. Restate the change in one line (what, how much, which account and category) for anything ambiguous.
- `create_transaction`: `kind` (`expense` or `income`), `amount_minor` in the account's currency, `account_id`, optional `category_id` (its kind must match), `date` (defaults to today), `title`, `note`. Look up the account and category ids first.
- `update_transaction`: only the fields you pass change. `note: ""` removes the note. Moving to another account only works within the same currency. Transfers and loan transactions can't be changed here; tell the user to do it in the app.
- `delete_transaction`: expenses and incomes only. Ask first unless the user clearly asked for that exact transaction to go.
- An account that can't go below zero (cash, bank, wallet) refuses an expense it can't cover; report the message as is.

### Bulk recategorize: always two steps

1. `recategorize_transactions` with `mode: "preview"`, `category_id`, and either `transaction_ids` or a `filter` (same fields as `list_transactions`, at most 500). It changes nothing. Show the user the count, the categories they come from, the total and a few examples, and mention anything `skipped`.
2. `recategorize_transactions` with `mode: "apply"` and the `change_id` from the preview.

When the preview says `needs_approval: true` (5 or more transactions), apply waits for the user to approve **on their phone**. Tell them to check the Spencio app before you call apply. They can allow all, pick some, or deny:
- `status: "applied"`: all done.
- `status: "partly_applied"`: they picked some; report `applied` and `not_applied`.
- `status: "denied"`: nothing changed. Don't retry unless they ask.
- If apply says it's **still waiting for the user's answer**, the phone still shows the request. Ask the user to answer it, then call apply again with the **same** `change_id` (it returns the same result; it never applies twice).

A `change_id` expires after 30 minutes; preview again if it's unknown.

## Everything else Spencio tracks

| Area | Read | Change |
|---|---|---|
| Accounts | `get_account_details` (bank details, card limit/owed/statements/holders, debit cards) | `update_account` (name, colour, bank, archive), `set_default_account`, `correct_balance` (match the bank) |
| Credit cards | `get_account_details` | `update_card` (limit, statement day, owed, due days), `set_card_statement_paid`, `record_card_payment` (logs a payment already made), `update_card_holder` (caps), `update_debit_card` |
| Transfers | `list_transactions` with `kind: transfer` | `record_transfer` (fee, cross-currency `received_minor`), `update_transfer`, `delete_transaction` |
| Loans | `list_loans` | `record_loan_payment`, `update_loan` (due date, note) |
| Goals | `list_goals` | `create_goal`, `update_goal`, `record_goal_saving` (contribute/withdraw) |
| Budgets | `get_budgets` | `create_budget`, `update_budget` (amount, alerts, mute, pause, archive), `reallocate_budget`, `delete_budget` |
| Subscriptions & bills | `list_bills`, `find_recurring` | `create_subscription`, `update_subscription` (incl. price change), `delete_subscription`, `record_bill_payment`, `skip_bill` |
| Categories | `list_categories` | `create_category`, `update_category` (rename, hide), `manage_subcategory` (move, promote, merge) |
| Tags | `list_tags` | `create_tag`, `update_tag`, `delete_tag`, `tag_transactions`, `manage_tag_rule` |
| Merchant rules | `list_merchant_rules` | `set_merchant_rule`, `delete_merchant_rule` |
| SMS tracking | `list_sms_review`, `list_sms_senders` | `dismiss_sms`, `manage_sms_sender` (track, pause, link to an account), `manage_sms_skip_pattern` |
| Preferences | `get_preferences` | `update_preferences` (savings target, fixed categories, bill reminder lead) |

- **Deletes always wait for the user's OK on the phone** (budgets, subscriptions, tags, merchant rules, merging subcategories), and so do changes to 5 or more items. Tell the user to check the Spencio app before you call them; a `status: "denied"` means nothing changed.
- Categories are never deleted, only hidden.
- **App-only** (no tool, tell the user to use the app): deleting an account (archive instead), card instalment plans and bank-loan EMIs (read-only), group funds, accepting or editing an SMS from the review queue (you can only dismiss), turning SMS tracking on/off, app lock, backups, notifications, primary currency.
- Ids: accounts, categories and most items are integers; transactions, budgets, goals, subscriptions, tags, card holders and debit cards use string ids. Always take them from a list or details tool.

## Editing turned off

If a write says editing is turned off, the user switched off "Edit and add transactions" in Settings › AI assistants. Reading still works. Tell them where the switch is; don't try other write tools.

## Style

- Show money with the currency symbol and thousands separators (Rs 18,640, ¥980), and name the date range.
- Keep answers short: a total, then the few lines that explain it. Offer the breakdown rather than dumping every row.
- Don't reveal raw ids to the user unless they ask.
