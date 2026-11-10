---
name: lead-to-invoice
description: Covers the Base Note ERP private-label workflow end to end. It runs from a lead through convert, the deal won, the PL project, the client agreement (contract) and its fee invoice, the quote sent and accepted, samples, the deposit invoice, the emailed payment request, the payment and the balance invoice, up to shipping, with the Blend Engine, Alchemy Engine and Upsell variants. It names each step's operation and what must be true first. Use it when the user wants to take a client from enquiry to invoice, asks what the next step is, where the contract or agreement fits, or why a project or invoice is stuck.
---

# From a lead to the balance invoice

This is the whole private-label sale as the ERP runs it. Each step names its operation and what
must be true before it. The crm-and-quotes, pl-projects and money skills give each area's
details.

**The order matters.** The PL project is created when its deal is **won**. The project's own quote,
drafted from a costing snapshot, is the one its gates and invoices use. So the order is: win the
deal when the client commits, then quote, sample and invoice on the project. A quote made on the
deal before the win does not count for the project. A contract (the client agreement) can be sent
at any point, alongside the stages.

## Working rules

- Call `describe_operation` before you use an operation for the first time in a conversation.
  Send decimals as JSON strings. A hidden value (cost, price) is missing from the response, never
  zero.
- Read the record before each step: edits and moves send its `expected_revision`, and a 409
  `stale_revision` means read it again and retry.
- Operations marked [key] take an Idempotency-Key: pass `idempotency_key` (a new UUID you make
  per intent). After an unknown outcome, retry with the same key and the same body.
- Every write is real and is recorded as the user's. Confirm each step that changes something,
  especially moves, issuing, emails and payments, before running it.
- When a step is refused, report the error code and what it means (listed below). Don't work
  around a gate.

## The steps

| # | Step | Operation | Must be true first |
|---|---|---|---|
| 1 | Record the lead | `leads_create` | A source option exists (`picklist_options_list`, kind `source`). Set Company to the client's business name, which decides the client. |
| 2 | Qualify it (optional) | `leads_transition_create` [key] to `qualified` | The lead is New. A New lead converts too. |
| 3 | Convert to a deal | `leads_convert_create` [key] with the Private Label `track` (from `tracks_list`) | The lead is New or Qualified and its owner is an active user. On `lead_match_ambiguous`, ask which client and pass `client`. |
| 4 | Complete the deal | `deals_update` (a PUT: `estimated_value`, `currency`, `expected_close_date`, with every other field sent back) | An exchange rate for a non-GBP currency exists (`fx_rates_latest_retrieve`), else `fx_rate_missing`. |
| 5 | Work the pipeline | `deals_move_create` [key] through Discovery call, Explainer sent and so on. Log calls with `activities_create` | The stage belongs to the Private Label track. Moving to these stages only tracks the sale. |
| 6 | Win the deal | `deals_move_create` [key] to the track's won stage | The client has committed. This makes the PL project at Choices received with its tasks (the deal's `project`). It happens once. |
| 6a | Send the agreement (optional) | `contracts_create`, `contracts_send_create` [key], `contracts_signed_copy_create`, then `contracts_fee_invoice_create` [key] | Any time, before or after the win. It runs alongside the stages. See "The client agreement" below. |
| 7 | Set up the project | `projects_partial_update` (Deposit %, target delivery, client-supplied oil, formula ownership), then `projects_skus_create` per SKU | SKUs are added before Approved (`project_stage_invalid` after). Units ordered meet the minimum order (`minimum_order`). Each SKU has a formula version (`provider` `manual` plus the recipe revision id) or the client's oil, and the bottle, cap and packaging have supplier items or manual prices. |
| 8 | Cost it | `projects_calculate_create` (stores nothing), then `projects_costing_records_create` [key] | No `cost_missing`. The SKU inputs are saved. The user has `view_cost`. |
| 9 | Draft the quote | `costing_records_quote_create` [key] (`valid_until`, `tax_rate`) | The project is at Choices received, Quoted or Sampling. The snapshot is not stale (`snapshot_stale` means take a new one). The user has `manage_money` and `view_sales_price`. |
| 10 | Send the quote | `quotes_send_create` | The quote is a Draft and its total is not negative. It is numbered QT- and gets a PDF (`pdf_attachment_id` → `download_file`). Nothing is emailed: the user sends it. |
| 11 | Move to Quoted | `projects_move_create` [key] `to` `quoted` | The project's quote is Sent or Accepted (`quote_not_issued`). |
| 12 | Move to Sampling | `projects_move_create` [key] `to` `sampling` | None. |
| 13 | Record the samples | `project_skus_sample_rounds_create`, then `sample_rounds_decide_create` (`approved` or `rejected` with feedback) | The project is at Sampling. Each round is decided once. After a rejection, add another round. |
| 14 | Record the acceptance | `quotes_accept_create` | The quote is Sent and still valid (`quote_expired`). An older accepted quote is superseded unless an invoice from it was issued (`quote_already_accepted`). |
| 15 | Move to Approved | `projects_move_create` [key] `to` `approved` | The quote is accepted (`accepted_quote_missing`) and every live SKU has an approved round (`samples_not_approved`). This drafts the **deposit invoice** (none at 0 %). |
| 16 | Issue the deposit | `invoices_issue_create` [key] | The invoice is a Draft. `issue_date` is not after today. A rate exists for the issue date. This numbers it INV- and sets the due date. |
| 17 | Ask for payment | `invoices_payment_requests_create` [key] (`sent_to`, optional `cc` and `message`) | The invoice is Issued and still owes something (`payment_request_nothing_due`). The user confirms the client contact's email, any CC and the message. It emails the amount due, due date and bank details, with the request PDF and the invoice PDF attached; the CC is copied in and replies go to it. Check it with `payment_requests_retrieve` (Queued, Sent or Failed). |
| 18 | Record the payment | `payments_create` [key] (direction `in`, the client, amount, currency, `allocations` `[{invoice, amount}]`) | The money has arrived. Each allocation is in the invoice's currency and no more than it owes (`over_allocation`). The rest stays on account. |
| 19 | Move to Production | `projects_move_create` [key] `to` `production` | The deposit is paid (`deposit_unpaid`). Every SKU has a formula version or client oil (`formula_missing`). With client-supplied oil, its MSDS, SDS and IFRA certificate are Done. |
| 20 | Move to Balance due | `projects_move_create` [key] `to` `balance_due` | The project is at Production (filling usually runs through work orders first: `find_operations` on work-orders). This drafts the **balance invoice**, which is what the deposit left of the quote (none when the deposit was 100 %). |
| 21 | Issue, chase and collect the balance | `invoices_issue_create` [key], `invoices_payment_requests_create` [key], `payments_create` [key] | The same as steps 16 to 18. |
| 22 | Ship | `projects_move_create` [key] `to` `shipped` | The balance is paid (`balance_unpaid`) and every SKU's compliance documents are Done or Not needed (`compliance_incomplete`; `compliance_items_list` with `project`). This releases the project's reserved stock. |

