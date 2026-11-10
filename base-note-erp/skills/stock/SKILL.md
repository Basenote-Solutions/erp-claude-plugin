---
name: stock
description: Items and stock in Base Note ERP - items and materials with their identifiers (CAS, EC, INCI), pack-size unit conversions, documents and ids in other systems, units of measure, material types, product classes and inventory categories, locations, lots, stock on hand and moves, receiving, adjusting, transferring and issuing stock, reservations, low stock and reorder levels, and importing opening stock. Use when the user asks what is in stock or where, a lot or expiry, a stock count or correction, moving stock between the lab and a factory, reserving stock for a project, items below reorder level, loading opening stock, or adding or changing an item, its CAS number, pack size, unit, SDS or category.
---

# Stock: items, locations, lots, postings and reservations

Stock is what Base Note (or a client, for client-supplied oil) holds: at the lab, at each
contract factory ("At factory: <name>"), in transit or at a client. Every change is a stock move
that never changes; balances and the average cost follow from the moves. Base Note's own stock
carries value at average cost; client-owned stock carries none.

## Records

- **Item** (code, name, class, inventory category, stock unit, stocked, lot-tracked, reorder
  level): tabs per category: Raw chemical, Accord, Finished oil, Bottle, Cap, Packaging, Sample,
  Finished goods, Other. Class `raw_material`, `packaging`, `intermediate`, `bulk`,
  `finished_good`, `kit`, `service` or `utility` (a service or utility is never stocked). Status
  active or archived; a material (raw material, intermediate, bulk, utility) also has a material
  type and Material status Draft, Approved, Needs updated documentation, Blocked (see
  lab-and-alchemy).
- **Location** (code, name, kind lab, factory, in_transit, client or other; a party for a factory
  or client). Seeded LAB and TRANSIT.
- **Lot** (item, lot number, expiry, owner: blank = Base Note), made when a lot-tracked item is
  first received under that number.
- **Stock on hand**: per item, location, lot and owner: On hand, Reserved, Available; Average cost
  and Stock value only with `view_cost` and never on client-owned stock.
- **Move** (kind receipt, issue, adjustment, transfer, sale), append-only.
- **Reservation** (State Active, Consumed, Cancelled) for a project or a work order.

## Operations

Call `describe_operation` before the first use of each operation for its exact body.

Items
- `items_list` (filter `category`, `item_class`, `material_status`, `status`, `search` by code,
  name or identifier, `code` exact), `items_retrieve` (the item with its `material` profile,
  `identifiers`, `uom_conversions`, `documents` and `external_refs`; its stock is `stock_list` or
  `stock_items_list` with `item`).
- `items_create` / `items_update` (PUT: every field, expected_revision): code, name, class (fixed
  once created), category, stock unit, lot-tracked, reorder level (in the stock unit); a material
  adds its `material` profile (`material_type`, chemical, trade and INCI names, density).
- `items_archive_create` / `items_restore_create` (expected_revision; a restored material returns
  to Draft).
- Every write to an item or its parts below sends the item's `expected_revision` and answers the
  whole item with its new `revision`: use that one for the next write.
