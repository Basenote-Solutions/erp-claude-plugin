---
name: pl-projects
description: Covers Base Note ERP private-label (PL) projects. That means the stages from Choices received to Shipped and their gates, SKUs, the cost calculator and snapshots, the quote drafted from a snapshot, sample rounds, project tasks, cost lines and files, the deposit and balance invoices, and the settings behind them (the calculator's defaults in project settings, and the task templates that give a new project its tasks). Use it when the user mentions a PL project, SKU, samples, costing, the calculator, margin, a snapshot, moving a project to a stage, on hold, project tasks or a project's deposit or balance, or wants to change the contingency, overage, buffer, default density or lead days, or a task template.
---

# Private-label projects

A PL project is one client's private-label perfume order: its SKUs (each product), samples,
costs, quote, deposit and balance, run step by step through fixed stages. Winning a Private
Label deal makes the project by itself (see the crm-and-quotes skill). It can also be created
directly.

## Before any call

- Call `describe_operation` before you use an operation for the first time in a conversation, to
  get its exact body. Send decimals as JSON strings (`"12.5"`).
- Hidden values are missing from the response, never zero: say they are hidden. Costs need
  `view_cost`, prices `view_sales_price`, the margin both, and Profit needs `view_cost`,
  `view_sales_price` and `view_supplier_price`.
- Reads need `view_projects`, writes `manage_projects`. A snapshot also needs `view_cost`. The
  quote from a snapshot also needs `manage_money` and `view_sales_price`. Project settings and
  task templates are written with `manage_settings`.
- Every edit sends the record's `expected_revision`. A 409 `stale_revision` means someone changed
  it: read it again, apply the change again and confirm.
- Operations marked [key] take an Idempotency-Key. After an unknown outcome, retry with the key
  you passed (a new UUID per intent) and the same body.
- Find a project with `search` (type `projects`) or `projects_list` (filter by `stage`, `client`,
  `owner` or `search`; `board=true`: open projects and those shipped in the last 30 days).
  `get_record` returns it with its history, open tasks and `web_url`.

## Stages and gates

The stages are Choices received → Quoted → Sampling → Approved → Production → Balance due →
Shipped, plus On hold and Cancelled (codes `choices_received`, `quoted`, `sampling`, `approved`,
`production`, `balance_due`, `shipped`, `on_hold`, `cancelled`). A project is numbered PL- and
moves one step forward at a time, through each gate (an unmet gate answers 422 with its code):

- **To Quoted**: a quote for the project has been sent or accepted (`quote_not_issued`).
- **To Approved**: the client's accepted quote (`accepted_quote_missing`) and at least one SKU,
  every one with an approved sample round (`samples_not_approved` names the SKUs). Entering
  Approved drafts the **deposit invoice**: the Deposit % of the accepted quote
  per tax rate, or none at 0 %.
- **To Production**: the deposit invoice paid (`deposit_unpaid`; not asked at a 0 % deposit); a
  formula version or the client's oil on every SKU (`formula_missing`); with client-supplied oil,
  its MSDS, SDS and IFRA certificate Done (`compliance_incomplete`).
- **To Balance due**: drafts the **balance invoice**, what the deposit left of the quote (none
  when the deposit was 100 %).
- **To Shipped**: the balance paid (`balance_unpaid`, with what is still outstanding) and every
  SKU's compliance documents Done or Not needed (`compliance_incomplete` names them; a document
  done but expired counts as not done).

`projects_retrieve` lists every gate, met or not, with its code. Read the gates before moving a
project and tell the user what is missing.

## Operations

- **Projects**: `projects_list`, `projects_create`, `projects_retrieve`; `projects_partial_update`
  (a PATCH; changing the target delivery can take `shift_tasks`); `projects_move_create` [key]
  (body `to`, `expected_revision`, `reason`).
- **SKUs**: `projects_skus_create` (before Approved only; `name`, `fragrance_type` (its values are in
  `describe_operation`),
  `oil_concentration_pct`, `fill_ml` and `units_ordered` are required),
  `project_skus_retrieve`, `project_skus_partial_update` (only the fields given change),
  `project_skus_archive_create` (before Approved only).
