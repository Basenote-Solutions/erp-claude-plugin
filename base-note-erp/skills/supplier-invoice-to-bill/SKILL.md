---
name: supplier-invoice-to-bill
description: Step-by-step workflow in BaseNote ERP from a supplier's invoice (a PDF, a photo or figures the user gives) to a posted bill that costs the right projects and shows what was really paid - the supplier as a party, one bill per invoice with a line per item and its tax, a project on each line, the invoice file on the bill, posting, a payment plan for a deposit and a balance, each payment with the GBP that left the bank and the bank's reference, the estimates to remove, and the import tool as the quicker route. Use when the user hands over or mentions a supplier's invoice, a printer's, factory's or freight bill, a pro-forma to pay, a deposit paid to a supplier, a bank transfer to record against a supplier, or asks where a project's supplier costs go. A supplier's invoice is never recorded as a project cost line.
---

# Workflow: from a supplier's invoice to a paid bill

**A supplier's invoice is a bill. It is never recorded as a project cost line**, a note or a
task. Only a bill keeps the supplier's own invoice number (so the same invoice is not entered
twice), its tax, its file, what is still owed and each payment at the amount that really left
the bank, and a project's cost is worked out from its bills. A project with no SKUs and no
purchase order takes bills like any other: a bill needs neither. A project cost line is only
an estimate of a cost that has not been invoiced yet (step 9).

The money skill describes each record; this is the order to work in. Call `describe_operation`
before the first use of each operation. Amounts, quantities and rates are JSON strings. Bills
and payments out need `view_money`, `manage_money` and `view_supplier_price`; without them the
amounts are absent, never zero: stop and say who must do it. Edits and commands send the
record's current `expected_revision`; operations marked [key] take an Idempotency-Key (a new
UUID per intent, the same one to retry after an unknown outcome).

## 1. Read the invoice

- From the PDF, photo or figures the user gives, take the supplier's name, their invoice
  number, the invoice date and due date, the currency, each line (what, quantity, unit price,
  tax) and the totals. Read them back to the user before writing anything, and ask for what
  the document does not say: most often which project each line is for.
- A PDF or a photo can be read by the import tool instead (see "The quicker route").

## 2. The supplier is a party

- Find it with `parties_list` (`role=supplier`, `search`) or `search` (type `parties`). Look
  before creating, so one supplier is one party.
- New: `parties_create` (`display_name`, `kind`, `roles: ["supplier"]`, `supplier_types`), its
  address with `parties_addresses_create` and its contact with `parties_contacts_create`. A
  party that exists under another role gets the supplier role with `parties_update` (a PUT:
  send every field back; `roles` replaces the set).
- The address and the contact go on the party, where every later document finds them, never
  into a note.
- The bank or pay-account details an invoice prints go on its bill, in `pay_to`
  (`bills_create`, `bills_update`; leave the key out of an update to keep them), never on
  the supplier in the same step. The supplier's own `payment_details` (one `Label: value`
  per line) are what the bill's are compared with, so copying an invoice's details there
  first would hide a changed account.
- Read back, `pay_to_differs` true means the invoice names an account number that the
  supplier's saved `payment_details` do not hold: tell the user and let them check with
  the supplier, by a route they already trust, before anything is paid.
- The supplier's `payment_details` are set or changed as a step of their own, once the
  user confirms the details are the supplier's (for a new supplier too): `parties_update`,
  with the key left out of every other party save. Only holders of `manage_money` read
  and write them, and `pay_to` and `pay_to_differs` are absent without it (sending
  `payment_details` without it answers 403).
- Never retype payment details into a note, a task or the bill's notes, which every
  reader of that record sees.

## 3. Draft the bill: one bill per invoice

- `bills_list` with `supplier` shows what is already entered for this supplier.
- `bills_create`: `supplier`, `supplier_ref` (their invoice number, exactly as printed),
  `bill_date` (the invoice's date), `due_date`, `currency` (`fx_rate` null: the latest rate on
  or before the bill date) and one line per item on the invoice: `description`, `qty`,
  `supplier_unit_price` (before tax), `tax_rate` (an id from `tax_rates_list`; blank: the
  default rate) and `project_id` (step 4). 409 `bill_duplicate`: this supplier already has a
  live bill with that number, so open that one instead of making a second.
- Set the answer's `bill_net_total`, `bill_tax_total` and `bill_gross_total` against the
  invoice's own totals. They are the server's: do not add lines up yourself. A difference
  means a quantity, price or tax rate was read wrongly: correct the draft with `bills_update`
  (a PUT of the whole bill).
- For an order placed in the ERP, `purchase_orders_bill_create` [key] drafts the bill from the
  purchase order instead (buy-to-stock, step 6).

## 4. Name each line's project

- A line's `project_id` is the project its cost belongs to (find the id with `search`, type
  `projects`). One invoice can be shared between projects line by line. When one printed line
  covers two projects, ask the user how it splits and enter a line for each.
- A stocked item's line names no project (422): stock is costed to a project when it is used,
  so naming one here would count it twice. A line that belongs to no project (an overhead)
  is left without one.
- On a bill that already exists, draft or posted: `bills_projects_create` [key]
  (`expected_revision`, `lines` of `{line_no, project_id}`; null clears one; the lines not
  named stay). No amount moves.