- Identifiers: `items_identifiers_create` (`scheme` `cas`, `ec`, `inci`, `synonym` or
  `supplier_code`, `value`; CAS and EC numbers are checked), `item_identifiers_destroy` (path pk =
  the identifier's id, `query.expected_revision`). To change one, delete it and add the new one.
- Pack sizes: `items_uom_conversions_create` (`uom`, `factor` = stock units in one `uom`; one of
  the two units is a count unit, such as `pcs` at `"25"` for a 25 kg pack of an item stocked in
  kg), `item_uom_conversions_destroy` (path pk = the conversion's id, `query.expected_revision`;
  409 `conflict` while a supplier item or price uses that unit).
- Documents: `items_documents_create` (`attachment_id`: a finalized upload of doc_type `sds`,
  `coa`, `allergen_declaration`, `ifra_info`, `vegan_origin_declaration`, `tds`,
  `composition_declaration` or `regulatory_evidence`), `items_documents_destroy` (path `pk` and
  `attachment_id`, `query.expected_revision`): removes the link, the file stays.
- Ids in other systems: `items_external_refs_create` (`system`, `external_id`; 409 `conflict`
  when that id is already mapped in that system), `items_external_refs_verify_create` (path `pk`
  and `ref_id`) marks it reviewed by the user. No operation edits or deletes a ref.

Reference data
- Units: `uoms_list` (seeded `kg`, `g`, `L`, `mL`, `pcs`), `uoms_create` (`code`, `name`,
  `dimension` `mass`, `volume` or `count`, `scale` = decimal places 0 to 6, `factor` = kg, L or
  pcs in one unit), `uoms_archive_create`. No operation edits a unit.
- Inventory categories (the tabs): `inventory_categories_list`, `inventory_categories_create`
  (`code`, `name`, `position`), `inventory_categories_update` (PUT `name` and `position`; the code
  stays), `inventory_categories_archive_create`.
- Material types (a material profile's `material_type`): `material_types_list`,
  `material_types_create` (`code`, `name`), `material_types_archive_create`.
- Product classes (a formula version's `product_class`): `product_classes_list`,
  `product_classes_create` (`code`, `name`, `ifra_category`), `product_classes_archive_create`.
- Category, type and class codes are lower-case: a letter, then letters, digits or `_`. An
  archived row stays on the records using it and is refused for new use; no operation restores
  one.

Locations and lots
- `locations_list` (`kind`, `q`, `archived=true` to include archived), `locations_retrieve`,
  `locations_create` (code, name, kind, `party`: a supplier for a factory, a client for a client
  location, none for lab or in transit; one active location per party and kind),
  `locations_partial_update` (PATCH `name`, `party`, expected_revision), `locations_archive_create`
  (409 `location_in_use` while it holds stock or active reservations). Code and kind never change.
- `lots_list` (`item`, `owner`, `in_stock`, `q`), `lots_retrieve` (its balance per location),
  `lots_partial_update` (PATCH `expiry`, `notes`, expected_revision).

Stock and postings
- `stock_list` (`item`, `location`, `lot`, `owner`, `ownership` base_note|client, `category`, `q`,
  `zero=true` for rows at 0): Stock on hand rows. `stock_items_list` (`item`, `category`, `q`,
  `low=true`, `in_stock=true`): per-item totals with the reorder level and the low flag.
- `moves_list` (`item`, `location`, `lot`, `kind`, `group`, `source_type`/`source_id`, `from`/`to`
  dates): every move, newest first; `group` shows all lines of one posting.
- `stock_receive_create` [key]: stock without a supplier document (found, gifts). Lot-tracked
  items need `lot_no` (with its `expiry` when it has one); Base Note stock needs a `unit_cost`
  (GBP per stock unit, `view_cost`); client stock names `owner` and takes no cost. Purchases
  arrive with a goods receipt instead (sourcing).
- `stock_adjust_create` [key]: signed quantity (`"-5"` takes away) and a `reason`; a gain enters at
  the average cost, or a typed `unit_cost` only while the item has none. A lot-tracked line names
  its `lot`, or on a gain a new `lot_no`.
- `stock_transfer_create` [key]: `from_location` → `to_location`; no value moves.
- `stock_issue_create` [key]: out at average cost, for a project or work order (`source_type`,
  `source_id`: its reservations are used) or with a `reason`.
- Each posting takes `lines`, an optional `effective_date` (default today) and `reason`, and
  answers its move group (`id`, `moves`).

Reservations and low stock
- `reservations_list` (`state`, `item`, `location`, `lot`, `source_type`, `source_id`),
  `reservations_create` [key] (`source_type` `projects.project` or `manufacturing.work_order`,
  `source_id`, optional `note`, lines `[{item, location, lot, qty}]`),
  `reservations_release_create` [key] (a `reason`; the quantity becomes available; 409
  `reservation_not_active` once consumed or cancelled). A formula batch is reserved with
  `scale_ups_reserve_create` [key] (lab-and-alchemy).
- Low stock: `stock_items_list` with `low=true`, or `run_report` `low_stock`; stock value:
  `run_report` `stock_value` (`view_cost`). Low = Base Note on hand less reservations below the
  item's reorder level; lab and admin users are notified when an item becomes low.

Opening stock import
- `import_types_list` shows the `opening_stock` template: columns `item` (code), `location` (code),
  `qty` (stock unit), `unit_cost` (GBP; blank for client-owned), `owner` (client's name; blank =
  Base Note), `lot_no`, `expiry`. It posts one receipt. It needs `view_inventory`,
  `manage_inventory` and `view_cost` (the file holds costs).
- Steps: `upload_file` the CSV (`text`, content_type `text/csv`, doc_type `import`) →
  `imports_create` [key] (`attachment_id`, `import_type` `opening_stock`) → `imports_retrieve`
  until ready → `imports_check_create` (writes nothing; show the issues) → `imports_apply_create`
  [key]. A CSV with the exact template headers is read directly; any other document goes to the
  AI reader. Opening stock lands only where none is: a lot number the item already has, or a
  non-lot item its owner already holds at that location, is refused, so a file is never posted
  twice.

## Rules that make writes succeed

- Reads need `view_inventory`; postings, reservations and lot edits `manage_inventory`. Items and
  material types: `view_catalog` / `manage_catalog`. Locations, inventory categories, units and
  product classes are settings: `manage_settings`. Verifying an external ref: `approve_material`.
- Every posting takes an Idempotency-Key: pass `idempotency_key` (a new UUID you make per
  intent); if the connection drops, retry with the same key and body and it is never made twice.
- Quantities are JSON strings in the item's stock unit, at its scale (kg 6 dp, g 3, L 6, mL 3,
  pcs 0); finer → 422 `invalid_quantity`. Dates `YYYY-MM-DD`, never after today (422
  `effective_date_in_future`).
- Outbound quantity ≤ on hand − other sources' active reservations. Any shortage refuses the whole
  posting with 409 `insufficient_stock`; its details list each line's requested and available
  quantity, and nothing is written. Show them; never split the posting to force it through.
- An adjustment, a release and an issue without a source need a `reason` (422 `reason_required`).
  A receipt of Base Note stock without a `unit_cost`, or a gain on an item with no average cost
  and no typed cost → 422 `cost_missing`.
- An item's class never changes. Its stock unit is locked once it has stock moves (409
  `uom_locked`), and so are lot-tracked and stocked (409 `conflict`); before then, remove its
  pack-size conversions to change the unit.
- Costs are hidden without `view_cost`: **absent, never zero**.

## Common jobs

1. **Where is it?** `stock_list` with `item` (or `q`); add `location` to narrow.
2. **Correct a count:** read the row, compute nothing yourself: ask the counted quantity, then
   `stock_adjust_create` [key] with the signed difference and the user's reason.
3. **Move to a factory:** for a work order use `work_orders_send_materials_create` [key]
   (manufacturing); otherwise `stock_transfer_create` [key].
4. **Hold stock for a project:** `reservations_create` [key], or a formula batch via scale-up
   reserve.
5. **Add an item's CAS number, pack size or SDS:** `items_retrieve` for its `revision`, then
   `items_identifiers_create`, `items_uom_conversions_create`, or `upload_file` (doc_type `sds`) →
   `items_documents_create`; each answer carries the next `revision`.

## Pitfalls

- Moves never change: a mistake is corrected by a new posting (adjustment or transfer).
- A reservation without a lot holds Base Note stock; a client's oil is held only by naming its
  lot. Client stock has no value.
- Shipped or cancelled projects and completed or cancelled work orders cannot hold reservations.
- `items_retrieve` shows no stock figures: read `stock_list` or `stock_items_list`.
- An archived item can be restored; an archived unit, category, material type, product class or
  location cannot be through the connector.

## Confirm with the user before

Every receive, adjust, transfer, issue, reservation or release (item, lot, location, signed
quantity, reason), applying an import, deleting an identifier, a pack-size conversion or a
document link, archiving anything, and adding or verifying an external ref (it is never edited
or deleted).
