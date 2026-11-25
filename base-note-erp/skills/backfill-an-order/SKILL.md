---
name: backfill-an-order
description: Step-by-step workflow in BaseNote ERP for bringing an order that began outside it (agreed by message, paid by bank transfer, already with suppliers) up to date so that its project page tells the truth - the parties and contacts, the deal's value, the project's SKUs with their packaging spec and target notes, the quote that was really sent, the money already received as an invoice with its payment, supplier invoices as bills, files on their records, tasks linked and owned, and last the stage, which an administrator sets directly with a reason. Use when the user says an order, client or project is not in the ERP yet, is half entered or out of date, asks to catch a project up, or when a project's stage, value or costs do not match what really happened.
---

# Workflow: bringing an outside order into the ERP

An order that began outside the ERP (agreed in messages, paid by transfer, suppliers already
at work) is brought in so that its project page says what is true: who it is for, what is
being made, what came in, what went out and what is left to do.

## Working rules

- Call `describe_operation` before the first use of each operation. Decimals are JSON strings.
  A value the user's role does not show is absent, never zero. Edits send the record's current
  `expected_revision`; operations marked [key] take an Idempotency-Key (a new UUID per intent).
- Record what happened, with its real dates and amounts, from the documents and what the user
  tells you. Never fill a gap with a guess: ask, or leave it and list it at the end.
- Start by reading: `search` for the client, the deal and the project, then
  `dashboards_projects_retrieve` with `project` once there is one. Its `missing` names what
  the project lacks (`skus`, `quote`, `value`, `next_step`, `costs`): that is the work list.
- Before each step, list its writes (records, amounts, dates) and get one go-ahead for the
  group. Postings (issuing an invoice, a payment, posting a bill) and the stage are each
  confirmed on their own.

Work in the order below. Each step gives the next one what it needs, and the stage comes
last, because the normal stage moves read history made inside the ERP.

## 1. Parties and contacts first

- The client is a party with the client role, each supplier one with the supplier role, an
  outside helper one with the partner role. Find them with `parties_list` (look before
  creating); add what is missing with `parties_create`, `parties_contacts_create` and
  `parties_addresses_create` (crm-and-quotes).
- An address, phone number or contact found in a task note or a message goes on the party,
  and you say so. Every later record names a party, which is why this comes first.

## 2. The deal's value

- The order's deal gets its real value with `deals_update` (a PUT: `estimated_value` and
  `currency`, every other field sent back). Until a quote exists, the value of the deal a
  project was won from is the project's expected revenue.
- If the client has committed and the deal is still open, win it with `deals_move_create`
  [key]: a Private Label deal makes the project, once. With no deal at all, make the project
  with `projects_create` (`name`, `client`, `currency`).

## 3. The project's SKUs

- `projects_skus_create` per product, at any quantity from 1: `name`, `fragrance_type`,
  `oil_concentration_pct`, `fill_ml` and `units_ordered`, with what is known of the packaging
  spec (`batch_no`, `barcode`, `box_size`, `box_material`, `label_size`, `label_material`,
  the bottle, cap and packaging fields) and `target_notes` (`top`, `middle`, `base`).
- SKUs are added only before Approved (409 `project_stage_invalid`), which is one reason the
  stage waits. A project already at Approved or later with SKUs missing is first set back to
  an earlier stage by an administrator (step 9), then forward again.

## 4. The quote, if one was really sent

- `quotes_create` with `source_type` `projects.project`, `source_id`, `service` `pl`, the
  lines as they were sent, `quote_date` (the day it went) and `valid_until`; then
  `quotes_send_create`, which numbers it and emails nothing.
- If the client accepted it, `quotes_accept_create`. That needs the quote still valid today
  (`quote_expired`): when its validity has passed, tell the user and agree the `valid_until`
  to record.
- No quote was ever sent: do not make one up. The deal's value stands.

## 5. Money already received

