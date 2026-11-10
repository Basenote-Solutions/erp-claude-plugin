---
name: sourcing
description: Buying in Base Note ERP - suppliers and supplier items, a supplier's ids in other systems (external refs), supplier availability, supplier price versions with MOQ tiers (draft, publish, cancel, restore), RFQs, supplier quotes, compare and award, purchase orders (issue, close, cancel, PO PDF), goods receipts, shipments and bills from POs. Use when the user mentions a supplier, vendor, RFQ or request for quote, a quote comparison, MOQ or price tiers, a price list or price version, a PO or purchase order, a delivery, goods receipt or GRN, or shipment tracking.
---

# Sourcing: suppliers, prices, RFQs, orders and receipts

Sourcing buys raw chemicals, accords, bottles, caps, packaging, printing and freight for contract
factories to fill. It asks suppliers for prices at quantity tiers,
records their quotes, awards one, orders, receives the goods into stock and tracks consignments.
Bills and payments out belong to money (see the money skill); the full chain is buy-to-stock.

## Records and statuses (UI labels)

| Record | Number | Statuses |
|---|---|---|
| Supplier (a party with the supplier role) | - | supplier types: factory, bottle, cap, packaging, chemical, printing, freight, other |
| Supplier item (supplier code, unit, MOQ, pack sizes, lead time, Incoterm, approval, preferred) | - | approval: not required, pending, approved, rejected |
| External ref (the supplier's id in another system) | - | unverified until reviewed |
| Supplier price version | - | Draft → Published; Cancelled |
| Request for quote (RFQ) | RFQ- | Draft, Sent, Closed |
| Supplier quote | - | Received, Awarded (expired once Valid until has passed) |
| Purchase order | PO- | Draft → Issued → Partially received → Received → Closed; Cancelled |
| Goods receipt | GRN- | posted in one step |
| Shipment | - | Booked, In transit, Delivered |

An RFQ is numbered when created, a purchase order when issued (a draft has no number), a receipt
when posted.

## Operations

Call `describe_operation` before the first use of each operation for its exact body.

Suppliers and supplier items
- `parties_list` with `role=supplier` (and `supplier_type`, `search`): the Suppliers list.
  `parties_create` with `roles: ["supplier"]` and `supplier_types`: New supplier;
  `parties_contacts_create` adds its contact.
- `suppliers_items_list` / `suppliers_items_create` (path pk = the supplier): what it sells of each
  item (`item`, `supplier_code`, `uom`, `moq`, `pack_sizes`, `lead_time_days`, `incoterm`,
  `approval_status` default `pending`, `preferred`: one per item, 409 `conflict` on a second).
  `supplier_items_retrieve` (with its linked documents) / `supplier_items_update` (PUT: every
  field, expected_revision). `supplier_items_documents_create` (expected_revision, `attachment`)
  links a finalized upload (doc_type `sds`, `coa`, `supplier_document` and the other material
  kinds: certificates, SDS).
- `supplier_availability_list` (`item`, required: that item's supplier items) /
  `supplier_availability_update` (path = the supplier item; `status`): what the supplier reported
  (`available`, `low_stock`, `out_of_stock`, `discontinued`, `unknown`). Never calculated from
  stock.
- `suppliers_external_refs_list` (path pk = the supplier): its ids in other systems (`system`,
  `external_id`, `verified_at` null until reviewed). `suppliers_external_refs_create` (`system`
  such as `alchemy`, `external_id`) maps an active supplier; 409 `conflict` when that id is
  already mapped in that system. `suppliers_external_refs_verify_create` (path `pk` and `ref_id`,
  no body) marks it reviewed by the user. No operation edits or deletes a ref.

Price versions (MOQ tiers)
- `supplier_price_versions_list` (`supplier`, `state` `draft`, `published` or `cancelled`), newest
  first; `supplier_price_versions_retrieve`: one version with its prices and its `revision`.
- `supplier_price_versions_create` (`supplier`, `effective_at`: a timestamp) makes a Draft;
  `supplier_price_versions_update` (PUT: expected_revision, `effective_at`) moves a draft's start.
- `supplier_price_versions_prices_create` adds a price: `supplier_item` (one of that supplier's),
  `currency`, `tier_qty` (the From quantity; 0 = every quantity), `uom`, optional `pack_size`,
  `supplier_unit_price`, `valid_from`, `valid_to` (null: open-ended). `supplier_prices_update`
  (PUT) / `supplier_prices_destroy` (`query.expected_revision`) change a draft's price. Each
  takes the version's `expected_revision`, which every change moves on: use the `revision` of
  the last answer.
- `supplier_price_versions_publish_create` [key] (expected_revision): its prices apply and never
  change again. It needs at least one price; 409 `price_overlap` when two of its prices for the
  same supplier item, currency, unit and tier apply on one day.
- `supplier_price_versions_cancel_create` (expected_revision) withdraws a draft, or a published
  version with a `reason` (422 `reason_required`): it stops applying.
  `supplier_price_versions_restore_create` (optional `effective_at`, default now) clones a
  published or cancelled version into a new draft.
- 409 `supplier_price_version_state`: the version's state does not allow the action (editing or
  publishing one that is no longer a draft, cancelling a cancelled one).
- `supplier_prices_list` (filter `item`, `supplier`, `state` default `published`): prices grouped
  by item.
- A tier applies from its quantity up: the price for a quantity is the highest tier at or below
  it in the newest published version on the date.
- Bulk: the `supplier_prices` import makes one new draft version per supplier in the file (publish
  it afterwards); the `suppliers` import adds suppliers and supplier items. The import steps are
  in the erp-basics skill.

RFQs and supplier quotes
- `rfqs_list` (`status`, `project`, `supplier`, `search`), `rfqs_retrieve`, `rfqs_create` (title,
  optional project, `due_date` = Reply by, `suppliers`, lines: an item (asked in its stock unit) or
  a `spec` with its `uom`, and 1 to 10 `tiers` such as `["500","1000","2000"]`), `rfqs_update`
  (PUT, expected_revision; a sent RFQ keeps its lines and project: 409 `sourcing_document_state`).
- `rfqs_send_create` (expected_revision): marks it Sent once it has a line and an invited
  supplier. **Nothing is emailed**: the user sends the request themselves. `rfqs_close_create`
  (expected_revision) closes a draft or sent RFQ: no more quotes against it.
- `supplier_quotes_create` (`supplier`, optional `rfq`: it must be Sent and the supplier invited,
  `supplier_reference`, `quoted_on`, `valid_until`, `currency`, `fx_rate`, the supplier's file as
  `attachment`, lines: the `rfq_line` answered or an item or spec, `tier_qty`,
  `supplier_unit_price`, `moq`, `lead_time_days`, `supplier_code`). `supplier_quotes_update` (PUT,
  expected_revision) replaces a Received quote's header and lines, never its supplier or RFQ.
  `supplier_quotes_list` (filter `rfq`, `supplier`, `status`, `item`), `supplier_quotes_retrieve`
  (lines, files, `expired` and, once awarded, its `price_version` and `purchase_orders`).
- `rfqs_compare_retrieve`: the RFQ's quotes side by side in GBP per line and quantity, MOQ flagged
  (`below_moq`), lowest unit price marked Best (`best`); quote lines answering no RFQ line come as
  `unmatched`.
- `supplier_quotes_award_create` [key] (expected_revision): Received → Awarded. It publishes the
  quote's item prices as the supplier's price version (spec lines are not published), making a
  supplier item where the supplier has none for the item; with `draft_po: true`, `po_lines`
  (`[{quote_line, qty}]`), `deliver_to` and `expected_date` it also drafts the purchase order (the
  answer's `purchase_order`).

Purchase orders
- `purchase_orders_list` (`open=true`: issued or partly received; `status`, `supplier`,
  `project`, `search` by number), `purchase_orders_retrieve` (lines with `received_qty` and
  `remaining_qty`; `has_pdf`).
- `purchase_orders_create` / `purchase_orders_update` (PUT, Draft only, expected_revision):
  supplier, project, order date, expected by, `deliver_to` location, currency, rate, notes, lines:
  a stocked item (received into stock) or a description (a service such as filling: billed, never
  received), qty, unit price, tax rate (blank: the default rate).
- `purchase_orders_issue_create` [key] (expected_revision): numbers it PO- and has the worker
  render its PDF.
- `purchase_orders_pdf_retrieve`, or `download_file` with `purchase_order_id`: a 5-minute link;
  404 `purchase_order_pdf_not_ready` until the worker has made it (retry shortly).
- `purchase_orders_close_create` [key]: ends an issued, partly received or received order (a
  `reason` unless fully received: 422 `reason_required`). `purchase_orders_cancel_create` [key]: a
  draft, or an issued order nothing was received on (a reason unless draft; its number is kept).
  Both take expected_revision.
- `purchase_orders_bill_create` [key] (`bill_date`, `supplier_ref`, `due_date`): a draft bill of
  what the lines have not been billed yet, at the order's prices and tax rates, for an issued,
  received or closed order; it answers the bill's id. 409 `nothing_to_bill` when all is billed.
  Posting and paying it: `bills_post_create` [key], then the money skill.

Goods receipts and shipments
- `goods_receipts_create` [key]: `purchase_order` (Issued or Partially received), `location`,
  `received_on` (default today, never later), lines `[{po_line, qty, lot_no, expiry}]`. Stock
  comes in at once at the order's price × its rate per stock unit, numbered GRN-. Without an
  order: `from_party` and each line's `item` and `unit_cost` (GBP), or `owner` (a client's
  stock, no cost). `goods_receipts_list` (filter `purchase_order`, `location`, `item`,
  `from_party`), `goods_receipts_retrieve`.
- `shipments_create` (carrier, tracking number, ETD, ETA not before ETD, status, and the
  `purchase_order`, `project` or `work_order` it carries); `shipments_update` (PUT) sends the
  whole record with expected_revision: Mark in transit / Mark delivered = `status` `in_transit` /
  `delivered`. `shipments_list` (`open=true`: not delivered; `status`, `purchase_order`,
  `project`, `work_order`), `shipments_retrieve`.

## Rules that make writes succeed

- Permissions: RFQs, quotes, orders, receipts and shipments read with `view_sourcing` and write
  with `manage_sourcing`. Supplier items, price versions, external refs and availability are
  catalog records: `view_catalog` to read, `manage_catalog` to write, `publish_supplier_prices`
  to publish or cancel a version, `approve_material` to verify an external ref.
- The compare, the PO PDF and `supplier_prices_list` also need `view_supplier_price`; the bill
  from a PO needs `view_sourcing`, `manage_money` and `view_supplier_price` (no
  `manage_sourcing`).
- Without `view_supplier_price`, prices, totals and quote figures are **absent, never zero**; a
  receipt's unit costs also need `view_cost`. Say the role does not show them. A quote or PO edit
  resends every line with its price, so it cannot be built from a read that hides them: stop and
  say who must do it.
- [key] operations take an Idempotency-Key: pass `idempotency_key` (a new UUID you make per
  intent); after an unknown outcome retry with the same key and body. Edits and commands send the
  record's current `expected_revision`.
- Decimals and quantities are JSON strings (`"1000"`, `"2.45"`). Receipt quantities are in the
  item's stock unit; RFQ and quote lines are in the line's unit (the compare does not convert).
- A quote against an RFQ needs the RFQ Sent and its supplier invited; a line answering an RFQ
  line is in that line's unit.
- A receipt line never exceeds what its PO line still expects (422 `over_receipt`); lot-tracked
  items need a `lot_no` (and expiry if it has one), other items take neither.
- Award refuses a PO line whose quantity is below the quote's lowest tier (422).
- 409 `sourcing_document_state`: the document's status does not allow the action (a quote
  against an RFQ that is not Sent, an awarded quote's or issued order's edit, a receipt against
  an order that is not open).
- `fx_rate` null takes the latest rate on or before the document date; `fx_rate_missing` means
  none exists: ask the user to enter one (Settings, Currencies). GBP is always 1.

## Common jobs

1. **Price for a quantity:** `supplier_prices_list` with the item; pick the newest published tier at
   or below the quantity.
2. **Record a quote:** `upload_file` the supplier's PDF (doc_type `supplier_quote`), then
   `supplier_quotes_create` with `attachment` and the lines read from it; check them with the user.
3. **Choose and order:** `rfqs_compare_retrieve`, show the Best per line, then on the user's choice
   `supplier_quotes_award_create` [key] with `draft_po`, then `purchase_orders_issue_create` [key].
4. **Receive a delivery:** `purchase_orders_retrieve` for the lines still expected and
   `deliver_to`; confirm quantities, lot numbers and expiry; `goods_receipts_create` [key]; report
   GRN-.
5. **Open orders:** `purchase_orders_list` with `open=true`; overdue = expected date before today.
6. **Enter a price list by hand:** `supplier_price_versions_create`, then a
   `supplier_price_versions_prices_create` per supplier item and tier (each with the last answer's
   `revision`); check the draft with the user; `supplier_price_versions_publish_create` [key].
7. **Correct a published price:** `supplier_price_versions_restore_create` from it, change the
   draft's prices and publish; cancel the wrong version if it must stop applying.

## Pitfalls

- A wrong receipt is never edited: correct it with a stock adjustment (stock skill), with a reason.
- Published prices never change: publish a new version, or cancel the wrong one.
- A sent RFQ keeps its lines and project; a supplier who has quoted stays invited.
- A supplier item starts with approval `pending` (an award makes missing ones so). A formula's
  costing (formula costs, the project calculator) reads published prices only of supplier items
  whose approval is `approved` or `not_required`: set it with `supplier_items_update`.
- Supplier names and prices are confidential: do not put them in anything for outsiders unless the
  user asks.

## Confirm with the user before

Awarding (publishes prices, may draft a PO), issuing, closing or cancelling a PO (show supplier,
lines and totals), posting a goods receipt (it moves stock), publishing or cancelling a price
version, drafting a bill, sending or closing an RFQ (neither is reversed), and adding or verifying
an external ref (it is never edited or deleted).
