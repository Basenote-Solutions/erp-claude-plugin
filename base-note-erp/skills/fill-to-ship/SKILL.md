---
name: fill-to-ship
description: Step-by-step workflow in Base Note ERP for a private-label project in production - check materials, create and issue the work order to the contract factory, send the materials, follow the filling, record use and returns, reconcile, collect the balance and finish the compliance documents, receive or ship the finished goods, complete the work order and move the project to Shipped. Use when the user wants to get a PL project filled at a factory and delivered to the client.
---

# Workflow: from a project in production to goods shipped to the client

Manufacturing, stock, compliance and the money skill describe each operation; this is the order
to run them in. Call `describe_operation` before the first use of each operation. Operations
marked [key] take an Idempotency-Key (a new UUID you make per intent); a move or posting on a
record also sends its current `expected_revision`. Stock shortages answer 409
`insufficient_stock` with each line's requested and available quantity, and nothing is posted.

## 1. Check the project

- `projects_retrieve`: the stage must be Production (deposit paid, every SKU has a formula
  version or client oil, client-oil MSDS, SDS and IFRA certificate done) or Balance due. Its gates
  say what is missing; moving a project there is the pl-projects skill.
- Note each SKU's id, units ordered and produced, formula version and fill.

## 2. Make sure the materials exist

- For each SKU's formula version: `scale_ups_create` by units (`external_version_id`, `basis`
  `units`, `units`, `fill_ml`, `concentration_pct`, `density`; `project` adds what the project
  already holds) shows per material what is needed, available and short (`need`, `available`,
  `short`). It writes nothing.
- Short: buy it (buy-to-stock). Bottles, caps and packaging are often ordered with `deliver_to`
  the factory's location, so they never pass through the lab.
- Hold what is available for the project: `scale_ups_reserve_create` [key] (the same body with
  `project` and one `location`), or `reservations_create` [key] for packaging (`source_type`
  `projects.project`, `source_id`, lines of `item`, `location`, `lot`, `qty`).

## 3. Create and issue the work order

- `work_orders_create`: `project`, `factory` (`parties_list` with `role=supplier` and
  `supplier_type=factory`), `materials_by`, `filling_by`, `ready_by`, lines
  `[{project_sku, units_ordered, overage_units}]` (blank: the SKU's own). Draft, WO- numbered.
- `work_orders_move_create` `to: "issued"`: refused with 409 `project_not_in_production` unless
  the project is at Production or Balance due. The first order to a factory creates its location
  "At factory: <name>".
- Attach the factory's confirmation: `upload_file` (doc_type `work_order`) →
  `work_orders_documents_create`.

## 4. Send the materials

- `work_orders_send_materials_create` [key]: `from_location` (e.g. LAB) and lines
  `[{item, lot, owner, qty}]` in each item's stock unit (lot for lot-tracked items, `owner` for a
  client's own oil). One transfer into the factory's location; reservations travel with it.
  Sending works from Issued to Ready.
- Track a consignment if wanted: `shipments_create` (`work_order`, `carrier`,
  `tracking_number`).

## 5. Follow the filling

- `work_orders_move_create` to `materials_received`, `filling`, `packed`, `ready` as the factory
  reports, with `event_date` for an earlier day (never after today) and a `note`.

## 6. Record use, return leftovers, reconcile

- `work_orders_retrieve` → `factory_stock` lists what the factory's location holds.
- `work_orders_record_use_create` [key]: what the factory used, per item and lot: issued at
  average cost, it becomes the project's material cost.
- `work_orders_return_materials_create` [key]: leftovers back to `to_location`.
- Show `reconciliation` per item: sent, used, returned, still at the factory, held there now. A
  negative "still at factory" means stock arrived another way (a PO delivered there). Nothing is
  posted from it; ask the user before any adjustment.

## 7. Release: balance and documents

- `projects_move_create` [key] `to: "balance_due"`: drafts the balance invoice (none at a 100 %
  deposit; `projects_stage_invoices_create` [key] redrafts a missing one).
- Issue it with `invoices_issue_create` [key], send it, and when the client pays record
  `payments_create` [key] (`direction` `in`, allocation to the invoice): see the money skill.
- Every live SKU's compliance documents Done or Not needed: `compliance_items_list` with
  `project` and `status=outstanding`; finish them with the compliance skill. A Done document past
  its expiry counts as not done.

## 8. Finished goods

- From Ready, either:
  - `work_orders_receive_goods_create` [key]: `location` and lines `[{line, units}]` (whole
    units): the SKU's finished-goods item comes in at zero cost; or
  - `work_orders_ship_direct_create` [key]: lines `[{line, units}]` sent from the factory
    straight to the client, without stock.
- Both run the release check first: 422 `balance_unpaid` (with what is outstanding) or 422
  `compliance_incomplete` (naming documents); so step 7 comes before them. No SKU takes more
  than ordered plus overage (422 `units_exceed_line`).
- Goods received into stock leave it by a stock posting when they go to the client; ask the user
  how they record it (a transfer to the client's location, or an issue for the project).
- Track the delivery: `shipments_create` (`project`, `work_order`, `carrier`,
  `tracking_number`, `etd`, `eta`); `shipments_update` to `status` `in_transit`, then
  `delivered` (a PUT: send the other fields again with `expected_revision`).

## 9. Close out

- `work_orders_move_create` `to: "completed"` (409 `finished_goods_missing` until every live SKU
  has goods received or shipped); it releases the order's own reservations.
- `projects_move_create` [key] `to: "shipped"`: the release check again; Shipped is final,
  releases the project's remaining reservations and tells the owner (unless the owner moved it).

## Pitfalls

- A work order with finished goods recorded cannot be cancelled (409 `work_order_has_output`).
- Record use takes from the factory's stock, not from what this order sent.
- Costs and the value used are absent without `view_cost`, never zero.
- Each move and posting raises the work order's `revision`: use the one in the answer next.

## Confirm with the user before

Issuing the work order, every send, use, return, receipt or direct shipment (items, lots,
quantities, locations), each project stage move, issuing the invoice, the payment, completing the
work order.
