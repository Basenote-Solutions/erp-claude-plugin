---
name: lead-to-invoice
description: Covers the BaseNote ERP private-label workflow end to end. It runs from a lead through convert, the deal won, the PL project, the client agreement (contract) and its fee invoice, the quote sent and accepted, samples, the deposit invoice, the emailed payment request, the payment and the balance invoice, up to shipping, with the Blend Engine, Alchemy Engine and Upsell variants. It names each step's operation and what must be true first. Use it when the user wants to take a client from enquiry to invoice, asks what the next step is, where the contract or agreement fits, or why a project or invoice is stuck.
---

# From a lead to the balance invoice

Use the CRM for the deal, the project for partner and delivery, and Money for the client’s invoices and payments. Each can be updated when the relevant event takes place.

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
- Explain missing information for the specific transaction and offer the relevant record. Project checklists are optional.

## From enquiry to billing

Convert the lead on its sales track when the user wants a deal. Winning a private-label deal provides its linked project. Open it to record the production partner and delivery or change the board stage.

If the user wants to quote, create the quote directly or use a project costing snapshot. Record sending and acceptance when they happen. To invoice an accepted quote, check its existing invoices, then choose full, deposit or balance as appropriate. A standard project invoice can also be created directly. Issue it with `invoices_issue_create` and record actual payments against it. Use the current operation descriptions for fields and permission requirements.

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
   `contract_invoice_exists`). Issue the draft and record any payments received.

The fee invoice is a standard invoice linked to the deal or project. Once issued, a project-linked invoice contributes to project revenue.

## Finding where a client stands

1. `get_record` on the deal shows its stage and outcome, and its `project` or `licence` once won.
2. `projects_retrieve` shows:
   - the stage;
   - the production partner, dated updates and delivery dates;
   - `money` (quotes, invoices, `accepted_quote_id`);
   - the project’s task card, when available.
3. `invoices_list` with `source_type` `projects.project` and `source_id` shows the deposit and
   balance with their status, payment status and outstanding amount.
4. `contracts_list` with `client` (or `source_type` and `source_id`) shows the client's
   contracts with their status, `sends` and `fee_invoice`.
5. `dashboards_projects_retrieve` with `project` shows the project's money at a glance:
   expected revenue, what was invoiced, what the client paid and owes, what it has cost, its
   margin, its next step and what it is `missing`.
6. `my_work` and the `money_due` widget show what is due across all projects.

## Corrections

Open the project, quote, invoice or payment that needs attention. Show the existing record and the change the ERP supports. Use the administrative correction on the original record when the signed-in administrator is correcting a posting.

## Other tracks

- **Blend Engine or Alchemy Engine deal**: the first won stage is Contract signed. Send the
  agreement from the deal as in "The client agreement", and once it is signed move the deal there
  with `deals_move_create` [key]: that is the win, and it drafts a licence instead of a project.
  Then complete the terms with `licences_update`, call `licences_activate_create`, then
  `licences_draft_invoice_create`, and issue the invoice (see the money skill).
- **Upsell**: quote the deal itself with `quotes_create` (`source_type` `crm.deal`). After the
  client accepts it, invoice it with `quotes_invoice_create` (kind `standard`, or a `deposit` then
  a `balance`).
