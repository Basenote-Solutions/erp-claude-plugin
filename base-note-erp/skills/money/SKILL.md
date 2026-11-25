---
name: money
description: Covers BaseNote ERP money. That means invoices (from a quote, deposit and balance, a contract's fee, issue, cancel), payments received and paid out and their allocation, supplier bills with a project on each line and a payment plan, what a project costs, expenses with receipts and their categories, emailed payment requests with a CC and the invoice PDF, Blend and Alchemy Engine licence billing, currencies, exchange rates, tax rates and the company details printed on documents (address, bank details, registration line). Use it when the user mentions an invoice, issuing, a payment, paid or unpaid, overdue, chasing payment, a payment request, a bill, a supplier's invoice, an expense, a receipt, an expense category, a licence fee or renewal, an exchange rate, VAT, a tax rate, bank details or the company details on a PDF.
---

# Money

BaseNote keeps light money records, with no ledger: **quotes and invoices** to clients,
**payments** in and out (allocated to invoices and bills), **bills** from suppliers, **expenses**
paid outside a bill, and **licences** for the Blend and Alchemy Engines, which bill on a cycle.
The base currency is GBP; every document keeps its own currency and rate.

## Before any call

- Call `describe_operation` before you use an operation for the first time in a conversation, to
  get its exact body. Send amounts, quantities and rates as JSON strings (`"1200.00"`).
- Hidden values are missing from the response, never zero: say they are hidden. Invoice, quote,
  licence and payment-in amounts need `view_sales_price`; bills, payments out and expenses need
  `view_supplier_price`.
- Reads need `view_money`, writes `manage_money`. Creating, editing, issuing or posting a
  document, recording or allocating a payment and requesting a payment also need the price
  permission of their amounts. Currencies, exchange rates, tax rates, expense categories and the
  licence lead days are settings and need `manage_settings`; changing the company details needs an admin.
- Every edit sends `expected_revision`. A 409 `stale_revision` means someone changed the record:
  read it again, apply the change again and confirm. Edits are PUTs that replace the whole draft,
  so read it first (tax rates and expense categories are PATCHes).
- Operations marked [key] take an Idempotency-Key: pass `idempotency_key` (a new UUID you make
  per intent). After an unknown outcome (a timeout, `idempotency_in_flight` or
  `concurrency_conflict`), retry with the key you passed and the same body. A new key could post
  twice.

## Records and statuses

- **Invoice**: kind Standard, Deposit, Balance or Licence. Status Draft → Issued (numbered INV-)
  → Cancelled. Payment Unpaid, Part paid or Paid; overdue once past due by the user's today.
- **Payment**: numbered PAY-. Received or Paid out. Posted when recorded; Void after a void.
- **Bill**: a supplier's invoice, numbered BILL-. Draft → Posted → Cancelled, with a payment
  status like an invoice's. Each line may name the project its cost belongs to, and a payment
  plan may split the bill into parts.
- **Expense**: numbered EXP-. Recorded, or void.
- **Payment request**: numbered PRQ-. Queued until the email has gone, then Sent, or Failed with
  an `error_code`.
- **Licence**: numbered LIC-. Draft → Active (can be Paused and resumed) → Ended.

## Operations

- **Invoices**: `invoices_list` (filter by status, kind, `service`, `payment_status`, overdue,
  outstanding, client, `issued_from` and `issued_to`, `q`, or `source_type` with `source_id`);
  `invoices_create` (kind `standard` or `licence`); `invoices_retrieve`; `invoices_update`
  (drafts only); `invoices_issue_create` [key] (optional `issue_date`);
  `invoices_cancel_create` [key] (needs a `reason`); `invoices_payment_requests_create` [key]
  (`sent_to`; optional `cc`, `due_date` and `message`).
- **From an accepted quote**: `quotes_invoice_create` with kind `standard` (in full), or `deposit`
  with `deposit_pct` and then `balance`. Asking again returns the live invoice already made.
- **From a contract**: `contracts_fee_invoice_create` [key] drafts a Standard invoice of the
  contract's fee (see the crm-and-quotes skill).
- **Payments**: `payments_list` (filter by direction, party, status, unallocated or dates);
  `payments_create` [key]; `payments_retrieve`; `payments_allocate_create` [key];
  `payments_void_create` [key] (needs a `reason`).
- **Payment requests**: `payment_requests_list` (filter by `invoice` or `state`),
  `payment_requests_retrieve` (`state`, `sent_to`, `cc`, `error_code`, and `attachment_id` for
  the request's PDF).
- **Bills**: `bills_list` (filter by `supplier`, `status`, `outstanding`, dates, or
  `source_type` with `source_id`), `bills_create`, `bills_retrieve`, `bills_update` (drafts
  only); `bills_post_create` [key]; `bills_cancel_create` [key] (needs a `reason`);
  `bills_projects_create` [key] (the lines' projects); `bills_instalments_update` (the payment
  plan); `bills_files_create` (the supplier's scan); `purchase_orders_bill_create` [key] (a
  draft bill of what a purchase order has not billed yet).
- **Expenses**: `expenses_list`, `expenses_create` [key], `expenses_retrieve`,
  `expenses_files_create` (the receipt), `expenses_void_create` [key] (needs a `reason`).
- **Expense categories**: `expense_categories_list` (`archived` true adds the archived ones),
  `expense_categories_create` (`code`, `name`), `expense_categories_partial_update` (`name`),
  `expense_categories_archive_create`, `expense_categories_restore_create`.
- **Licences**: `licences_list` (can filter by `renewal_within_days`), `licences_create`,
  `licences_retrieve`, `licences_update` (Edit terms); `licences_activate_create`,
  `licences_pause_create`, `licences_resume_create`, `licences_end_create` (needs a `reason`);
  `licences_draft_invoice_create`.
- **Currencies and rates**: `currencies_list`, `currencies_create`, `currencies_archive_create`;
  `fx_rates_list`, `fx_rates_create`, `fx_rates_latest_retrieve` (the default rate for a currency
  and date).
- **Tax rates**: `tax_rates_list`, `tax_rates_create`, `tax_rates_partial_update`,
  `tax_rates_archive_create`, `tax_rates_restore_create`, `tax_rates_make_default_create`.
- **Company details**: `site_retrieve` (every ERP user can read it: `legal_name`, `address`,
  `company_number`, `vat_number`, `bank_details`, `registration_line`, the base currency,
  timezone and date format) and `site_update` (a PUT, admin only).
- **Other settings**: `money_settings_retrieve` and `money_settings_update` (`licence_lead_days`,
  0 to 90).
- **PDFs**: an invoice's or quote's `pdf_attachment_id`, or a payment request's `attachment_id`,
  → `download_file` with `attachment_id`.

## Rules that make writes succeed

- **Exchange rates**: `fx_rate` is GBP per one unit of the document's currency (`"0.79"` on a USD
  document means $1 = £0.79). Rates are fetched each morning or entered by
  hand; one entered for a day wins. A document takes the latest rate on or before its date unless
  a rate is typed on the draft; GBP is always 1. `fx_rate_missing` means no rate exists: ask the
  user for it and add it with `fx_rates_create`, never invent one. Rates are append-only, so a
  mistake is corrected with a newer entry.
- **Lines**: a quote or invoice line is `description`, `qty`, `unit_price` and `tax_rate` (blank:
  the default rate). A `description` may run over several lines, up to 2,000 characters: on the
  invoice PDF the first line prints as the line's title and each following line as a bullet (a
  leading `-`, `*` or `•` is dropped). An invoice's stock line also names `item` and `location`.
