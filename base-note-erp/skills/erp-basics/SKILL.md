---
name: erp-basics
description: How to work in Base Note ERP through its connector - which tool to use, how to find and call any operation, and the rules every read and write follows (permissions, decimals, revisions, idempotency keys, errors, confirmation). Also what no area skill covers - tasks on any record (the board, moving and blocking, checklists, comments, files, labels), notifications and their preferences, the import tool (import types, template CSVs, documents read by AI, check and apply), uploading files and their doc types, and settings (company details, users and roles, numbering series, Claude connections). Use it whenever the user asks anything about the ERP, its records, reports or tasks, before the area skills, and for a task board or a blocked task, a notification, importing a spreadsheet, PDF or photo, attaching a file, adding or deactivating a user, roles or company settings.
---

# Base Note ERP: the essentials

Base Note ERP covers CRM and quotes, private-label (PL)
projects, sourcing, contract manufacturing, the lab and the Alchemy formulation engine, stock,
compliance and light money (no ledger). Through the connector you act **as the signed-in user**:
exactly their role's permissions, every change recorded under their name with "via Claude" in
History.

## Which tool

| Need | Tool |
|---|---|
| Find a record by name, number or code | `search` (all types the user may read), then `get_record` |
| One record with its history, open tasks and web link | `get_record` (type = API collection such as `invoices`, or `money.invoice`) |
| Who the user is, and what needs attention | `my_work` (their user id, timezone and business date; open tasks, unread notifications, Home widgets) |
| Figures across records | `run_report` (no name: the catalogue and its parameters) |
| Anything else | `find_operations` → `describe_operation` → `read` (GET) or `write` |
| A PDF, attachment or export | `download_file`; to add one, `upload_file` then link it with `write` (a file you hold: give its `size`, run the answer's curl in your sandbox, then finish with `attachment_id`) |

Area skills name the operations for each job: crm-and-quotes, pl-projects, sourcing,
manufacturing, stock, lab-and-alchemy, compliance, money, reports; and the workflows
lead-to-invoice, brief-to-formula, buy-to-stock, fill-to-ship.

## Calling operations

1. `find_operations` with a few words ("issue invoice", "stock receive") or an `area`.
2. `describe_operation` before the first use of an operation in a conversation: it gives the path
   parameters, query parameters, the resolved body schema, the error codes and whether an
   Idempotency-Key is needed. Never guess field names.
3. `read` with `operation_id`, `path_params` (e.g. `{"pk": "<uuid>"}`) and `query`; `write` adds
   `body`. Most paths take `pk`; tasks take `task_id`, checklist steps `item_id`, notifications
   `notification_id` and users `user_id`.

Lists answer `{"items": [...], "next_cursor": ...}`, 50 a page: pass `query.limit` (up to 200)
and `query.cursor` for the next page. A few short lists (users, roles, import types, a task's
files) answer a plain array. Most lists take a text filter (`search` or `q`) and status filters:
describe the list operation to see them.

`get_record` shows a record's ten latest history events; `history_list` (`entity_type` such as
`projects.project`, `entity_id`) pages through all of them.

## Values

- Money, quantities, percentages, densities and rates are **JSON strings** (`"12.50"`), never
  numbers; a number is refused (422).
- Business dates are `YYYY-MM-DD`; timestamps are UTC with `Z`.
- Every money document has its own currency; the base currency is GBP.
  Leave `fx_rate` null to take the latest rate entered on or before the document's date.
- No arithmetic of your own for totals, conversions or availability: read them from the API
  (calculators and previews exist for that).
- **Hidden values are absent, never zero.** Four permissions hide fields everywhere:
  `view_cost` (costs, stock value), `view_sales_price` (quotes, invoices, licence fees, deal
  values), `view_supplier_price` (supplier prices, POs, bills) and `view_formula_pct` (formula
  percentages). If a figure is missing, say the user's role does not show it.

## Writes

- **Confirm first.** Before any `write`, say in one or two lines what will change (record, field,
  amount, quantity, status) and wait for the user's go-ahead. Postings are real: stock moves,
  issued invoices, payments and releases are corrected only by their own reversing commands
  (adjustment, void, cancel), never edited away.