- **Calculator**: `projects_calculate_create` (a calculator run: a POST that stores nothing;
  `overrides` per SKU id try other values); `projects_costing_records_create` [key] (a snapshot
  of the saved inputs; no body); `projects_costing_records_list` and `costing_records_retrieve`;
  `costing_records_quote_create` [key] (a draft quote from a snapshot).
- **Samples**: `project_skus_sample_rounds_create` (only at Sampling; `provider`,
  `external_version_id`, `sent_items`, `sent_on`); `sample_rounds_decide_create` (only at
  Sampling; `outcome` `approved` or `rejected` with `client_feedback`; each round is decided
  once: 409 `sample_round_decided`); `sample_rounds_list` (filter by `project`, `sku` or
  `outcome`).
- **Stage invoices**: `projects_stage_invoices_create` [key] with `kind` `deposit` (from Approved
  on) or `balance` (from Balance due on) drafts one again after the live one was cancelled (409
  `stage_invoice_exists` while one is live). A contract's fee invoice for the project (see
  crm-and-quotes) is a standard invoice: it is listed with the project's invoices and counts in
  its profit, but it is not the deposit and does not count towards a gate.
- **Cost lines** (shipping and other direct costs of an open project, in any currency, shared
  over the SKUs by units produced): `projects_cost_lines_list`, `projects_cost_lines_create`,
  `project_cost_lines_partial_update`, `project_cost_lines_destroy` (its `expected_revision`
  goes in `query`).
- **Files**: `upload_file` with doc_type `project`, then `projects_documents_create` with the
  `attachment_id`.
- **Tasks**: `tasks_list` with `record_type` `projects.project` and `record_id`; `tasks_create`,
  `tasks_update`, `tasks_done_create`, `tasks_cancel_create`, `tasks_reopen_create`. The default
  task template's lines (`task_templates_list`) became tasks when the project was made. The
  board, checklists, comments and files of a task are in the erp-basics skill.
- **Project settings** (the calculator's and a new project's defaults):
  `project_settings_retrieve`; `project_settings_update` (a PUT with `expected_revision`: send
  `contingency_pct`, `overage_pct`, `default_density` and `default_lead_days` every time, and
  `buffer_per_unit` when the user may see costs).
