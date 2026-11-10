---
name: buy-to-stock
description: Step-by-step workflow in Base Note ERP from asking suppliers for prices to paying them - RFQ, supplier quotes, compare, award, purchase order, issue, shipment tracking, goods receipt into stock, bill from the PO, posting it and the payment out. Use when the user wants to buy materials, bottles, caps or packaging, get quotes from suppliers, order from the best one, receive a delivery or pay a supplier for an order.
---

# Workflow: from a request for quote to stock and the supplier paid

Sourcing and stock describe each record and operation; this is the order to run them in. Call
`describe_operation` before the first use of each operation. Decimals and quantities are JSON
strings; values the user may not see are absent, never zero.

**Permissions:** RFQs, quotes, orders and receipts need `manage_sourcing`; the compare and the PO
PDF also `view_supplier_price` (without it prices are absent, so quotes and PO lines cannot be
checked or edited); the bill from the PO needs `view_sourcing`, `manage_money` and
`view_supplier_price`; posting it and paying need `manage_money` and `view_supplier_price`. If one
is missing, stop at that step and say who must do it.

## 1. Ask for prices (RFQ)

- Check the items exist (`items_list`); write a spec line for anything not yet an item.
- `rfqs_create`: `title`, optional `project`, `due_date` (Reply by), `suppliers` (parties with
  the supplier role: `parties_list` with `role=supplier`), lines with an `item` (asked in its
  stock unit) or a `spec` and `uom`, and 1 to 10 `tiers` such as `["500","1000","2000"]`. It is
  numbered RFQ- at once.
- `rfqs_send_create` (expected_revision) marks it Sent; it needs a line and an invited supplier.
  **Nothing is emailed**: offer to draft the request text for the user to send each supplier.

## 2. Record the quotes

- For each reply: `upload_file` the supplier's document (doc_type `supplier_quote`), then
  `supplier_quotes_create` with `supplier`, `rfq`, `supplier_reference`, `quoted_on`,
  `valid_until`, `currency` (`fx_rate` null: the latest rate), `attachment` and a line per RFQ
  line and From quantity: `rfq_line`, `tier_qty`, `supplier_unit_price`, `moq`, `lead_time_days`,
  `supplier_code`. Read the figures back to the user before saving.
- The RFQ must be Sent (409 `sourcing_document_state` otherwise) and the supplier invited (422):
  invite a late supplier with `rfqs_update` first. A line answering an RFQ line is in its unit.

## 3. Compare and award

- `rfqs_compare_retrieve`: per line and quantity each quote's GBP unit price and total, MOQ
  (flagged above the quantity), lead time, the lowest marked Best. Summarise; let the user choose
  (price is not the only criterion: lead time, MOQ, validity).
- `supplier_quotes_award_create` [key] with expected_revision, `draft_po: true`, `po_lines`
  (`[{quote_line, qty}]`, each at the tier its quantity reaches; below the lowest tier is refused),
  `deliver_to` (a location, usually LAB or a factory's) and `expected_date`. The quote reads
  Awarded, its item prices are published as the supplier's prices (a spec line's are not), and a
  Draft PO (the answer's `purchase_order`) takes the quote's supplier, currency, rate and the
  RFQ's project.
- `rfqs_close_create` (expected_revision) once no more quotes are wanted.

## 4. Issue the purchase order

- `purchase_orders_retrieve` the draft; fix lines with `purchase_orders_update` if needed (a PUT
  of the whole order with expected_revision; a service such as filling is a description line:
  billed, never received).
- `purchase_orders_issue_create` [key] with expected_revision: numbered PO-; the worker renders
  the PDF. Fetch it with `download_file` (`purchase_order_id`) and give the user the link to send
  the supplier; 404 `purchase_order_pdf_not_ready` means try again in a moment.
- Optional tracking: `shipments_create` (carrier, tracking number, ETD, ETA, `purchase_order`);
  `shipments_update` (the whole record, expected_revision) to `in_transit` and `delivered`.

## 5. Receive the goods into stock

- `purchase_orders_retrieve`: the lines still expected (`remaining_qty`) and the delivery
  location.
- Ask what actually arrived: quantities (in the item's stock unit), lot numbers and expiry for
  lot-tracked items (other items take neither).
- `goods_receipts_create` [key]: `purchase_order`, `location`, `received_on` (never after today),
  lines `[{po_line, qty, lot_no, expiry}]`. Stock comes in at once at the PO price × rate (GRN-);
  the PO becomes Partially received or Received. A line beyond what the PO line still expects →
  422 `over_receipt`. Check it in `stock_list`.
- A wrong receipt is corrected by a stock adjustment, never edited.

## 6. Bill from the PO

- `purchase_orders_bill_create` [key]: `supplier_ref` (their invoice number), `bill_date`,
  `due_date` (default: the supplier's terms): a draft bill of what is not billed yet, at the
  order's prices and tax rates; the answer is the bill's id. 409 `nothing_to_bill` when all is
  billed, 409 `bill_duplicate` when the supplier already has a bill with that reference.
- Compare it with the supplier's invoice; adjust with `bills_update` (expected_revision) if the
  supplier billed differently. Attach their scan: `upload_file` (doc_type `supplier_bill`) →
  `bills_files_create`.
- `bills_post_create` [key] with expected_revision: Posted, totals frozen at the bill date's rate.

## 7. Pay the supplier

- `payments_create` [key]: `direction` `out`, `party` (the supplier), `payment_date`, `amount`,
  `currency` (the bill's), `method` (`bank_transfer`, `card`, `cash`, `other`), `reference`, and
  `allocations` `[{bill, amount}]` (each ≤ what the bill still owes: 422 `over_allocation`). An
  unallocated rest stays on account; `payments_allocate_create` [key] allocates it later.
- The bill's payment status becomes part paid or paid.
- Finally `purchase_orders_close_create` [key] with expected_revision: a Received order closes as
  it is; one that will not be fully received needs a `reason`.

## Pitfalls

- Use the same Idempotency-Key only to retry the same posting after an unknown outcome.
- `fx_rate_missing`: no rate for that currency on or before the date; ask the user to add one.
- Published prices never change; a wrong award's prices need a cancelled price version
  (`supplier_price_versions_cancel_create`, `publish_supplier_prices`).
- A sent RFQ keeps its lines and project, an awarded quote and an issued PO their lines.

## Confirm with the user before

Sending and closing the RFQ, awarding, issuing the PO (supplier, lines, total), posting the
receipt (items, lots, quantities, location), drafting and posting the bill, the payment (amount,
date, bill), and closing the PO.
