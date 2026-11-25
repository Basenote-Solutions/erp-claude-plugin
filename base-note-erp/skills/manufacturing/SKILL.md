---
name: manufacturing
description: Find and settle an existing factory work order. Use the project for new third-party production.
---

# Previous factory orders

For a new order, use the project’s production partner, expected delivery and dated updates. The fill-to-ship skill covers this workflow.

For an existing work order, use `get_record` or `work_orders_list` to find its number. Review its project, dispatches, reservations, materials at the factory and available commands. Use reconciliation to see what the factory still holds.

Record the remaining real events through the existing order. Return unused materials to their actual location, record materials used, or record finished units received or shipped direct. A direct shipment records the dispatch. A physical receipt also posts the units into stock. Use the revision shown and one idempotency key per posting. Close the order once reconciled. Its files, history and contribution to project cost stay accessible.
