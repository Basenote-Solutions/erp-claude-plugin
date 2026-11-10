---
name: reports
description: Covers Base Note ERP reports and dashboards. That means the reports catalogue (open pipeline, won and lost by source, project profit, revenue by service, stock value, outstanding payments, low stock, compliance outstanding, the accountant export), running a report inline, exporting a CSV, the Home dashboard widgets and the user's own preferences for them, and the This week page (what moved, what to chase, overdue tasks, payments due, and won, invoiced and received against last week). Use it when the user asks for a report, figures, totals, a summary, a CSV or spreadsheet export, what is owed or overdue, profit or margin, revenue, stock value, the pipeline value, what happened this week, what needs chasing, or what needs attention today.
---

# Reports, Home and This week

Reports are read-only queries over the whole ERP. Each one needs its area's view permission:
`view_crm`, `view_projects`, `view_money`, `view_inventory` or `view_compliance`. Amount columns
follow the price permissions:

- `view_sales_price` for sales amounts;
- `view_cost` for costs;
- `view_supplier_price` for supplier amounts.

A column the user may not see is left out of the rows and the CSV, never shown as zero. Say that
it is hidden rather than reporting 0.

## Tools and operations

- `run_report` without a name lists the reports this user may run, with each one's parameters
  (name, kind, mandatory, choices) and the columns they will see. Call it first when unsure. It
  is the same as `reports_list`.
- `run_report` with `name` and `params` runs a report inline (`reports_retrieve`):
  - Results are capped at 2,000 rows and a short time limit. Past that, the API returns 422
    `export_limit_exceeded`; export the report instead.
  - The tool shows the first `max_rows` (default 200, up to 500) and sets `rows_shown` when it cut
    the result.
- CSV export:
  1. Call `write` with `reports_runs_create` [key], path `pk` set to the report name, the body
     set to the parameters themselves (for example `{"date_from": "2026-01-01", "date_to":
     "2026-03-31"}`) and an `idempotency_key` (a new UUID; reuse it with the same body to retry).
  2. Poll `report_runs_retrieve` until `state` is `done` (or `failed`, with an `error_code`);
     before that it is `queued` or `running`.
  3. Call `download_file` with `report_run_id`. It returns the CSV as text when small, and always
     a link that lasts 5 minutes (`report_runs_download_retrieve`).
- `reports_runs_list` lists the user's own exports of one report, newest first.
- `dashboards_home_retrieve` returns the user's Home: `layout` (their widget codes in order),
  `widgets` (each one computed), `available` (every widget they may read, with its title) and
  `customised`. The `my_work` tool includes it, with who the user is, their open tasks and their
  unread notifications.
- `dashboards_week_retrieve` returns the This week page (see below). It takes no parameters.
- `me_preferences_retrieve` returns the user's own `timezone`, `date_format` and `home_widgets`
  (blank or empty: the default), what is in force under `effective`, the company defaults under
  `site`, and `widgets` (every widget code with its label).
- `me_preferences_partial_update` with `home_widgets` (an ordered list of widget codes) shows,
  hides and orders the user's own Home widgets; an empty list goes back to the role's default.
  It also sets the user's `timezone` and `date_format`.