- **Issue**: numbers the invoice and freezes its totals at the issue date's rate (a typed rate is
  kept). Sets the due date, unless the draft has one, from the client's payment terms, else the
  standard term (read the answer's `due_date`). Sells any stock lines at average cost, all or nothing: with `insufficient_stock` nothing
  is issued. `issue_date` cannot be after the user's today.
- **Cancel**: refused while payments are allocated (`invoice_has_allocations`): void them first.
  A deposit cannot be edited or cancelled while its balance invoice exists
  (`balance_invoice_exists`): cancel the balance first. Cancelling an issued invoice puts its
  stock back at the same value.
- **Invoices from a quote**: in full, or a deposit then a balance. The kinds exclude each other,
  so a quote is never billed twice, and deposits plus balance add up to the quote to the penny.
  Lines keep the quote's tax %.
- **Payment**: needs `direction` (`in` or `out`), `party`, `amount` and `currency`. `allocations`
  is a list of `{invoice or bill, amount}`, each a document of that party (`party_mismatch`), in
  its currency (`currency_mismatch`), and no more than it still owes (`over_allocation`). What is
  not allocated stays on account; use it later with `payments_allocate_create` [key]. A part
  payment is a payment allocated for less than the document owes.
- **What the bank says**: `amount_gbp` on `payments_create` is the GBP that left or reached the
  bank, fees included, taken from the bank's receipt. It stands in place of `fx_rate` (both
  together: 422) and the rate is worked out from it; on a GBP payment the difference from
  `amount` is the fee. Never work it out from a rate yourself. 422 on `amount_gbp`: it is far
  from the day's rate, most often a foreign amount typed in its place. `external_ref` is the
  bank's own reference for the transfer: one payment each, so a second one carrying it is
  refused (422) and a statement line is never recorded twice.