## The client agreement

A contract is optional and sits outside the deal and project stages: none of them waits for it, and
signing one moves nothing.

1. `contracts_create` from a template (`contract_templates_list`; the seeded
   `development_agreement` is the development and pre-order agreement) for the client, linked
   with `source_type` `crm.deal` to the deal, or `projects.project` to the project once the deal
   is won. Set `fee`, `currency` and `package_lines` for the development fee.
2. Go through the filled `body` with the user and correct it with `contracts_partial_update`.
   `contracts_preview_create` renders the PDF to show them.
3. `contracts_send_create` [key] (`sent_to`, optional `cc` and `message`) emails the client the
   PDF and numbers the contract AGR-. The user confirms the recipient, CC and message first.
4. When the signed copy is back, `upload_file` it (doc_type `contract_signed`), then
   `contracts_signed_copy_create`.
5. `contracts_fee_invoice_create` [key] drafts the fee invoice. It needs `manage_money`, a fee
   and a currency (422 `contract_fee_missing`), and is made once while one is live (409
   `contract_invoice_exists`). Issue, chase and collect it as in steps 16 to 18.

The fee invoice is a Standard invoice linked to the deal or project. It is not the project's
deposit: the move to Production still needs the deposit invoice paid, and the `money_due` widget
lists deposit and balance invoices only. Once issued, one linked to the project counts in the
project's revenue (`project_profit`).

## Finding where a client stands

1. `get_record` on the deal shows its stage and outcome, and its `project` or `licence` once won.
2. `projects_retrieve` shows:
   - the stage;
   - every gate, met or not, with its code;
   - `money` (quotes, invoices, `accepted_quote_id`);
   - `open_generated_tasks`.
3. `invoices_list` with `source_type` `projects.project` and `source_id` shows the deposit and
   balance with their status, payment status and outstanding amount.
4. `contracts_list` with `client` (or `source_type` and `source_id`) shows the client's
   contracts with their status, `sends` and `fee_invoice`.
5. `my_work` and the `money_due` widget show what is due across all projects.

## When something is missing or wrong

- **A stage invoice was cancelled and is needed again**: use
  `projects_stage_invoices_create` [key] with kind `deposit` or `balance`. It returns
  `stage_invoice_exists` while a live one is there.
- **The deposit must change after the balance exists**: cancel the balance invoice first
  (`balance_invoice_exists`).
- **An invoice must be cancelled but has payments**: void the payment first with
  `payments_void_create` [key], which needs a `reason` (`invoice_has_allocations`).
- **The client changed their mind on price**: take a new snapshot, draft and send a new quote,
  then accept it. Accepting it supersedes the old quote, unless an invoice from the old quote was
  already issued.
- **The project is paused or dropped**: use `projects_move_create` [key] with `on_hold` or
  `cancelled` and a `reason`. To resume, set `to` to the project's `prior_stage`.
- **The contract's wording must change after it was sent**: edit it with
  `contracts_partial_update` and send it again; a Signed or Cancelled one no longer changes (409
  `contract_locked`).

## Other tracks

- **Blend Engine or Alchemy Engine deal**: the first won stage is Contract signed. Send the
  agreement from the deal as in "The client agreement", and once it is signed move the deal there
  with `deals_move_create` [key]: that is the win, and it drafts a licence instead of a project.
  Then complete the terms with `licences_update`, call `licences_activate_create`, then
  `licences_draft_invoice_create`, and issue the invoice (see the money skill).
- **Upsell**: quote the deal itself with `quotes_create` (`source_type` `crm.deal`). After the
  client accepts it, invoice it with `quotes_invoice_create` (kind `standard`, or a `deposit` then
  a `balance`).
