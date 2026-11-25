---
name: pl-projects
description: Manage private-label projects, partners, delivery, board stages, optional checklists, SKUs, sample rounds, pricing and project files.
---

# Private-label projects

Find a project with `projects_list` or `search`. `get_record` includes its accessible history and web link. A project can start with its name, client and the site's base currency, and its card can start empty. Winning a private-label deal also creates a project.

## Production partner and board

Use `projects_partial_update` for the producer, their reference, expected delivery and actual delivery. Accept a typed partner name or an existing party. Record dated news with the project partner-updates operation. Staff can record these facts in any order. Use the fill-to-ship skill for linked purchases, bills and shipments.

Read the project stage catalogue so the user sees the current labels. `projects_move_create` takes the selected code and revision. Project managers can move directly to another active stage, including reopening a project. Administrators can add, rename, reorder, archive and restore stages. The project’s stage history, invoices, payments, stock and delivery dates each record their own event.

## Tasks and templates

Open the project’s record card to add checklists and steps, remove a whole checklist, or edit one step. Use project-linked tasks for work with its own owner. Choose the project apply-template operation only when the user requests a starter checklist. Checklists work at any stage.

## Optional project records

- Use the SKU, sample-round, reference, artwork, packaging and project-paper operations to record the product specifications and work the team actually has.
- Calculate a price from the project’s inputs. A saved snapshot can be used to draft a quote. Read the project’s money section before creating an invoice to see which invoices already exist. Use Money to issue it and record actual payments.
- Use the project dashboard and profit report for expected revenue, invoicing, outstanding amounts, actual costs and margin. The ERP applies the reader’s field permissions.
- Attach documents through the connector upload flow, then link the accepted attachment to its record.

Use `find_operation` and `describe_operation` for the current request fields. Use the returned revision for edits and one idempotency key per intended posting. After a stale revision, reload the project and show the user the relevant change before applying their edit.