- **Payment request**: only for an issued invoice (409 `money_transition_invalid` otherwise) that
  still owes something (409 `payment_request_nothing_due`). It emails `sent_to` the amount still
  owed, the due date (`due_date`, else the invoice's), the company's bank details and the
  message, with two PDFs attached: the request and the invoice itself, lines and prices included.
  An optional `cc` is copied in, and replies go to it. Send it only to the client contact the user
  names, add a `cc` only when the user gives one, and put nothing in `message` beyond what the
  user approved, never costs or other figures. A request is never edited or recalled: after a
  Failed one (`email_rejected`: the provider refused the message; `email_unavailable`;
  `pdf_unavailable`), make a new one.
- **Bill**: supplier, `supplier_ref` (the supplier's own invoice number: one live bill per
  number and supplier, 409 `bill_duplicate`), bill date and lines priced by
  `supplier_unit_price`, before tax. Posting freezes the totals at the bill date's rate.
  Cancelling is refused while payments are allocated (409 `bill_has_allocations`). **A
  supplier's invoice is always a bill, never a project cost line**: the
  supplier-invoice-to-bill skill has the whole job.
- **A project on each bill line**: a line's `project_id` names the project its cost belongs
  to, so one invoice is shared between projects line by line. A stocked item's line names
  none (422): stock is costed to its project when it is used. `bills_projects_create` [key]
  (`expected_revision`, `lines` of `{line_no, project_id}`; null: no project) sets them on a
  draft or a posted bill without moving any amount.
- **Payment plan**: `bills_instalments_update` (a PUT: `expected_revision` and `instalments`
  in order, each `label`, `bill_instalment_amount`, optional `due_date` and `note`) splits a
  draft or posted bill into parts, such as a deposit and a balance. The parts must add up to
  the gross total (422), and posting refuses a plan that no longer does; an empty list
  removes it. The bill's `instalments` then show each part's paid and outstanding amounts,
  payments filling them in order, and `next_due_date` with `next_due_amount` say what is owed
  next (without a plan: the bill's own due date and what it still owes).
- **Payment details**: a supplier's bank or pay-account details are the party's
  `payment_details` (one `Label: value` per line), read and written only with `manage_money`:
  leave the key out of a party save to keep them. They are never copied into notes. A bill
  keeps the details its invoice prints in `pay_to`; `pay_to_differs` true means they name
  an account number the saved ones do not hold: have the user check with the supplier
  before paying. Both are absent without `manage_money`.
- **What a project costs**: stock used for it at average cost, its bill lines and its
  expenses, both with their VAT, and its cost lines. The paid part of a bill counts at the
  GBP that left the bank, the unpaid part at the bill's own rate. A bill line counts for its
  own project, else for the bill's (its source, or its purchase order's project); a stocked
  item's line counts when the stock is used, so nothing counts twice. A project cost line is
  an estimate of what is not billed yet: remove or reduce it once the real bill is in.
  `dashboards_projects_retrieve` shows the split (reports skill).
- **Expense**: `expense_amount` is as on the receipt, tax included (the tax is worked out from
  `tax_rate`). It is recorded at once. Link a project with `source_type` `projects.project` and
  `source_id` so it counts in the project's profit. Upload the receipt with `upload_file`
  (doc_type `expense_receipt`) and pass its id as `receipt_attachment_id`, or link it afterwards
  with `expenses_files_create`. `paid_by` is the user who paid; null means the company's account.
- **Expense categories**: Travel, Shipping, Samples, Software, Marketing, Office, Professional
  fees and Other are seeded. A `code` is unique and fixed once made; only the name changes. An
  archived category stays on its expenses and is refused for new ones (422): restore it first.
- **Company details**: `site_update` changes what documents print: the legal name, the `address`
  (one line per line), the company and VAT numbers, the `bank_details` (a `Label: value` line
  prints as a row on the invoice) and the `registration_line` (the footer line of invoices and
  contracts). It is a PUT that always needs `base_currency`, `timezone` and
  `fiscal_year_start_month`: read `site_retrieve` and send those back unchanged; a field left out
  stays as it is. The base currency is locked once a money document or stock move exists, and the
  fiscal-year start once a document has a number (409 `site_settings_locked`). A quote's or
  invoice's PDF is rendered once and a payment request keeps the bank details of its day, so a
  change shows on later documents only.
- **Licence**:
  - Activate (and Resume) needs a fee, a start date and an active tax rate.
  - Every hour the system drafts the next invoice once the next invoice date is within
    `licence_lead_days` (default 7), then moves the date on by the cycle (Monthly, Quarterly or
    Annual). `licences_draft_invoice_create` does it now (`licence_not_active` unless active).
  - A reorder of accords or oils is an `invoices_create` with `source_type` `money.licence`.
- **Tax rates**: Standard 20 % (the default), Zero 0 % and Exempt 0 % are seeded; on the wire the
  rate is `tax_pct`. The % is frozen once a document uses it (`tax_rate_in_use`): create a new rate
  instead. The default rate is never archived, nor one an active or paused licence bills at.

## Common jobs

1. **Issue and chase**: `invoices_retrieve` and show the lines and totals; on a yes,
   `invoices_issue_create` [key]; give the PDF with `download_file`. To email it, confirm the
   address, any CC and the message, then `invoices_payment_requests_create` [key], and check its
   `state` later with `payment_requests_retrieve`.
2. **Money received**: find the client's issued invoices with `invoices_list` (client,
   outstanding); `payments_create` [key] (direction `in`) with the amount, the date it landed,
   reference and allocations (for a foreign payment, `amount_gbp`: what reached the bank);
   report the new payment status.