## 5. Put the invoice file on the bill

- A file you hold: `upload_file` with `record_type` `bills` and `record_id`. It stores the
  file (PDF, PNG or JPEG) and attaches it to the bill in the same call.
- A file the user holds: `ask_for_file` (`record_type` `bills`, `record_id`, `what`: the
  supplier's invoice). It shows an upload box in the conversation where the app has one and
  always answers `upload_page`, a link to give the user. Either way the file lands on the bill.
- Read the bill's `files` to confirm. Never send the user to look for a Files section.

## 6. Post it

- Show the supplier, their number, the lines with their projects and the totals. On a yes,
  `bills_post_create` [key] with `expected_revision`: the totals freeze at the bill date's
  rate. A posted bill that is wrong is cancelled with `bills_cancel_create` [key] (a `reason`;
  refused while payments are allocated) and entered again.

## 7. A payment plan, when it is paid in parts

- A deposit and a balance (or any stages) are one bill with a plan, not two bills:
  `bills_instalments_update` (a PUT: `expected_revision` and `instalments` in order, each a
  `label`, a `bill_instalment_amount` and an optional `due_date` and `note`). The parts must
  add up to the bill's gross total (422); an empty list removes the plan. It works on a draft
  or a posted bill and moves no amount.
- The bill then shows each part's paid and outstanding amounts (payments fill the parts in
  order) and what is owed next: `next_due_date` with `next_due_amount`.

## 8. Record each payment when the money has left

- The user pays from their bank; the ERP records it. Ask for the bank's receipt or statement
  line and take three things from it: the day, the GBP that left the account (fees included)
  and the bank's own reference for the transfer.
- `payments_create` [key]: `direction` `out`, `party` (the supplier), `payment_date`, `amount`
  and `currency` (the bill's currency: what this payment settles), `amount_gbp` (the GBP that
  left the bank; it stands in place of `fx_rate`, and both together answer 422),
  `external_ref` (the bank's reference: a second payment carrying the same one is refused, so
  a transfer is never recorded twice), `method` and `allocations` `[{bill, amount}]`.
- A part payment is a payment allocated for less than the bill still owes. An allocation
  above what it owes answers `over_allocation`.
- `amount_gbp` is the bank's figure: never work it out from a rate. 422 on `amount_gbp` means
  it is far from the day's rate, most often the foreign amount typed in its place: check the
  receipt with the user.
- The bill reads Part paid, then Paid.

## 9. What the project shows, and the estimates

- `dashboards_projects_retrieve` with `project`: `bills_paid_gbp` is what left the bank for
  the project's bill lines and `bills_owed_gbp` what is still to pay at the bill's rate, both
  with VAT. A bill shared between projects counts on each for its own lines.
- A cost that is not invoiced yet (freight quoted but not billed) may stand as a project cost
  line, `projects_cost_lines_create`: an estimate, shown as `project_costs_gbp`. Once its bill
  is in, remove it with `project_cost_lines_destroy` or reduce it with
  `project_cost_lines_partial_update`, or the cost counts twice. Read
  `projects_cost_lines_list` whenever a bill is posted for a project.

## The quicker route: the import tool

- Import type `supplier_bill` reads a supplier's invoice (a PDF or a photo, or its template
  CSV) into draft bills to check: one bill per supplier and invoice number, a row per line,
  the line's project by its number in `project`, the printed payment details in `pay_to`.
  An invoice that is already a bill is refused on its row and the bill is named. A PDF or
  photo that was read stays on each bill it made.
- A bank statement's CSV export has its own import type (`import_types_list` names it): it
  is read by the ERP itself and never by an AI model, and gives one supplier payment per
  transfer with the GBP that left the bank and the bank's reference, allocated to the bill
  a row names, else to the supplier's oldest unpaid bills. A transfer already recorded is
  left out, so the same statement can be imported again.
- `import_types_list` names the import types this user may run; the steps (upload with
  doc_type `import`, `imports_create` [key], check, apply) are in the erp-basics skill.
- A document that is not a template CSV is read by an AI model outside the ERP, prices
  included: say so and get the user's go-ahead first.
- What an invoice import gives is a draft, not a posted bill: compare it with the invoice,
  check each line's project and the `pay_to_differs` flag, then go on from step 6.

## Pitfalls

- A project without SKUs or a purchase order is no reason for a cost line: `bills_create`
  needs only the supplier and the lines.
- A receipt with no supplier invoice behind it (a taxi, a courier paid at the counter) is an
  expense linked to its project (money skill).
- Two invoices from one supplier are two bills, even for one order. One invoice paid in two
  parts is one bill with a plan.
- `supplier_ref` is the supplier's number; the ERP gives the bill its own BILL- number.
- Supplier names, prices and payment details are confidential: none of them goes into a brief
  or anything else meant for outsiders.

## Confirm with the user before

Creating or changing the supplier (and its payment details), drafting the bill (supplier,
number, lines, tax, projects, totals), attaching the file, posting, setting the payment plan
or the lines' projects, each payment (date, amount, the GBP that left the bank, the bank's
reference, the bill), removing an estimate, and starting or applying an import. A posting and
a payment are real: a mistake is put right by cancelling the bill or voiding the payment, each
with a reason.