- For live CRM totals: `deals_summary_retrieve` (per track and stage) and `tracks_board_retrieve`
  (one track's board).

## The reports and what each answers

| Name | Answers | Parameters |
|---|---|---|
| `pipeline` | Open deals by track and stage, with value and GBP value | `track`, `owner` (ids) |
| `won_lost_by_source` | Deals closed in a range, by track and source, with the win rate and won and lost GBP | `date_from`*, `date_to`*, `track` |
| `project_profit` | Per project: revenue (issued invoices, net, GBP) less stock used, services billed, expenses and project costs, giving profit and margin | `project`, `stage`, `client` |
| `revenue_by_service` | Issued invoices per month and service, net, GBP | `date_from`*, `date_to`*, `service` (`pl`, `blend`, `alchemy`, `upsell`, `other`) |
| `stock_value` | Base Note's own stock per item and location at average cost (client stock carries no value) | `location`, `item` |
| `outstanding_payments` | Issued invoices and posted bills not fully paid, with what is owed, owed in GBP and days overdue | `direction` (`receivable`: owed to us; `payable`: we owe) |
| `low_stock` | Items below their reorder level, with on hand, reserved and available | none |
| `compliance_outstanding` | Compliance documents Not started or In progress, by scope and project (cancelled projects left out) | `project`, `scope` (`project_sku`, `recipe`, `company`), `status` (`not_started`, `in_progress`) |
| `accountant_invoices`, `accountant_payments`, `accountant_bills`, `accountant_expenses` | The accountant export: every document in the range in its own currency and in GBP. Cancelled and void ones are listed with their status | `date_from`*, `date_to`* |

\* required. Dates are `YYYY-MM-DD`, and a range spans at most 366 days (422
`report_window_too_large`). Id parameters take record ids: find them with `search`, or with
`users_lookup_list` for owners and `tracks_list` for tracks.

Notes on reading the figures:

- `project_profit` needs `view_projects`, `view_sales_price`, `view_cost` and
  `view_supplier_price`. Stocked purchases count when they are used, not when billed, so nothing
  counts twice. A project without revenue has no margin.
- `won_lost_by_source` counts a deal on the day it closed, in the user's own timezone.
- Owed in GBP is the document's frozen GBP total less the paid part converted once at its rate. An
  unpaid document owes exactly its GBP total.
- An inline row that names a record carries its `record_id` when the user may open it: pass it to
  `get_record` to give the user the web link. A CSV export never has it.

## Home widgets

| Code | Shows | Needs |
|---|---|---|
| `pipeline` | Open pipeline value by track and stage | `view_crm` |
| `projects` | PL projects by stage, with each one's next task deadline | `view_projects` |
| `money_due` | Deposit and balance invoices still owed (drafts too), by due date | `view_money` |
| `overdue_tasks` | Overdue tasks by person | (none) |
| `my_tasks` | The user's open tasks | (none) |
| `low_stock` | Stock below its reorder level | `view_inventory` |
| `compliance_outstanding` | Compliance documents outstanding per project | `view_compliance` |
| `licences` | Active licences and renewals in the next 30 days | `view_money` |
| `lab_work` | Recent bench trials and briefs | `view_recipes` |

The defaults depend on the user's role, and a widget the user may no longer read is left out.

## This week

`dashboards_week_retrieve` covers Monday to Sunday of the user's current week (`week_start`,
`week_end`). A section is absent without its read permission, and a figure without its price
permission. Lists show at most 20 rows; the counts beside them cover everything.

| Key | Shows | Needs |
|---|---|---|
| `figures` | `won` (won deal value), `invoiced` (invoices issued) and `received` (payments in), each `this_week` against `last_week`, in GBP | `view_sales_price`, with `view_crm` for `won` and `view_money` for the other two |
| `tasks` | Overdue tasks (the count and the oldest) and how many fall due by the week's end | (none) |
| `deals` | Stage moves this week (all, forward, won, lost, and the latest), new deals and new leads | `view_crm` |
| `chasing` | Open deals and leads waiting on the client since our last email, 7 to 60 days ago | `view_crm` |
| `missing` | Open deals without a value (`unpriced`) and without an open task (`no_next_step`) | `view_crm` |
| `projects` | Project stage moves this week | `view_projects` |
| `payments_due` | Invoices (`receivable`) and bills (`payable`) still owed and due by the week's end, overdue ones included, with GBP totals | `view_money` |

## Common jobs

1. **"What's owed to us?"**:
   1. Call `run_report` `outstanding_payments` with `{"direction": "receivable"}`.
   2. Sort by days overdue and total the GBP owed from the rows.
   3. Offer to chase payment (see the money skill).
2. **"How did we do last quarter?"**: call `revenue_by_service` and `won_lost_by_source` for the
   quarter's dates.
3. **"Is project PL-… profitable?"**: find the project id, then call `run_report`
   `project_profit` with `{"project": "<id>"}`.
4. **Accountant pack**: export each of the four `accountant_*` reports for the period with
   `reports_runs_create` [key] (a new key per report), then download each CSV.
5. **"What needs me today?"**: call `my_work`, then summarise the overdue tasks, notifications and
   widgets.
6. **"How is this week going?"**: call `dashboards_week_retrieve`, then give the figures against
   last week, what moved, what to chase and what is due. Name the sections the user may not see
   as hidden rather than empty.
7. **"Put the pipeline on my Home"**: read `available` from `dashboards_home_retrieve`, then send
   the whole ordered list in `home_widgets` with `me_preferences_partial_update`.

## Pitfalls

- Report parameters are ids and codes, not names. Resolve names first.
- The `money_due` widget lists deposit and balance invoices only, and This week's `payments_due`
  only what falls due by the week's end. For everything owed, run `outstanding_payments`.
- This week's `invoiced` figure is gross, tax included; the `revenue_by_service` report is net.
  Don't set one against the other.
- Don't do arithmetic on hidden columns, and don't present a partial total as the whole. If
  `rows_shown` is set, export the report for the full figures.
- An export that failed needs narrower parameters, or the permission the user lost. Its
  `error_code` says which.
- A download link expires after 5 minutes. Fetch a new one with `download_file` rather than
  reusing an old link.

## Confirm with the user before

Starting an export, which queues a background job, and changing the user's Home widgets, timezone
or date format. Running a report inline or reading a dashboard only reads, so it needs no
confirmation.