- **Edits carry `expected_revision`**: the record's current `revision` from a read. A 409
  `stale_revision` means someone changed it: read it again, show the difference, ask again.
- **Idempotency keys**: stock and money postings and other one-time commands (a project's stage
  move, starting or applying an import, a report export) need one (`describe_operation` says
  `idempotency_key: true`). Make a new UUID for each intent and pass it as `idempotency_key`; if
  the outcome is unknown (5xx, timeout, `idempotency_in_flight`, `concurrency_conflict`) retry
  with the **same key and the same body**: it is made once. A new intent gets a new key.
  422 `idempotency_key_reused` means the body changed.
- Gates refuse a move with what is missing (e.g. a project stage); report the reasons instead of
  working around them.

## Errors

Errors come back as `{"error": {"code", "message", "details"}}` with the HTTP status:
- 422 `validation_error`: `details.fields` names each bad field.
- 403 `permission_denied`: the user's role cannot do it; tell them, don't retry.
- 404 `not_found`: wrong id, or a record they may not read.
- 409: `stale_revision`, `insufficient_stock` (details list each shortage), or a rule conflict.
- 429 `rate_limited`: wait and retry.

## Tasks on any record

crm-and-quotes and pl-projects show how to create, complete and reopen tasks; the rest is here.
Any signed-in user reads the tasks they may see (unlinked ones, and those on record types they
may read); every write needs `manage_tasks`. Call `describe_operation` for the bodies.

- **Read**: `tasks_list` (filters: `owner` or `person`, a user id or `me`, where person also
  finds tasks they are a member of; `status`, where `open` means To do, Doing or Blocked; `due`
  `overdue`, `today` or `week`; `label`; `record_type` with `record_id`; `search`),
  `tasks_retrieve`, and `tasks_board_retrieve`: the same filters but status, as the columns To
  do, Doing, Blocked and Done (done in the last 14 days; cancelled tasks are left off), each with
  its `count` and at most 200 cards.
- **Move**: `tasks_move_create` (`expected_revision`, `status` `open`, `doing`, `blocked`, `done`
  or `cancelled`). `blocked` needs `blocked_reason` and whom we wait on: `waiting_on_user_id` (a
  teammate) or `waiting_on_text` (anyone else, by name); leaving Blocked clears them. A done or
  cancelled task can only reopen, to `open`, `doing` or `blocked` (409
  `task_transition_invalid`). The owner and members are told when a task is blocked or done.
- **Edit**: `tasks_update` is a PUT: send `title`, `notes`, `owner_id`, `due_date` and `priority`
  (`low`, `normal`, `high`) again with `expected_revision`; `member_ids` and `label_ids` left out
  stay as they are. Owner and members are active users who can read the linked record.
- **Checklist**: `tasks_checklist_create` (`text`) adds a step; `task_checklist_update` (path
  `item_id`; `done` true or false, optional `text`) ticks, unticks or renames it;
  `task_checklist_destroy` removes it. They take no revision and answer the whole task.
- **Comments**: `tasks_comments_list` (oldest first); `tasks_comments_create` (`body`,
  `mention_ids` of active users who can see the task). Mentioned users, the owner and the
  members are notified. There is no operation to edit or delete a comment.
- **Files**: `upload_file` with doc_type `task`, then `tasks_files_create` (`attachment_id`);
  `tasks_files_list`, then `download_file`. A file belongs to one task: upload it again for
  another.
- **Labels**: `task_labels_list`, `task_labels_create` (`name`, `colour` `blue`, `green`,
  `amber`, `red` or `grey`), `task_labels_archive_create`; a task takes them as `label_ids`. An
  archived label stays on the tasks that carry it.
- `task_record_types_list` names the record types a task may link to (`crm.deal`,
  `projects.project`, `manufacturing.work_order`, `compliance.item` and others).

## Notifications

Notifications are in-app only (never email) and each user's own. `my_work` already returns the
unread ones (the newest 30) with the open tasks the user owns; use the operations for the rest:

- `notifications_list` (newest first; `unread=true` for unread only): each has a `title`,
  `event_type`, `document_number`, `read_at`, and the `entity_type` and `entity_id` of the record
  it is about.
