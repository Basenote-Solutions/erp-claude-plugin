---
name: lab-and-alchemy
description: The lab and the Alchemy engine in Base Note ERP - formulas (recipes) and versions (draft, approve, release) entered by percentage or by ratio and mass, accord trial versions and the trial-sheet import, formula costs, briefs, AI formula runs, substitutions, bench trials, scale-up, material formulation data, compositions and allergen declarations, IFRA Category 4 and allergen screening, rulesets and their rules, regulated substances and the accord library. Use when the user mentions a formula, recipe, revision or version, an accord or a trial sheet, ratios or a batch in grams, what a formula costs, a brief, generating a formula, a substitution, a bench trial, scale-up, a material's data or status, IFRA, allergens, a ruleset or a regulated substance.
---

# Lab and Alchemy: formulas, briefs, screening and trials

The lab designs fragrance concentrates for private-label clients and for the Blend and Alchemy
engines. Formulas (recipes) hold versions; the Alchemy engine drafts formulas from a client's
brief with AI, suggests substitutions, and screens versions against IFRA Category 4 and each
market's rules. Everything the AI proposes is only a **draft for a chemist**: predicted scent is
a prediction, and screening is a check against recorded data, never a regulatory approval.

## Records and statuses

- **Formula** (code, name, type, client, output item). Code and output item never change.
- **Formula version** (recipe revision): Draft → Approved → Released ⇄ Superseded. Drafts coexist
  (a formula's trial versions); the released version is its **best** and may change back to an
  older one. Each carries a verdict (good, bad or none), development notes and a batch size in
  grams. Lines per layer (`concentrate`, `finished`), each layer totalling 100 % w/w; a line
  entered by ratio also keeps its `ratio` and `dilution_pct`, and shows its grams of the batch.
- **Formula brief** → **formula runs** (state pending = Generating, ready, failed; kind generate or
  substitute).
- **Bench trial** (BT-0001…): Verdict Approve, Adjust or Reject; Rating 1–5. An AI draft shows
  "AI draft — not yet verified" until a trial of it says Approve.
- **Screening** (compliance evaluation) per market: Pass, Fail or Needs review; Outdated or Not
  assessed when stale. Markets: `eu`, `ni`, `gb`, `uae`, `us`; IFRA Category 4 always applies.
- **Ruleset** per pack (`ifra_cat4`, `eu`, `gb`, `uae`, `us`): Draft or Published (then frozen).
- **Material** status: Draft, Approved, Needs updated documentation, Blocked.

## Operations

Call `describe_operation` before the first use of each operation for its exact body. Versions
are named elsewhere by the pair `provider` (`"manual"`) + `external_version_id` (the
revision id).

Formulas and versions
- `recipes_list` (`formula_type`, `client`, `output_item`, `in_development`: no released version
  yet, `search`), `recipes_retrieve` (with each version's id, number and status), `recipes_create`
  (code, name, type, client; output item optional: none makes a bulk item (an accord: an
  intermediate of category Accord) with the formula's code and name; optional `lines` for the
  draft, as `recipe_revisions_update` takes them), `recipes_partial_update` (PATCH with
  expected_revision: name, type, client, notes).
- `recipes_revisions_create`: a new draft copying the chosen version (`source_revision`), else the
  latest; drafts coexist. `recipe_revisions_list` (`recipe`, `recipe_code`, `status`, `search`),
  `recipe_revisions_retrieve`.
- `recipe_revisions_update` (PUT, Draft only, expected_revision): the whole draft: product format
  and class, target markets, change note (optional), work instructions, lines `[{layer, seq, role,
  item, pct}]`; or by ratio, as the lab weighs it: concentrate lines `{layer, item, ratio,
  dilution_pct}` (the stock's %, null 100) and the solvent `{layer: finished, item, ratio}`, and
  the server works out the orders and percentages (send back `ratio`/`dilution_pct` when re-saving
  such a draft). Every line gives a ratio or none does; a diluted line needs a solvent line (its
  own ratio may be `"0"`). A version answers `dosage_pct`, `total_ratio`, `total_oil_ratio`, each
  line's `finished_pct` and, once it has a `batch_g`, its `mass_g`. `recipe_revisions_destroy`
  deletes a draft.
- `recipe_revisions_partial_update` (PATCH, any status, no expected_revision): the version's
  annotations `verdict` (good, bad or ""), `development_notes`, `batch_g` (grams, 3 decimals).
- `recipe_revisions_copy_create` (`code`, `name`): a new formula of the same type and client whose
  draft 1 copies this version.
- `recipe_revisions_approve_create` [key] (expected_revision; `approve_recipes`): checks each layer
  totals 100 (422 `percent_total_invalid`), no Blocked material (409 `material_blocked`), no
  archived one (422 `material_not_approved`) and no nesting cycle (422 `recipe_cycle`);
  `release: true` releases in the same step ("Mark as best").
- `recipe_revisions_release_create` [key] (no body; `approve_recipes`): Approved or Superseded →
  Released (the formula's best); the previous release is superseded.
- `formula_costs_retrieve` (path pk = the version's id): its cost per kg of finished product,
  material by material at today's supplier prices, else unit costs: `materials` (`finished_pct`,
  `formula_qty_kg` in 1 kg, `cost_per_kg`, `material_line_cost`, `source`),
  `concentrate_cost_per_kg`, `finished_cost_per_kg` and `missing` (`cost_missing`: a material
  without a cost leaves the totals null, never read null as zero; `ethanol_price_assumed`: the
  ethanol took the company's assumed price). No per-litre figure.

Briefs, runs and substitutions (`manage_recipes` with `view_formula_pct`)
- `briefs_list`, `briefs_retrieve`, `briefs_create`, `briefs_update` (see brief-to-formula).
- `briefs_runs_create` [key]: generate (or refine with `previous_run` or `revision` and a `note`):
  202 with a pending run; poll `formula_runs_retrieve` until `ready` or `failed` (a few minutes;
  an hourly limit applies, substitution runs included). `formula_runs_list` (`brief`, `kind`,
  `state`, `saved_revision`, `origin_of`: the run a version was drafted from).
- `formula_runs_save_create`: a ready run as a draft revision of a new recipe (code, name, type,
  client, optional output item) or of `recipe`; its screening is recorded with it; a run saves
  once.
- `substitution_runs_create` [key] (`revision`: the recipe's latest, `requests` such as vegan,
  approved suppliers only, items to remove) → poll `formula_runs_retrieve` → at most 3 options per
  problem line → `formula_runs_accept_create` (`material`, `option`) saves one as a draft: a
  draft is replaced in place, an approved or released version gets a new draft. 409
  `substitution_stale` when the formula changed since: suggest again.

Screening and rules
- `compliance_evaluations_create` [key] (`manage_recipes`): `markets`, `external_version_id`;
  needs a finished layer with an ethanol or carrier line (422 `finished_layer_required`;
  propellants are not supported). `compliance_evaluations_list` (`external_version_id`,
  `market`, `latest`), `compliance_evaluations_retrieve` (findings, allergens to declare, and
  `assessment`: current, outdated or not_assessed).
- `compliance_evaluations_review_create` [key] (`review_compliance`): `decision` `pass` or `fail`
  with a `reason`, once, on a current Needs-review market (409 `review_not_allowed` when Outdated
  or Not assessed).
- `compliance_preview_create`: screen unsaved lines, or a saved version, without recording
  anything.
- `rulesets_list` (`pack`, `status`), `rulesets_retrieve`, `rulesets_rules_list`,
  `rulesets_changes_retrieve` (against the pack's previous published ruleset), `rulesets_create`
  (`pack`, `version_label`, optional `copy_from`: a ruleset of the same pack whose rules to copy),
  `rulesets_update` (PUT, a draft, expected_revision; the pack stays), `rulesets_destroy` (a draft
  with its rules), `ruleset_rules_destroy` (one rule of a draft; rules enter a draft only through
  the `regulatory_rules` import).
- `rulesets_publish_create` [key] (expected_revision): Draft → Published, frozen; a job re-screens
  what it affects. It needs its source, licence and effective date, an IFRA one also rules and
  its amendment (422 `ruleset_incomplete` lists the problems); 409 `ruleset_effective_date_taken`
  when a published ruleset of the pack starts that day; 409 `ruleset_status_invalid` on any
  change to a published one.
- `regulated_substances_list` (`search` by name, INCI name, CAS or EC number; `name` exact),
  `regulated_substances_retrieve`, `regulated_substances_create`, `regulated_substances_update`
  (PUT, expected_revision): `name` (unique ignoring case), `cas_numbers` and `ec_numbers`
  (checked), `inci_name`, `notes`. No operation deletes a substance.
- Ruleset and substance writes need `manage_regulatory_data`; they read with `view_catalog`.

Materials and lab work
- `material_data_retrieve` / `material_data_update` (PUT: the whole formulation profile, with
  expected_revision, 0 while none is saved: family, descriptors, strength, volatility, typical
  min/max, vegan, animal-derived, natural content…), `material_data_composition_update` (PUT with
  the material data's expected_revision: replaces the whole composition, each line a `substance`
  or a `component` material at `constituent_min_pct` to `constituent_max_pct`; shares are formula
  percentages; 422 for an intermediate or bulk, whose composition is its recipe, and 422
  `composition_cycle`), `material_data_regulatory_profile_retrieve`.
- Bulk: imports `formulation_materials`, `material_compositions`, and `allergen_declarations` (a
  supplier's allergen declaration: material code, substance, CAS, % of the neat material; each
  substance set at that % × the stock's strength, the other composition lines kept).
- The lab's accord trial sheet comes in through the `accord_trials` import: one row per material
  per version (`accord`, `accord_name`, `version`, `material`, `material_name`, `dilution_pct`,
  `ratio`, `mass_g`, `made`, `notes`, `accord_notes`, `solvent`) → accord formulas with a draft
  per version, its lines by ratio; an accord that exists gets the versions after its current
  ones; a diluted version needs a `solvent` named once. It needs `view_formula_pct`. The import
  steps are in the erp-basics skill.
- `materials_approve_create` (Draft, Blocked or Needs updated documentation → Approved) and
  `materials_block_create` (Approved or Needs updated documentation → Blocked, a `reason`) need
  `approve_material`; `materials_flag_documentation_create` (Approved → Needs updated
  documentation, a `reason`) `manage_catalog`. Each sends the item's expected_revision; 409
  `material_transition_invalid` from another status or on an archived item.
- `bench_trials_list` (`recipe`, `version`, `verdict`, `made_by`, `search`), `bench_trials_retrieve`
  (with `version_status`, `stale`: the formula changed since, and `photos`), `bench_trials_create`
  (`external_version_id`, `batch_g`, `verdict` `approve`, `adjust` or `reject`, `rating`,
  `observations`, `trial_date` never after today), `bench_trials_partial_update` (PATCH,
  expected_revision; never the version), `bench_trials_photos_create` (`attachment`: an upload of
  doc_type `bench_trial_photo`, PNG or JPEG).
- `scale_ups_create`: a read (`view_recipes`, `view_formula_pct` and `view_inventory`): per
  material planned kg, needed, available, short (and value with `view_cost`) for a batch by
  `basis` `mass` (`quantity_kg`) or `units` (`units`, `fill_ml`). `scale_ups_reserve_create` [key]
  (also `manage_inventory`; `project`, `location`): reserves what one open PL project still needs
  at one location; 422 `nothing_to_reserve` when nothing is left to take there.
- `accord_library_list` (`search`, `client`): accords with the version in use and its lines.

## Rules

- **Formula percentages are hidden without `view_formula_pct`**: line %, ratios and masses,
  compositions, screening concentrations, run proposals, the AI's text, development notes and
  trial observations are absent (never zero). Editing a draft, a brief or a composition,
  generating and accepting all need it.
- Reads need `view_recipes` (material data, rulesets and substances `view_catalog`); formulas,
  drafts, briefs, runs, screenings and trials `manage_recipes`; material data `manage_catalog`.
- Costs need `view_cost` and `view_supplier_price`. Approve and release need `approve_recipes`.
- Percentages are JSON strings with up to 6 decimals; each layer must total exactly 100.
- Approval and release accept a material in any status except Blocked; a Blocked or archived
  material refuses them.
- A brief's text goes to an outside AI service: never type client names, prices or supplier
  names into it.

## Common jobs

1. **Change a released formula:** `recipes_revisions_create` → `recipe_revisions_update` with the
   whole line set → `compliance_evaluations_create` [key] → approve (with `release` when it should
   be used now).
2. **A market fails:** read the evaluation's findings (substance, limit, action), then
   `substitution_runs_create` [key] and offer the options, or reduce the line in a new draft.
3. **What does a batch need?** `scale_ups_create`, show what is short, then reserve on request.
4. **New AI formula:** follow the brief-to-formula skill.
5. **What does a formula cost?** `formula_costs_retrieve` on the version; name the materials in
   `missing` instead of giving a partial total.
6. **Record a trial as weighed:** `recipes_revisions_create` → `recipe_revisions_update` with the
   lines by ratio → `recipe_revisions_partial_update` with `batch_g`, the verdict and the notes.

## Pitfalls

- An approved version never changes: make a new draft. Saving or accepting orders, makes or
  releases nothing.
- Screening judges a range by its bounds; missing data never passes (Needs review). Re-run
  `compliance_evaluations_create` [key] after any edit.
- A version with no finished layer (a concentrate-only accord) cannot be screened: 422
  `finished_layer_required`.
- A published ruleset never changes; a ruleset or a rule is deleted only while Draft.

## Confirm with the user before

Generating (uses the AI and the hourly quota), saving or accepting a draft, approving, releasing,
deleting a draft version, publishing or deleting a ruleset or a rule, approving or blocking a
material, replacing a composition, applying an import, reserving stock.
