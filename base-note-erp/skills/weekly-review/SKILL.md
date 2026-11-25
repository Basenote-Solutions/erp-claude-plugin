---
name: weekly-review
description: Step-by-step workflow in BaseNote ERP for the Monday tidy-up - reading This week and the projects summary, then walking the user through overdue cards person by person (re-date, reassign, finish or archive each), cards linked to no record, projects standing still, deals without a value, and the invoices and bills due, ending with a short written summary. Use when the user asks for the weekly review, a Monday review, a tidy-up or clean-up of the task board, what is overdue or stuck across the company, which projects have stalled, which deals have no value, or what needs attention this week.
---

# Workflow: the weekly review

Once a week the company clears what has drifted: cards past their date, cards about nothing
in particular, projects nobody has touched, deals with no value, money due. The ERP finds
them; the user decides what happens to each. Your part is to read, lay each group out, ask,
and then make the changes the user chose.

## Working rules

- **Reads first.** Read everything in step 1 before proposing a change. Reads need no
  confirmation.
- **Ask, do not decide.** Never re-date, reassign, finish, archive or link a card on your own
  judgement, however obvious it looks: a card that seems stale may be waiting on something
  the ERP does not hold.
- **One confirmation per group of writes.** Collect the user's answers for a group (one
  person's overdue cards, the unlinked cards), read the list of changes back, get one yes,
  then write them. Each card edit sends the card's current `revision` as `expected_revision`.
- A section the user's role may not read is absent from the answers: say it is hidden, not
  that it is empty. Call `describe_operation` before the first use of each operation.

## 1. Read the week

- `my_work`: who the user is and their business date. When their Home carries the
  `needs_attention` widget, it counts the overdue cards, the cards about no record, the still
  projects, the deals without a value and the invoices and bills due by Sunday (a count is
  null when the role may not read what it counts).
- `dashboards_week_retrieve`: the lists behind those counts, at most 20 rows each beside a
  count of them all (the reports skill names every section).
- `dashboards_projects_retrieve`: every open project with its money, `next_step`,
  `days_still` and `missing`.

Tell the user the size of each group, then take the groups in this order.

## 2. Overdue cards, person by person

- `tasks.people` in the week's answer lists each owner with their count and oldest date. For
  one person, `tasks_list` with `owner` and `due` `overdue` gives their cards (overdue by the
  card's own date or an open step's).
- For each card ask which it is: **re-date** it or **reassign** it (`tasks_update`, a PUT:
  send `title`, `notes`, `owner_id`, `due_date` and `priority` again), **finish** it
  (`tasks_done_create`), **archive** it because it no longer matters (`tasks_archive_create`:
  kept, off the board), or leave it.
- A card overdue through one of its steps takes the change on the step:
  `task_checklist_update` (`due_date`, `owner_id`, or `done` true). A step that waits for an
  action (`completes_on`) is not ticked by hand: do the action if it is due, or leave the
  step.
- Finish one person before the next, so each confirmation is short.

## 3. Cards about no record

- `tasks.unlinked` lists open cards that have no record and that nobody has marked as about
  none; `tasks_list` with `unlinked=true` pages through all of them.
- For each, ask what it is about. A number the user gives (a project's PL-, a deal's D-, a
  purchase order, work order or parcel) is found with `task_records_retrieve` (`number`; 404:
  nothing the user may read carries it). Link it with `tasks_record_update`
  (`expected_revision`, `record_type`, `record_id`), or mark it as about none with
  `no_record: true`.
- You may suggest a record from the card's title, but the user picks it. A link shows the card
  on that record's page and to its readers only.

## 4. Projects standing still

- `projects.stalled` lists open projects, not on hold, where no stage has changed and no step
  has been ticked for 14 days; the projects summary gives each one's `days_still`,
  `next_step` and `missing`.
- For each, ask two things: what is the next step, and who owns it. Put the answer on the
  project: a step on its card (`task_checklists_items_create` with its person and
  `due_date`) or a card of its own (`tasks_create` with `record_type` `projects.project` and
  `record_id`).
- If the project has moved on in reality, its stage is behind: move it (pl-projects). If it
  is waiting on the client for good reason, put it on hold with the reason
  (`projects_move_create` [key], `to` `on_hold`).
- A project whose `missing` is long (no SKUs, no quote, no costs) needs catching up, not a
  card: offer the backfill-an-order skill.

## 5. Deals without a value

- `missing.unpriced` lists open deals with no value, `missing.no_next_step` those with no
  open task. A deal without a value is flagged, never stopped: the review is where it gets
  one.
- Ask the value and currency of each, then `deals_update` (a PUT: read the deal and send
  every field back with `estimated_value` and `currency`). A deal with a preferred package
  takes its value from the package (crm-and-quotes).
- For a deal with no next step, ask for one and make its task, linked to the deal.

## 6. Invoices and bills due

- `payments_due` lists the invoices (`receivable`) and bills (`payable`) owed and due by the
  week's end, overdue ones included. `run_report` `outstanding_payments` shows everything
  owed either way.
- Invoices: say which are overdue and offer to chase each (a payment request emails the
  client: the money skill, and its confirmation, apply).
- Bills: say which fall due and what each needs next (a bill with a payment plan shows
  `next_due_date` and `next_due_amount`). The user pays from their bank. When they say a
  transfer has gone, record it with the GBP that left the bank and the bank's reference
  (supplier-invoice-to-bill, step 8).

## 7. The summary

End with a short written summary in the conversation, in the user's words where you can:

- what changed, by group (cards re-dated, reassigned, finished, archived, linked; projects
  given a next step; deals given a value);
- what was left as it is, and why;
- the three or four things that matter most this week, each with its owner and date.

Give the `web_url` of anything the user may want to open. Write nothing into the ERP for the
summary unless the user asks.

## Confirm with the user before

Each group of card changes (the list read back first), every link or "about none" mark, a new
card or step, a stage move or hold, a deal's value, a payment request and a recorded payment.
Reading the dashboards and lists needs no confirmation.