- **Task templates** (what a new project's tasks are made from): `task_templates_list`
  (`archived=true`: archived ones too); `task_templates_create` (`name`, `lines` of `title`,
  `stage`, `days_before_delivery`, `default_owner`); `task_templates_update` (a PUT with
  `expected_revision` that replaces the name and every line);
  `task_templates_make_default_create` and `task_templates_archive_create` (both
  `expected_revision`).
- **Also**: `activities_create` with `project`; `recipe_revisions_list` for the formula versions
  of SKUs and rounds; `compliance_items_list` filtered by `project` (other compliance operations:
  `find_operations`); `run_report` `project_profit` with `project` (see the reports skill).

## Rules that make writes succeed

- **New project**: `name`, `client` (a party with the client role) and `currency` are required.
  The owner defaults to the user, the project rate to the latest rate, Target delivery to today
  plus the settings' lead days and Deposit % to the default.
- **Editing a project**: only while it is open (not Shipped or Cancelled). The currency and rate
  are fixed once the project has a quote that was not declined, the deposit % while a deposit
  invoice is live; `client_supplied_oil` cannot be turned off while a SKU uses the client's oil.
- **SKU**:
  - `units_ordered` must meet the minimum order (422 `minimum_order`). Units produced defaults to ordered
    plus the settings' overage %, rounded up, and is never below ordered.
  - The formula version is `provider` `manual` plus `external_version_id` (the recipe revision
    id), or `client_oil` true (only on a project with `client_supplied_oil` on).
  - The bottle, cap and packaging take supplier items (whose price tiers cost them) or manual
    prices in GBP.
  - Pricing is `pricing_method` `markup` with `markup_pct`, or `profit_share` with
    `profit_share_pct` and `retail_price`.
- **Calculator**: a missing price returns `cost_missing`, naming the SKU and the part (give it a
  supplier item or a manual price, or the formula's materials a cost). A retail price at or below
  cost returns `retail_below_cost`. Trying another pricing in `projects_calculate_create` needs
  `view_cost`.
- **Snapshot and quote**: a snapshot freezes the saved inputs, so save SKU changes first. The
  quote has one line per SKU (units ordered × the client unit price), plus the formula ownership
  fee when the client buys the formula (set `formula_ownership_fee` on the project first), in the
  client currency at the project rate. It is allowed only at Choices received, Quoted or
  Sampling, and returns 409 `snapshot_stale` if the project's currency or rate changed since the
  snapshot: take a new one.
- **Move**:
  - Forward one step; back one step with a `reason`.
  - On hold: `to` `on_hold` with a `reason`. Resume: `to` is the project's `prior_stage`.
  - Cancel: `to` `cancelled` with a `reason`.
  - A missing reason answers 422 `reason_required`.
  - Shipped and Cancelled are final, and both release the stock reserved for the project. Any
    other move returns 409 `project_stage_invalid`.
- **Sample rounds**: a round names an approved or released formula version (a SKU of the client's
  oil may name none). An approved round's formula version becomes the SKU's.
- **Project settings**: `contingency_pct` (extra oil the calculator allows), `overage_pct` (the
  default units produced above those ordered), `buffer_per_unit` (unless the SKU has its own),
  `default_density` (g/mL, when neither the SKU nor its formula gives one) and
  `default_lead_days` (a new project's target delivery, 0 to 730). A change counts from then on
  (the next calculator run, SKU saved or project created); snapshots keep what they froze.
- **Task templates**: each line makes one task, due `days_before_delivery` days before the
  project's target delivery (never before today) and owned by its `default_owner`, or by the
  project's owner when blank. Only the default template is used, and only when a project is
  created: editing a template changes no task that exists. The default cannot be archived (make
  another the default first) and an archived template cannot be made the default.

## Common jobs

1. **Cost and quote**: add the SKUs; run `projects_calculate_create` and show the cost per unit,
   price per unit and margin the user may see; save the inputs with `project_skus_partial_update`;
   take a snapshot with `projects_costing_records_create` [key]; draft the quote with
   `costing_records_quote_create` [key] (`valid_until`, `tax_rate`); send it with
   `quotes_send_create`.
2. **Through sampling**: move to Quoted, then to Sampling. Create a round per SKU with
   `project_skus_sample_rounds_create` and record each answer with `sample_rounds_decide_create`.
   Once the quote is accepted, move to Approved and report the draft deposit invoice it made (in
   the project's `money`).
3. **Deposit to Production**: issue the deposit invoice and collect the payment (see the money
   skill), then move to Production.
4. **Finish**: move to Balance due; issue and collect the balance invoice; get every SKU's
   compliance documents to Done or Not needed; move to Shipped.
5. **Status check**: `projects_retrieve` for the stage, the gates, its `money` (quotes, invoices,
   `accepted_quote_id`) and `open_generated_tasks`; `tasks_list` for what is due.
6. **Change the default tasks of new projects**: `task_templates_list`; either
   `task_templates_update` the default (send every line again, changed or not), or
   `task_templates_create` a new one and `task_templates_make_default_create` it.
7. **Change a calculator default**: `project_settings_retrieve`, then `project_settings_update`
   with all the values and its `revision`; run `projects_calculate_create` on a project to show
   the effect.

## Pitfalls

- Sample rounds can be added and decided only while the project is at Sampling.
- SKUs can be added and archived only before Approved.
- A quote on the deal does not satisfy the project's gates: the quote must be the project's.
- Once the balance invoice exists (at Balance due), its deposit can no longer be edited or
  cancelled (409 `balance_invoice_exists`: cancel the balance first).
- `task_templates_update` replaces the lines: a line left out is removed from the template.
- Manufacturing (work orders at a contract factory) starts at Production: an order can be
  drafted earlier but issued only at Production or Balance due (the manufacturing and
  fill-to-ship skills).

## Confirm with the user before

Every stage move (say which gate it checks and which invoice it drafts), putting a project on hold
or cancelling it, archiving a SKU, deciding a sample round, taking a snapshot, drafting or sending
a quote, recreating a stage invoice, deleting a cost line, and changing project settings or a task
template (both change what every later project gets).
