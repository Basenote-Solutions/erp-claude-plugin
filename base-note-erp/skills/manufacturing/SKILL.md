---
name: manufacturing
description: Work orders to contract factories in Base Note ERP - creating and issuing them, sending materials to the factory, recording what it used and returning leftovers, the reconciliation, receiving finished goods or shipping them direct, status updates and the order's files and work instructions. Use when the user mentions a work order or WO, a factory or filler, filling, sending materials, factory stock, what the factory used, reconciliation, finished goods or ship direct.
---

# Manufacturing: work orders at contract factories

Contract factories fill a PL project's SKUs. A work order (WO-)
orders some of a project's SKUs from one factory. Materials move by stock transfer into the
factory's location (named "At factory: <factory>"); the factory reports what it used (issued at
average cost, which becomes the project's material cost), leftovers come back, and the finished
units are received into a location or shipped straight to the client.

## Statuses (Progress)

Draft → Issued → Materials received → Filling → Packed → Ready → Completed; Cancelled.
API codes: `draft`, `issued`, `materials_received`, `filling`, `packed`, `ready`, `completed`,
`cancelled`. The page's next-step buttons are Issue to factory, Materials received, Filling,
Packed, Ready, Complete; Record a status update dates a step earlier or skips ahead.

## Operations

Call `describe_operation` before the first use of each operation for its exact body.