3. **Supplier bill**: follow the supplier-invoice-to-bill skill. In short: the supplier as a
   party, `bills_create` (or `purchase_orders_bill_create` [key]) with a project on each line,
   the invoice file on the bill (`upload_file` with `record_type` `bills`, or `ask_for_file`),
   `bills_post_create` [key], a plan with `bills_instalments_update` when it is paid in parts,
   then each `payments_create` [key] (direction `out`) with `amount_gbp` and `external_ref`
   from the bank's receipt, allocated to the bill.
4. **Expense**: upload the receipt, then `expenses_create` [key] with a category from
   `expense_categories_list`.
5. **Licence**: `licences_update` for the terms, `licences_activate_create`,
   `licences_draft_invoice_create`, then issue the invoice.
6. **What is owed**: `run_report` `outstanding_payments`, or `invoices_list` with overdue (see the
   reports skill).
7. **Change what documents print**: `site_retrieve`, show the user the company block, then
   `site_update` with the three required fields unchanged and the fields to change.

## Pitfalls

- A supplier's invoice typed as a project cost line has no supplier number, no file, no tax
  and no payment: enter the bill, then remove the line.
- A deposit and a balance paid to a supplier on one invoice are one bill with a payment plan,
  not two bills.
- A payment request emails the client as soon as the worker runs. Check the address before, not
  after.
- The bullets of a multi-line `description` print on the invoice PDF. The quote PDF prints the
  description as one run of text.
- An invoice drafted from a contract's fee is a Standard invoice, not a project's deposit: it
  opens no project gate.
- `site_update` is not a money setting: it needs an admin, and `manage_settings` does not
  allow it.

## Confirm with the user before

Issuing, cancelling or editing an invoice; recording, allocating or voiding a payment; sending a
payment request (it emails a client: show the recipient, the CC and the message word for word,
and say that the invoice PDF goes with it); posting or cancelling a bill, setting its lines'
projects or its payment plan, changing a party's payment details; recording or voiding an
expense; activating, pausing, resuming or ending a licence; drafting a licence invoice; adding
exchange rates; changing tax rates, expense categories or the company details that documents
print. State the amounts the user can see and the effect.
