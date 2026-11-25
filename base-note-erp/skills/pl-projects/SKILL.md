---
name: pl-projects
description: Covers BaseNote ERP private-label (PL) projects. That means the stages from Choices received to Shipped and their gates, an administrator setting the stage of an order that began outside the ERP, SKUs with their packaging spec, barcode and target notes, artwork and its approval, reference perfumes and their matches, the packaging and sourcing briefs, the cost calculator and snapshots, the quote drafted from a snapshot, sample rounds, project tasks, cost estimates and files, the deposit and balance invoices, the projects summary, and the settings behind them (project settings and task templates). Use it when the user mentions a PL project, SKU, samples, costing, the calculator, margin, a snapshot, moving a project to a stage, on hold, project tasks, a project's deposit or balance, packaging, a barcode, artwork, a reference perfume or a match, a packaging or sourcing brief, or wants to change the contingency, overage, buffer, default density or lead days, or a task template.
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
  task templates are written with `manage_settings`. Setting a stage directly is for
  administrators (the user's `is_admin` flag, which no role gives).
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

**Setting the stage directly.** An order that began outside the ERP has no history here for
the gates to read. An administrator puts such a project at its true stage in one step with
`projects_set_stage_create` [key] (`expected_revision`, `to`: any stage but the one it is at,
`reason`). The reason is kept in the project's history and the owner is told; no stage invoice
is drafted, and `projects_retrieve` goes on listing each gate, met or not. Anyone who is not
an administrator gets 403. Do it last, once the SKUs, quote, invoices and payments that
really exist are recorded: the backfill-an-order skill gives the order. A project run in the
ERP moves with `projects_move_create`, whose gates say what is still to do.

## Operations

- **Projects**: `projects_list`, `projects_create`, `projects_retrieve`; `projects_partial_update`
  (a PATCH; changing the target delivery can take `shift_tasks`); `projects_move_create` [key]
  (body `to`, `expected_revision`, `reason`); `projects_set_stage_create` [key]
  (administrators; see above); `dashboards_projects_retrieve` (the summary: see "Common
  jobs").
- **SKUs**: `projects_skus_create` (before Approved only; `name`, `fragrance_type` (its values are in
  `describe_operation`),
  `oil_concentration_pct`, `fill_ml` and `units_ordered` are required),
  `project_skus_retrieve`, `project_skus_partial_update` (only the fields given change),
  `project_skus_archive_create` (before Approved only). Both writes also take the packaging
  spec (`batch_no`, `barcode`, `box_size`, `box_material`, `label_size`, `label_material`,
  beside the bottle, cap and packaging fields) and `target_notes` (`top`, `middle`, `base`).
- **Artwork** (a SKU's box and label designs, with approval): `project_skus_artwork_list`;
  `project_skus_artwork_create` (`kind` `box`, `label` or `other`, and `attachment_id`: an
  upload of doc_type `artwork`); `sku_artwork_send_create` (`expected_revision`, `approver`);
  `sku_artwork_decide_create` (`expected_revision`, `outcome` `approved` or
  `changes_requested`, `notes`).
- **Reference perfumes** (what a SKU's scent is measured against):
  `project_skus_references_list`; `project_skus_references_create`;
  `sku_references_partial_update` (a PATCH with `expected_revision`);
  `sku_references_destroy` (`expected_revision` in `query`);
  `project_skus_reference_matches_create` (the closest database perfumes to the SKU's target
  notes; it stores nothing).
- **Papers** (PDFs of the project as it stands, for outside people):
  `projects_packaging_brief_create` (optional `card_id`), `projects_sourcing_brief_create`,
  `projects_briefs_list` (the newest of each, with its `attachment_id` for `download_file`).
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
- **Cost lines** (estimates of direct costs of an open project that are not invoiced yet,
  such as shipping still to come, in any currency, shared over the SKUs by units produced):
  `projects_cost_lines_list`, `projects_cost_lines_create`,
  `project_cost_lines_partial_update`, `project_cost_lines_destroy` (its `expected_revision`
  goes in `query`). A supplier's invoice is never one of these: it is a bill whose lines name
  the project (the supplier-invoice-to-bill skill).
- **Files**: `upload_file` with `record_type` `projects` and `record_id` stores a file and
  puts it on the project in one call; `ask_for_file` when the user holds it.
  `projects_documents_create` (`attachment_id`) links an upload of doc_type `project` made
  without them.
- **Tasks**: a project has one card of its own: `tasks_list` with `record_type`
  `projects.project`, `record_id` and `record_card` true.
  - Its `checklists` are the project's stages, and their steps are the default task
    template's lines (`task_templates_list`), each with its person (`owner`) and `due_date`;
    tick one with `task_checklist_update` (path `item_id`, `done` true).
  - A step that carries `completes_on` ticks itself when its action is recorded for the
    project (the quote sent, samples sent, a round decided, artwork approved, the deposit or
    balance invoice issued, an RFQ sent, a purchase order issued, the project shipped): do
    the action instead of ticking the step.
  - The card's title and due date follow the project (change them with
    `projects_partial_update`), and it cannot be a template.
  - `record_card` false lists the project's other tasks (`tasks_create` with `record_type`
    `projects.project` and `record_id`, `tasks_update`, `tasks_done_create`,
    `tasks_reopen_create`, `tasks_archive_create`).
  - A template card is put on a project with `tasks_copy_create` (`record_type`
    `projects.project`, `record_id`): the seeded "Packaging order (template)" carries the
    steps of a packaging order.
  - The board, checklists, comments and files of a task are in the erp-basics skill.
- **Project settings** (the calculator's and a new project's defaults):
  `project_settings_retrieve`; `project_settings_update` (a PUT with `expected_revision`: send
  `contingency_pct`, `overage_pct`, `default_density` and `default_lead_days` every time, and
  `buffer_per_unit` when the user may see costs).
- **Task templates** (what a new project's tasks are made from): `task_templates_list`
  (`archived=true`: archived ones too); `task_templates_create` (`name`, `lines` of `title`,
  `stage`, `days_before_delivery`, `default_owner`, `completes_on`); `task_templates_update`
  (a PUT with `expected_revision` that replaces the name and every line);
  `task_templates_make_default_create` and `task_templates_archive_create` (both
  `expected_revision`).
- **Also**: `activities_create` with `project`; `recipe_revisions_list` for the formula versions
  of SKUs and rounds; `compliance_items_list` filtered by `project` (other compliance operations:
  `find_operations`); `run_report` `project_profit` with `project` (see the reports skill);
  parcels for samples and components sent (`parcels_create` with `project`: crm-and-quotes).

## Rules that make writes succeed

- **New project**: `name`, `client` (a party with the client role) and `currency` are required.
  The owner defaults to the user, the project rate to the latest rate, Target delivery to today
  plus the settings' lead days and Deposit % to the default.
- **Editing a project**: only while it is open (not Shipped or Cancelled). The currency and rate
  are fixed once the project has a quote that was not declined, the deposit % while a deposit
  invoice is live; `client_supplied_oil` cannot be turned off while a SKU uses the client's oil.
- **SKU**:
  - `units_ordered` is any whole number from 1: there is no minimum order. Units produced
    defaults to ordered plus the settings' overage %, rounded up, and is never below ordered.
    Below a supplier item's lowest price tier there is no supplier price for the part: give
    it a manual price.
  - The formula version is `provider` `manual` plus `external_version_id` (the recipe revision
    id), or `client_oil` true (only on a project with `client_supplied_oil` on).
  - The bottle, cap and packaging take supplier items (whose price tiers cost them) or manual
    prices in GBP.
  - Pricing is `pricing_method` `markup` with `markup_pct`, or `profit_share` with
    `profit_share_pct` and `retail_price`.
  - `barcode` is a GTIN of 8, 12, 13 or 14 digits whose check digit must hold (422). Enter
    the number as it was issued and ask when it is refused: never make one up or alter a digit
    to pass.
  - `target_notes` holds up to 20 notes a level (the screens call `middle` Heart). They are
    what the SKU's reference perfumes are matched against.
- **Artwork**: each upload is the next version of its SKU and kind, a draft, on an open
  project. Sending names the approver, an active user who can see projects, who is told.
  Only that approver or an administrator decides it (403 for anyone else), and asking for
  changes needs `notes` saying which. Approving supersedes the earlier versions of that SKU
  and kind, so the approved one is the one to print. 409 `artwork_status_invalid`: only a
  draft is sent and only a sent one decided. When no artwork of the project's live SKUs is
  left open, the step waiting for the artwork approval ticks.
- **Reference perfumes**: a reference is a perfume from the fragrance database
  (`perfume_source_id`: a match's `source_id`) or one typed in, with `name`, `brand`, `year`,
  `url`, a `buyer_note`, where it is close (`close_note`) and differs (`differs_note`), a
  sourcing `state` (`to_get`, `got` or `not_available`) and where it was found (`supplier`,
  `size`, `supplier_unit_price` with `currency`). On every read the ERP works out `matched`,
  `total`, `match_pct` and the `shared` notes against the SKU's target notes.
  `project_skus_reference_matches_create` (`limit`, default 10) answers `matches`, best first,
  and `unresolved` (target notes no perfume carries); no matches means the database has none.
  **Never work out a match or a similarity yourself**: the ERP scores every perfume the same
  way, and a figure of your own would not agree with the page or the sourcing brief. Report
  `match_pct` as given, and say when a SKU has no target notes to match.
- **Papers**: each call answers 202 and queues a PDF; read `projects_briefs_list` a moment
  later. The packaging brief prints each live SKU's spec, the open steps of the card named
  by `card_id` as the flow, an approval checklist and the artwork. The sourcing brief prints
  a buying list of every SKU's references with boxes for Got and N/A, then a page per SKU
  comparing its target notes with its closest references. Neither carries a cost or a price.
  Nothing is sent: the user passes the PDF on (the brief-an-outside-person skill).
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
- **Task templates**: each line makes one step on the project's card, in the checklist of its
  stage, due `days_before_delivery` days before the project's target delivery (never before
  today) and given to its `default_owner`, or to the project's owner when blank. A line's
  `completes_on` (a code from `task_actions_list`; blank: none) is the action that ticks its
  step. Only the default template is used, and only when a project is created: editing a
  template changes no step that exists. The default cannot be archived (make another the
  default first) and an archived template cannot be made the default.

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
   `accepted_quote_id`) and `open_generated_tasks`; `tasks_list` for what is due. For the
   money at a glance, `dashboards_projects_retrieve` with `project`: expected revenue, what
   was invoiced, what the client paid and owes, bills paid and owed, expenses, estimates,
   stock used, the total cost and margin, the next step, how long it has stood still and
   what it is `missing`.
6. **Change the default tasks of new projects**: `task_templates_list`; either
   `task_templates_update` the default (send every line again, changed or not), or
   `task_templates_create` a new one and `task_templates_make_default_create` it.
7. **Change a calculator default**: `project_settings_retrieve`, then `project_settings_update`
   with all the values and its `revision`; run `projects_calculate_create` on a project to show
   the effect.
8. **Artwork to approval**: `upload_file` the design with doc_type `artwork`,
   `project_skus_artwork_create`, then `sku_artwork_send_create` to the approver the user
   names. Record the decision with `sku_artwork_decide_create` only when the user is that
   approver or an administrator and has said what they decided.
9. **Find reference perfumes**: save the SKU's `target_notes`, call
   `project_skus_reference_matches_create`, show the matches with their `match_pct` and
   shared notes, and add the ones the user picks with `project_skus_references_create`.
10. **Packaging to a printer**: fill each SKU's packaging spec, get the artwork approved,
    then `projects_packaging_brief_create` and give the user the PDF from
    `projects_briefs_list`. The printer's own quote and invoice go through sourcing and a
    bill (buy-to-stock).
11. **An order that began outside the ERP**: follow the backfill-an-order skill.

## Pitfalls

- Sample rounds can be added and decided only while the project is at Sampling.
- SKUs can be added and archived only before Approved.
- A quote on the deal does not satisfy the project's gates: the quote must be the project's.
- Once the balance invoice exists (at Balance due), its deposit can no longer be edited or
  cancelled (409 `balance_invoice_exists`: cancel the balance first).
- `task_templates_update` replaces the lines: a line left out is removed from the template,
  and a line sent without its `completes_on` loses it. Send every line back whole.
- A supplier's invoice for a project is a bill, not a cost line, even when the project has no
  SKUs or purchase order yet.
- Manufacturing (work orders at a contract factory) starts at Production: an order can be
  drafted earlier but issued only at Production or Balance due (the manufacturing and
  fill-to-ship skills).

## Confirm with the user before

Every stage move (say which gate it checks and which invoice it drafts), putting a project on hold
or cancelling it, archiving a SKU, deciding a sample round, taking a snapshot, drafting or sending
a quote, recreating a stage invoice, deleting a cost line, and changing project settings or a task
template (both change what every later project gets). Also setting a stage directly (say the
stage and the reason that will be kept), sending artwork for approval and recording its
decision, adding or removing a reference, and making a packaging or sourcing brief.