- A payment in belongs to an invoice. From the accepted quote: `quotes_invoice_create` (kind
  `deposit` with `deposit_pct`, or `standard` for the whole; later `balance`). Without a
  quote: `invoices_create` (kind `standard`, `source_type` `projects.project`, `source_id`).
- `invoices_issue_create` [key] with `issue_date` the day it was really invoiced (never after
  today; a foreign currency needs a rate for that day). Issuing emails nothing. Do not send
  a payment request for money that has already arrived.
- `payments_create` [key]: `direction` `in`, `payment_date` the day the money landed,
  `amount`, `currency` and `allocations` `[{invoice, amount}]`; for a payment in another
  currency also `amount_gbp`, the GBP that reached the bank, and the bank's own reference as
  `external_ref`.

## 6. Supplier invoices as bills

- Each supplier invoice is a bill, with its file, a project on each line and each payment at
  the GBP that left the bank: follow the supplier-invoice-to-bill skill.
- Then read `projects_cost_lines_list` and remove any cost line that stood in for an invoice
  now entered (`project_cost_lines_destroy`), or the cost counts twice.

## 7. Files on their records

- The client's brief, designs and approvals go on the project: `upload_file` with
  `record_type` `projects` and `record_id`, or `ask_for_file` when the user holds them. Each
  supplier invoice goes on its bill, and artwork on its SKU (pl-projects).

## 8. Tasks linked and owned

- `tasks_list` with `record_type` `projects.project` and `record_id` shows the project's
  cards. `tasks_list` with `unlinked=true` and `search` finds stray ones: link each with
  `tasks_record_update`.
- Each open card is owned by the person who does it, has a due date, and names any outside
  person with `contact_id` (`tasks_update`). A new card is made with `tasks_create`, which
  needs its `record_type` and `record_id`.
- On the project's own card, a step whose action you performed above (the quote sent, the
  deposit invoice issued) has ticked itself. A step done outside the ERP with nothing left to
  record is ticked with `task_checklist_update` once the user confirms it.

## 9. The stage, last

- A normal move goes one step at a time, and each gate reads what was done in the ERP: a sent
  quote, approved sample rounds, a paid deposit. An order whose history happened elsewhere
  is put at its true stage in one step by an administrator: `projects_set_stage_create` [key]
  (`expected_revision`, `to`, `reason`).
- The reason is kept in the project's history and the owner is told. No stage invoice is
  drafted, which is why the invoices came first.
- For anyone who is not an administrator it answers 403: say that an administrator must do
  it, and stop.
- `projects_retrieve` goes on listing the gates not met: that is information, not a fault.
  From here the project moves the usual way (pl-projects).

## 10. Check the result

- `dashboards_projects_retrieve` with `project`: the expected revenue and where it comes from
  (`expected_revenue_source`), what was invoiced, what the client paid and still owes, bills
  paid and owed, expenses, estimates, the total cost, the margin (shown only to roles that
  see prices, costs and supplier prices), the `next_step`, `days_still` and `missing`, which
  should now be empty.
- Show it to the user with the project's `web_url`, and list what could not be recorded and
  why.

## Pitfalls

- Samples already sent are parcels (crm-and-quotes): a parcel of kind `sample` on the project,
  moved to sent with `sent_on` the day it left. Sample rounds are recorded only while a
  project is at Sampling.
- A supplier's invoice is never typed as a project cost line, whatever the project has or
  lacks. A cost line is an estimate of what is not invoiced yet.
- The quote that counts for a project is the project's own (`source_type` `projects.project`),
  not one on the deal.
- A backdated invoice or payment takes the exchange rate of its own day: `fx_rate_missing`
  means the user must enter that day's rate first (money skill).
- Setting the stage changes only the stage. It does not make the SKUs, quote, invoices or
  bills that the page is read for: without the steps before it, the summary stays wrong.

## Confirm with the user before

Every write, grouped by step as above: creating or changing parties, the deal's value and
winning the deal, each SKU, the quote and its acceptance, each invoice and its issue date,
each payment (date, amount, the bank's GBP and reference), each bill and its payments,
uploads, task changes, and the stage with its reason.
