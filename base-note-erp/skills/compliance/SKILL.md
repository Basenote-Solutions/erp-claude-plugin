---
name: compliance
description: Compliance documents in Base Note ERP - each project SKU's, formula's and the company's checklist (IFRA certificate, SDS, MSDS, allergen declaration, PIF, CPSR, SCPN and CPNP notifications, labelling check, permits), their status, owner, reference, files and expiry, the project gates they feed, and how formula screenings relate. Use when the user asks what documents are outstanding or expiring, to mark a document done or not needed, attach a certificate, or why a project cannot go to production or ship.
---

# Compliance: the document checklist and its gates

Every PL project SKU must have its regulatory paperwork before the goods ship. The ERP keeps a
checklist per project SKU, per formula and for the company, with a status, owner, reference,
expiry date and files. The checklists feed two project gates. Formula screening against IFRA and
market rules (compliance evaluations) is lab work, described in lab-and-alchemy, and is recorded
separately from the project's release gates, which read only these checklists.

## Records and statuses

- **Compliance item** (document): scope `project_sku`, `recipe` (formula) or `company`; doc type
  `ifra_certificate`, `sds`, `msds`, `allergen_declaration`, `pif`, `cpsr`, `scpn`, `cpnp`,
  `labelling_check`, `permit`, `other`. Other and a permit need a `title`; a permit is always
  the company's.
- Status: Not started, In progress, Done, Not needed (codes `not_started`, `in_progress`, `done`,
  `not_needed`). Outstanding = not started or in progress. Statuses
  follow no fixed order: a document can be set to any of them at any time.
- Every project SKU gets its checklist when added: IFRA certificate, SDS, allergen declaration,
  PIF, CPSR, SCPN notification (UK), CPNP notification (EU), labelling check (Art. 19), owned by
  the project's owner. With client-supplied oil each SKU also gets the oil's MSDS, SDS and IFRA
  certificate, marked client oil (`client_oil` true; these three come only from the checklist,
  never by hand). Nothing is ever deleted: a mistake is marked Not needed.
- Company documents: permits, licences and other documents Base Note holds.
- Each item also shows `expiry_state` (`none`, `valid`, `expiring` within 30 days, `expired`),
  its `label`, its `attachments` and `upload_doc_type`, the doc_type its files are uploaded as.

## Operations

Call `describe_operation` before the first use of each operation for its exact body.

- `compliance_items_list`: filter `status` (`outstanding` or one status), `project`,
  `project_sku`, `recipe`, `scope`, `owner` (a user id or `me`), `doc_type`, `search` (title,
  reference, project number, SKU name or formula code), `expiring=true` (expiring within 30 days
  or expired, not Not needed, not of a cancelled project; soonest first).
- `compliance_items_retrieve`: one document with its files; `get_record` (type
  `compliance-items`) adds its history and open tasks.
- `compliance_items_create`: Add a document: `scope`, `doc_type`, the `project_sku` or `recipe`
  (neither for `company`), `title`, `owner`, `reference`, `expiry_date`, `notes`. It starts Not
  started. One per doc type per SKU or formula, Other aside: 409 `compliance_item_exists`.
- `compliance_items_partial_update` (`expected_revision`): `owner`, `title`, `reference`,
  `expiry_date`, `notes`; only the fields given change.
- `compliance_items_attachments_create` (`attachment`): links a finalized upload; no revision is
  sent. Upload first with `upload_file` (PDF, PNG or JPEG) under the item's `upload_doc_type`:
  `compliance_formula_document` for a PIF, CPSR or allergen declaration (they state formula
  percentages) and `compliance_document` for the rest; another doc_type is refused (422 on
  `attachment`). Files are evidence and stay once added.
- `compliance_items_set_status_create` (`status`, `expected_revision`, `reason`): Start, Mark
  done, Not needed, Reopen.
- Reports: `run_report` `compliance_outstanding`; the Home widget via `my_work`.
- Material documents (SDS, CoA, IFRA information from a supplier) sit on the item or supplier
  item, uploaded under their own doc_type (`sds`, `coa`, `ifra_info`, `allergen_declaration`...)
  and linked with `items_documents_create` or `supplier_items_documents_create`; a material
  needing new papers is flagged with `materials_flag_documentation_create` (lab-and-alchemy).
- Formula screening: `compliance_evaluations_list` (`external_version_id`, `market`,
  `latest=true`) and `compliance_evaluations_retrieve`: Pass, Fail or Needs review (`pass`,
  `fail`, `needs_review`) per formula version and market.

## Rules that make writes succeed

- Reads need `view_compliance`, writes `manage_compliance`, uploads `upload_attachments`. Opening
  a PIF's, CPSR's or allergen declaration's files also needs `view_formula_pct`.
- **Mark done** needs a file linked first (422 `compliance_file_required`), or for an SCPN or CPNP
  notification its reference instead (422 `compliance_reference_required`; a done one cannot
  lose its reference either). **Not needed** needs a reason (422 `reason_required`), kept in
  the document's history.
- Edits and status changes send the item's current `expected_revision` (409 `stale_revision`:
  read it again). Linking a file raises the revision too.
- An owner is an active user who can see compliance.
- Dates are `YYYY-MM-DD`. Only the item's own expiry date counts (the upload asks none).

## The gates they feed

Compliance registers two gates, which projects check:

- **Project → Production** (`projects.production`): with client-supplied oil, each live SKU's
  client-oil MSDS, SDS and IFRA certificate are Done (on a SKU made from one of our formula
  versions they may be Not needed instead). Without client-supplied oil it asks nothing.
- **Project release** (`projects.release`: → Shipped, and a work order's Receive goods or Ship
  direct): every live SKU's checklist is Done or Not needed, and none of its default documents
  is absent. The gate reads SKU documents only: formula and company documents are
  tracked separately, and an archived SKU's are ignored.
- A Done document whose expiry date has passed counts as **not done**.
- A refused gate answers 422 `compliance_incomplete` with `details.gate` (`production` or
  `release`) and `details.items`, one per document still to do: `sku_name`, `doc_type`,
  `client_oil`, `status`, `problem` (`not_done`, `expired` or `missing`) and `item`, its id.
  List them for the user by SKU. `projects_retrieve` shows both gates, met or not, without
  moving anything.

## Expiry

Expiring documents = expiring within 30 days or expired. The owner (no owner, or one who can no
longer see compliance: the administrators) is notified once when it comes within 30 days and again
when its expiry date changes; Not needed documents and those of a cancelled project are left
out. Renewing = attach the new file and update `expiry_date`.

## Common jobs

1. **What is outstanding on a project?** `compliance_items_list` with `project` and
   `status=outstanding`; group by SKU.
2. **File a certificate:** `upload_file` the PDF under the item's `upload_doc_type` →
   `compliance_items_attachments_create` → set reference and expiry with
   `compliance_items_partial_update` → `compliance_items_set_status_create` to `done`, each with
   the `revision` the step before answered.
3. **Why won't the project ship?** `projects_retrieve` (its gates), then the outstanding list
   above.
4. **What expires soon?** `compliance_items_list` with `expiring=true`.
5. **A company permit:** `compliance_items_create` with `scope` `company`, `doc_type` `permit`,
   a `title` and its `expiry_date`; attach the file and mark it done as in job 2.

## Confirm with the user before

Marking a document Done or Not needed (say which SKU and why), linking a file (it cannot be
removed), and adding documents.