- `work_orders_list` (filter `status`, `project`, `factory`, `search`).
- `work_orders_retrieve`: the whole order: `lines` (per SKU: units ordered, overage, to fill,
  received, shipped), `events` (status updates), `materials` (every send, use and return),
  `reconciliation`, `factory_stock` (what the factory's location holds now), `dispatches`,
  `allowed` (what its status permits now: `move_to`, `back_to`, `cancel`, `edit`, `send`, `use`,
  `return`, `finish`), `documents`, `material_cost` (`view_cost`), `revision`. Read `allowed`
  before proposing a step.
- `work_orders_create`: `project` (open: not Shipped or Cancelled), `factory` (a supplier of type
  factory), `materials_by`, `filling_by`, `ready_by`, `notes`, `lines`
  `[{project_sku, units_ordered, overage_units}]` (whole units; blank = the SKU's own; each live
  SKU of the project once; the lines may also come later). Made as a Draft and numbered at once.
  SKU ids come from `projects_retrieve`.
- `work_orders_partial_update` (`expected_revision`): a draft's factory, dates, notes and lines
  (`lines` replaces them all); once issued only the dates and notes, until completed or cancelled
  (409 `work_order_status_invalid` otherwise).
- `work_orders_move_create` (`expected_revision`): `to` a status, `event_date` (default today,
  never later), `note`; `reason` for one step back or for `cancelled`.
- `work_orders_send_materials_create` [key]: `from_location`, lines `[{item, lot, owner, qty}]`:
  one transfer into the factory's location. Stock reserved for the project travels with it.
- `work_orders_record_use_create` [key]: lines `[{item, lot, owner, qty}]` from `factory_stock`
  (its `item_id`, `lot_id` and `owner_id`) with the quantity used: one issue at the factory's
  location at average cost (a client's own oil costs nothing).
- `work_orders_return_materials_create` [key]: `to_location`, lines: leftovers back.
- `work_orders_receive_goods_create` [key]: `location`, lines `[{line, units}]` (line = the work
  order line id): the SKU's finished-goods item comes in at zero cost (made on first receipt).
- `work_orders_ship_direct_create` [key]: lines `[{line, units}]`: units sent from the factory
  straight to the client, without stock. Track them with `shipments_create` (`work_order`,
  `carrier`, `tracking_number`; a sourcing record, so `manage_sourcing`) and find them with
  `shipments_list` (`work_order`).
- The five postings also take `expected_revision`, `effective_date` (default today, never later)
  and a `note`.
- `work_orders_documents_create` (`attachment_id`): Files such as the factory's confirmations or
  packing photos; upload first with `upload_file`, doc_type `work_order`.
- Work instructions for the factory live on the formula version: `recipe_revisions_retrieve`
  (its instructions and instruction documents; percentages need `view_formula_pct`).
- Context: `locations_list` (`kind=factory`), `stock_list` (`location`: the factory's),
  `moves_list` (`source_type` `manufacturing.work_order` with `source_id`), `reservations_list`.

## Rules that make writes succeed

- Reads need `view_manufacturing`, writes `manage_manufacturing`. Costs and the value used need
  `view_cost`: without it they are **absent, never zero**.
- Every edit, move and posting sends the order's current `expected_revision` (409
  `stale_revision`: read the order again); the postings ([key]) also take an Idempotency-Key (a
  new UUID you make per intent; reuse it to retry an unknown outcome).
- Quantities are JSON strings in the item's stock unit; finished-goods units and a line's units
  are whole numbers, sent as strings too (`"500"`). Lot-tracked items name their `lot`; a
  client's own oil without a lot names `owner` (the client).
- A step its status does not allow answers 409 `work_order_status_invalid`: sending works from
  Issued to Ready; record use and return materials from Issued on, also on a completed or
  cancelled order (late reports); receive goods and ship direct only at Ready.
- **Issue to factory** needs at least one line, no SKU archived since it was added (422 on
  `lines.N.project_sku`: remove the line), the factory still a factory, and the project in
  Production or Balance due (409 `project_not_in_production`: deposit unpaid or production gates
  unmet). The first order issued to a factory creates its location.
- Forward moves go to any later status up to Ready (a draft only to Issued; Completed only from
  Ready). One step back, never from Issued to Draft, or Cancel, from any status before
  Completed, needs a reason (422 `reason_required`); Cancel is refused once finished goods are
  recorded (409 `work_order_has_output`). Completed and Cancelled are final. Event dates never
  after today (422 `effective_date_in_future`).
- Send and record use: a shortage refuses the whole posting with 409 `insufficient_stock`, its
  `details.shortages` listing each line's requested and available quantity; nothing is posted.
  Record use can only take what the factory's location holds, not what this order sent. Send
  from, and return to, a location other than the factory's own.
- Receive goods and Ship direct first run the project's release check: 422 `balance_unpaid` (with
  what is outstanding), 422 `compliance_incomplete` (naming documents) or 409
  `project_stage_invalid` (the project was cancelled). So the balance invoice, drafted when the
  project moves to Balance due, must be paid first (at a 100 % deposit: the deposit, covering the
  quote). No SKU takes more than units ordered plus overage (422 `units_exceed_line`, with what
  each line has left), and an archived SKU's line takes none.
- **Complete** needs finished goods received or shipped on every live SKU (409
  `finished_goods_missing`). Completing or cancelling releases this order's reservations; the
  project's own reservations stay for its other orders.

## Reconciliation

Per item: `sent`, `used`, `returned`, still at the factory (`at_factory` = sent − used −
returned), what the factory's location holds now (`on_hand_at_factory`) and, with `view_cost`,
the `material_cost` used. It is a view: nothing is posted from it. A negative "still at
factory" means stock reached the factory another way (a PO delivered there, leftovers of an
earlier order). Read it from `work_orders_retrieve`; settle differences with record use or
return materials, and ask the user before any stock adjustment.

## Common jobs

1. **Start an order:** `projects_retrieve` (stage, SKUs) → `work_orders_create` → confirm →
   `work_orders_move_create` to `issued`.
2. **Send materials:** check `stock_list` at the source location, confirm items, lots and
   quantities → `work_orders_send_materials_create` [key].
3. **Follow the filling:** `work_orders_move_create` to `materials_received`, `filling`, `packed`,
   `ready` as the factory reports (with `event_date` for an earlier day).
4. **Close out:** record use from `factory_stock`, return leftovers, show the reconciliation, then
   receive goods or ship direct, then move to `completed`.

## Pitfalls

- `lines` in `work_orders_partial_update` replaces every line: send the ones to keep as well.
- Units are whole numbers but still JSON strings (`"500"`), like every quantity.
- Record use takes from the factory's stock, whichever order or purchase brought it there.
- `shipments_update` is a PUT: send `carrier`, `tracking_number`, the dates, the links and the
  `status` again with `expected_revision`, or what is left out is reset.
- The order's `revision` rises with every move and posting: take it from the answer (the whole
  order) for the next one.

## Confirm with the user before

Issuing, every posting (send, use, return, receive, ship direct: name items, lots, quantities and
locations), completing and cancelling. Postings are corrected only by new postings, never edited.