- `notifications_read_create` (path `notification_id`) marks one read: only when the user asks.
- `notification_preferences_retrieve` lists every event type with `in_app` true or false.
  `notification_preferences_update` is a PUT that replaces the whole choice: the event types you
  list take the setting you give and every other one goes back on. To mute one more kind, send
  it together with the kinds already muted.

## Imports

The import tool turns a file into rows that are reviewed, then applied once. Each import type
needs its own area's permissions, and uploading needs `upload_attachments`.

1. `import_types_list`: the types this user may run, each with its `columns`, the `required`
   ones, `help` per column and `lookups` (columns whose cells name an existing code or name).
   The types are `tasks`, `parties`, `leads`, `deals`, `activities`, `materials`, `suppliers`,
   `supplier_prices`, `formulation_materials`, `material_compositions`,
   `allergen_declarations`, `regulatory_rules`, `accord_trials` and `opening_stock` (the stock
   skill walks through that one).
2. `upload_file` with doc_type `import` (CSV, PDF, PNG or JPEG), then `imports_create` [key]
   (`attachment_id`, `import_type` or `""` to detect it, `filename`). One import per upload:
   upload the file again for another.
3. A CSV whose header row is the type's columns (a template) is parsed inside the ERP and is
   `ready` at once. **Any other document** (a PDF, a photo, a CSV with other headers) is
   `extracting` while an AI model outside the ERP reads it: the
   whole document goes there, with the type's columns and the codes and names its lookup
   columns may hold. A template CSV is never sent. Say so and get the user's go-ahead before
   starting such an import, then read `imports_retrieve` until it is `ready` or `failed`
   (`error_code`, e.g. `undetected`, `too_large`, `ai_disabled`). A Trello board's JSON (tasks)
   and a mailbox export (activities) are converted inside the ERP, not by the AI.
4. `imports_retrieve`: the `rows` (`{"values": {column: cell}, "uncertain": [columns]}`, every
   cell a JSON string), the AI's `note` and `duplicates` (earlier imports of the same file).
   `imports_extract_create` (`import_type`) reads a ready or failed import again as another
   type, by the AI unless it is a template.
5. `imports_check_create` (optional `rows`: the corrected rows; left out: the stored ones)
   writes nothing: `issues` block the apply, `warnings` do not, `matches` names the record each
   lookup cell found and `current` what the ERP holds now. `import_lookups_list` (path `kind`,
   `search`) suggests values for a lookup column.
6. `imports_apply_create` [key] (the same optional `rows`, at most 5,000) writes all the rows
   or none: 422 `import_invalid` while an issue remains, 409 `import_state` unless the import
   is `ready`; 200 instead of 201 means it was applied before and nothing was written.
   `created_refs` lists what it created; `imports_list` shows past imports of the types the
   user may run.

Show the user the rows, issues and warnings and get their go-ahead before applying: an applied
import is not undone as a whole.

## Uploads and doc types

`upload_file` needs `upload_attachments`. Its `doc_type` decides what the file may be, who may
open it and which operation links it: an upload of the wrong doc type is refused when linked, so
upload it again with the right one. Each takes a PDF, PNG or JPEG of up to 20 MB unless noted;
a refused upload answers 422 `upload_rejected` with `details.reason` (`doc_type`,
`content_type`, `size`...).

- `general`: a file with no record of its own, shared with every ERP user.
- `task` → `tasks_files_create`: readers of the record the task is linked to.
- `import` (also CSV; a CSV, or a document the AI reads, of at most 10 MB) →
  `imports_create` [key]: holders of the import type's permissions.
- `project` → `projects_documents_create`: `view_projects`.
- `work_order` → `work_orders_documents_create`: `view_manufacturing`.
- `compliance_document`, or `compliance_formula_document` for a PIF, CPSR or allergen
  declaration → `compliance_items_attachments_create`: `view_compliance`, the second also
  `view_formula_pct`.
- `contract_signed` → `contracts_signed_copy_create`: `view_crm` and `view_sales_price`.
- Material documents `sds`, `coa`, `allergen_declaration`, `ifra_info`,
  `vegan_origin_declaration`, `tds`, `composition_declaration`, `regulatory_evidence` →
  `items_documents_create` or `supplier_items_documents_create` (which also takes
  `supplier_document`): `view_catalog`.
