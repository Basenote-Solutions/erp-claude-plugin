---
name: brief-to-formula
description: Step-by-step workflow in Base Note ERP from a client's fragrance brief to a released formula - write the brief, generate and refine an AI formula, save it as a draft recipe revision, screen it for IFRA and the markets, record a bench trial, approve it, send a sample round to the client and release it. Use when the user wants to create a new fragrance or formula for a client, turn a brief into a formula, or take an AI draft through to approval.
---

# Workflow: from a client's brief to a released formula

Lab-and-alchemy describes each record and operation; this is the order to run them in. The AI
proposes; a chemist decides. Nothing here orders, makes or reserves anything.

**Before starting**, check the user holds `manage_recipes` and `view_formula_pct` (and
`approve_recipes` for steps 7 and 9): without `view_formula_pct` percentages are absent and briefs
and drafts cannot be written. Call `describe_operation` before the first use of each operation.

## 1. Write the brief

- Find the client and PL project (`search`); the brief takes the project's client when none is
  given, and a project of another client is refused.
- `briefs_create`: required `title`, `product_format` (`alcohol_spray` or `perfume_oil`),
  `dosage_pct` (the concentrate's % of the finished product, above 0 and below 100, a string such
  as `"18"`), `markets` (`eu`, `ni`, `gb`, `uae`, `us`) and `materials`, which must hold the base:
  a line with `kind` `base`, its `role` (`ethanol` for a spray, `carrier` for an oil, also `water`
  or `other`) and `pct`; leave `pct` null on one base line to take the balance. Dosage plus base
  lines total 100. Other lines: `must_include`, `excluded`, `prohibited` materials (item ids from
  `items_list`; each material once).
- An `alcohol_spray` also needs its `concentration` (`edt`, `edp`, `parfum`, `extrait` or
  `custom`); a `perfume_oil` leaves it blank.
- Describe the scent in the text fields (`description`, `family`, `top_notes`, `middle_notes`,
  `base_notes`, `strength`, `longevity`, `references`, `likes`, `dislikes`) and the product
  constraints (animal policy, natural minimum, substances to avoid, approved suppliers only, fill
  volume, batch size, loss, densities, needed by). A budget needs `view_cost` and both its basis
  and amount.
- `briefs_update` is a PUT of the whole brief with expected_revision, its material lines included.
- The brief's text goes to an outside AI service: never type client names, prices or supplier
  names into it.

## 2. Generate

- `briefs_runs_create` [key] with an optional `note`: 202 with a run in state `pending`
  (Generating). Poll `formula_runs_retrieve` about every 30-60 seconds until `ready` or `failed`
  (a few minutes). An hourly limit applies; one call starts one run.
- A failed run names why in `error_code`: `ai_disabled` (the AI is not set up), `ai_unavailable`
  or `ai_bad_output` (no usable answer: run it again), `no_candidates` (no material fits the
  brief: check its exclusions, animal policy, substances to avoid and suppliers).

## 3. Review and refine

- From the ready run show: the concentrate lines with their flags (Material not approved, Above
  its typical maximum), the Predicted scent (a prediction, not a lab result), Brief checks (Pass,
  Fail, Needs review, Not applicable), Compliance screening per market and, with `view_cost` and
  `view_supplier_price`, the cost per kg and per unit.
- To refine: `briefs_runs_create` [key] with `previous_run` (the ready run) and a `note` saying
  what to change ("more cedar, less sweet"). Repeat until the user picks one run.

## 4. Save as a draft recipe revision

- `formula_runs_save_create` on the chosen run: for a new recipe `code`, `name`, `formula_type`
  (values in `describe_operation`; default `rnd`), `client` (none: the brief's) and an
  optional `output_item` (an active intermediate, bulk or finished item; none: the recipe makes
  its own with its code and name); or `recipe` for a new draft of an existing one. Blank
  `change_note` marks it an AI draft. A run saves once; a repeat answers the same revision.
- The run then names its draft (`saved_external_version_id`), and the draft's screening for the
  brief's markets is recorded with it.

## 5. Screen and adjust

- `compliance_evaluations_create` [key] with `markets` and `external_version_id` (the revision
  id; `provider` defaults to `manual`). Each market: Pass, Fail or Needs review.
- Fail: read the findings (substance, limit, action) and either edit the draft
  (`recipe_revisions_update` with expected_revision and the whole line set, then screen again) or
  `substitution_runs_create` [key] → poll `formula_runs_retrieve` → `formula_runs_accept_create`
  (it rewrites the draft in place and records its screening).
- Needs review: a holder of `review_compliance` records Pass or Fail with a reason via
  `compliance_evaluations_review_create` [key] while the screening is current.
- What it costs: `formula_costs_retrieve` on the revision (`view_cost` and `view_supplier_price`).

## 6. Bench trial

- `bench_trials_create`: `external_version_id`, `batch_g`, `verdict` (`approve`, `adjust`,
  `reject`), `rating` 1-5, `observations`, `trial_date` (default today, never later). A trial with
  verdict Approve clears "AI draft — not yet verified". Photos: `upload_file` (doc_type
  `bench_trial_photo`) → `bench_trials_photos_create`.
- Adjust: make a new draft as in step 5 and trial again.

## 7. Approve

- `recipe_revisions_approve_create` [key] with expected_revision (`approve_recipes`): each layer
  totals 100 (422 `percent_total_invalid`) and no material is Blocked (409 `material_blocked`) or
  archived (422 `material_not_approved`); the change note is optional. Approved versions never
  change. A sample round needs an approved or released version, so approve before step 8.

## 8. Sample round to the client

- The project must be at Sampling. `projects_retrieve` gives its SKUs;
  `project_skus_sample_rounds_create` on the SKU with `provider` `"manual"`,
  `external_version_id`, `sent_items` (what was sent with it) and `sent_on` → Pending.
- When the client answers: `sample_rounds_decide_create` (`outcome` `approved` or `rejected`,
  `client_feedback`, expected_revision), once per round. An approved round's version becomes the
  SKU's. After a rejection, refine (step 3 with `revision` and a note) and send another round.
  `sample_rounds_list` with `outcome=pending` shows what is waiting.

## 9. Release

- `recipe_revisions_release_create` [key] (no body; `approve_recipes`): Approved → Released, the
  version projects use; the previous released version is superseded. (Approve with
  `release: true` does both at once.)

## Pitfalls

- Drafts coexist: a saved AI formula becomes another draft of its recipe; the chemist marks the
  best one.
- Any edit after screening leaves the market Not assessed: screen again before approving.
- 409 `substitution_stale`: the formula changed after the suggestion; suggest again.
- Screening and predicted scent are not approvals: say so when reporting results.

## Confirm with the user before

Each AI run, saving the draft (code, name, output item), every screening review, approving,
recording the client's decision and releasing.
