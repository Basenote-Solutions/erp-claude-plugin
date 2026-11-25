---
name: fill-to-ship
description: Track third-party production and delivery on a project. Record a purchase, bill, physical stock receipt or shipment when requested.
---

# Production partner and delivery

1. Find the project with `projects_list` and read it with `projects_retrieve`.
2. For a partner change, call `projects_partial_update` with the current revision. Record a typed partner name or choose an existing directory party. Record the partner’s reference and expected delivery when known. The client’s target date is a separate field.
3. To record news from the partner, use `find_operation` for project partner updates. Supply the date and the update. When delivery is confirmed, record its actual date.
4. To change the board, read the stage catalogue and move directly to the requested stage. The project can be reopened and its delivery date corrected later.

Use the supplier-order, bill, shipment or inventory operation only for the event the user wants to record. A service line on a supplier order covers production performed by a partner. A factory-to-client delivery can be tracked on the project. A stock receipt records goods that actually arrive at a company location.

Describe each operation before first use. Confirm the intended write with the user and reuse its idempotency key when retrying an unknown outcome.