- `work_instruction` → a formula version's `instruction_attachments`; `bench_trial_photo` (PNG
  or JPEG) → `bench_trials_photos_create`: `view_recipes`.
- `supplier_quote` → a supplier quote's `attachment`: `view_sourcing` and
  `view_supplier_price`.
- `supplier_bill` → `bills_files_create`, `expense_receipt` → `expenses_files_create`:
  `view_money` and `view_supplier_price`.
- `regulatory_source` → a ruleset's `source_attachment`: `manage_regulatory_data`.
  `compliance_evidence` → a screening review's `evidence_attachment`: `view_recipes` and
  `view_formula_pct`.
- `assistant` (up to 5 MB): Alchemy assistant chat attachments, `use_assistant`.

Quote, invoice, payment-request, contract and purchase-order PDFs and report exports are made by
the ERP: fetch them with `download_file`, never upload them.

## Settings and reference data

Settings are the work of administrators and of holders of `manage_settings`. Confirm every
change with the user first, and expect 403 `permission_denied` for a user who may not make it.
Call `describe_operation` for the bodies.

- **Company details**: `site_retrieve` (any user) and `site_update` (an administrator: the
  user's `is_admin` flag, which no role gives): the legal name, address, company and VAT
  numbers, bank details and registration line printed on documents, base currency, timezone,
  date format and fiscal-year start. A PUT: read it first and send `base_currency`, `timezone`
  and `fiscal_year_start_month` again. 409 `site_settings_locked`: the base currency is fixed
  once a money document or a stock move exists, the fiscal-year start once a document has a
  number.
- **Users and roles** (administrators only, reads included): `users_list`, `roles_list`,
  `users_roles_update` (`role_codes` replaces all of the user's roles and counts from their
  next request), `users_active_update` (`is_active` false signs the user out and keeps their
  records), `users_destroy` (only an invite never used; else 409 `user_in_use`: deactivate
  instead), `users_sessions_revoke_create` (signs them out everywhere) and
  `users_unlock_create` (clears a sign-in lockout). **Adding a user and resetting a password
  or second factor are not done through the connector**: both answer a one-time setup code,
  which is a credential, so only the web app does them and `find_operations` does not list
  them. Send the administrator to Settings → Users: Add a user, or Reset password (Resend
  setup email for someone awaiting setup) on the user's row. Nobody deactivates or deletes
  their own account (409 `user_self`). Anyone finds colleagues by name with
  `users_lookup_list`.
- **Numbering series** (`manage_settings`): `numbering_series_list`. A series is never edited:
  `numbering_series_archive_create` it, then `numbering_series_create` (`doc_type`, `prefix`,
  `pattern` with one `{seq}` such as `{prefix}{seq:05d}`, `reset_period` `none`, `fiscal_year`
  or `month`). The new series carries the same sequence on. Do both in one go: while a document
  type has no active series, its documents cannot be numbered (422 `numbering_series_missing`).
- **Claude connections**: `connections_list` (the user's own; an administrator another user's
  with `user`) and `connections_revoke_create` (path `grant_id`), which ends one for good:
  ending the connection in use cuts this conversation off from the ERP. A new connection is
  approved only in the web app, like signing in and changing a password.
- **Area settings** are in their skills: money (currencies, exchange rates, tax rates, expense
  categories, money settings), stock (units, material types, product classes, inventory
  categories, locations), crm-and-quotes (contract templates, picklists, tracks and stages) and
  pl-projects (project settings, task templates). For any other reference list,
  `find_operations` with its name; most are written with `manage_settings`.

## Good practice

- Give the user the record's `web_url` (from `search` or `get_record`) when they may want to open
  it in the ERP.
- Keep answers short: summarise lists, show the few rows that matter, offer more.
- Costs, supplier prices and formula percentages are confidential company data: use them to
  answer the user, but don't put them into anything meant for outsiders unless the user asks.
- Use your own judgement on product wording, but use the ERP's labels for statuses and fields
  so the user can find them on screen.
