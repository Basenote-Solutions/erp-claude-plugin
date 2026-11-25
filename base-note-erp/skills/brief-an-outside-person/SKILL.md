---
name: brief-an-outside-person
description: Step-by-step workflow in BaseNote ERP for getting work to someone who has no login (a partner who runs errands or sourcing, a printer's rep, a courier) - the person as a contact on a party, cards and steps that name them, each card linked to its record, a parcel with its contents and delivery address for anything they carry or deliver, packaging details on the project's SKUs, then the printed brief for that person and day, and ticking the steps when the user says what was done. Also the project's packaging brief for a printer and its sourcing brief for a buyer. Use when the user wants to brief, hand over to or send instructions to an outside person, prepare a run sheet or to-do list for a partner, print what someone must collect or deliver, or make a packaging or sourcing brief.
---

# Workflow: brief an outside person

Someone with no login gets their work on paper: a brief, a PDF the ERP prints from its own
records for one person and one day. A brief prints what the records hold, so the work is to
get the records right first and print second. Text typed only into a brief is lost to the
ERP: the next brief, and whoever opens the project, will not have it.

Nothing is emailed. The user hands over or sends the PDF themselves.

## Working rules

- Call `describe_operation` before the first use of each operation. Edits send the record's
  current `expected_revision`.
- Cards and briefs need `manage_tasks`; naming a contact on a brief also needs
  `view_parties`; parties are written with `manage_parties`, parcels with `manage_crm`, SKUs
  and the project's papers with `manage_projects`.
- A brief leaves the company. It carries no cost, price or percentage, and you put none into
  its free text.

## 1. The person is a contact

- Find them with `contacts_list` (`search`; `role=partner` for the outside people who work
  with the company, `role=supplier` for a supplier's people).
- New: a party with the `partner` role for someone who is neither client nor supplier
  (`parties_create` with `roles: ["partner"]`), or the supplier's own party for a printer's
  rep, then `parties_contacts_create` with their name and phone. The phone prints on the
  brief.

## 2. The work is on cards and steps that name them

- A card for each errand: `tasks_create` with `title`, `owner_id` (the teammate who answers
  for it: a card is always owned by a user), `contact_id` (the outside person doing it),
  `due_date` and the record it is about (`record_type` and `record_id`).
- A step of a card that they do: `task_checklists_items_create` (or `tasks_checklist_create`
  for the first checklist) with `text`, `contact_id` and `due_date`. An existing card or step
  takes the person with `tasks_update` or `task_checklist_update` (`contact_id`).
- Link each card to its record. The link is what prints the record's details under the card:
  a parcel's address and contents, a SKU's packaging spec.
- A repeated job has a template card to copy: `tasks_list` with `template` true, then
  `tasks_copy_create` with `record_type` and `record_id` (the seeded "Packaging order
  (template)" has the steps of a packaging order, to copy onto a project).

## 3. What they carry or deliver is a parcel

- `parcels_create`: `kind` (`components`, `reference`, `documents`, `other`, or `sample` with
  its `sample_type`), what it is about (`project`, `deal`, `client` or `lead`), `to_contact`
  (whom it goes to), `address_id` (one of that contact's party's addresses, copied onto the
  parcel), `from_text` (where it is collected), `carrier_contact` (the outside person
  carrying it) and `lines`: an `item`, or a `code` and `description`, each with a `qty`.
- The delivery address belongs on the party first (`parties_addresses_create`), then on the
  parcel by `address_id`. An address found in a note or a message is put on the party, not
  left where it was found and not typed only into the brief.
- Link the errand's card to the parcel (`record_type` `crm.parcel`, `record_id`): the brief
  then prints the parcel's address and contents under that card.
- The parcel's own packing list: `parcels_pdf_create`, then `download_file` with the
  parcel's `pdf_attachment_id`.

## 4. Packaging details live on the project's SKUs

- `project_skus_partial_update`: `batch_no`, `barcode`, `box_size`, `box_material`,
  `label_size`, `label_material`, beside the bottle, cap and packaging fields. A card linked
  to the SKU (`record_type` `projects.project_sku`) prints that spec under it.
- A printer needs the whole pack, not a card: see "The project's papers" below.

## 5. Make the brief for that person and day

- `handovers_create` (the API calls a brief for a person a handover): `title`, `contact` and
  `day`. It prints the open cards the user may see that name the person, on the card or on
  an open step, and are due on or before that day or undated, in date order: each with its
  record, due date, notes, its open steps as boxes to tick with who does each, and its
  record's details. `card_ids` names the cards instead. `note` is one line under the title.
- `body` is free text (`## ` opens a section, `- ` a bullet, a blank line ends a paragraph),
  for what no record holds: how to find the place, whom to ask for, opening hours.
- 422 on `day`: no open card names this person by that day, so the cards of step 2 are
  missing or dated later.
- Read `handovers_retrieve` until `status` is `ready` (or `failed`), then
  `handovers_pdf_retrieve` for a link that lasts 5 minutes (404 `handover_pdf_not_ready`
  before that). Give the user the link.
- A brief is made once and never edited, and its cards are fixed when it is made. After a
  change to the records, make a new brief.

## 6. If the brief needs something the ERP does not hold

Add it to the ERP first, then print: the address on the party, the contents on the parcel,
the spec on the SKU, the step on the card, the file on the project. Only then is the free
text for what is left. Reading the PDF back with the user is the check: anything they add by
hand belongs in a record.

## 7. Afterwards: what was done

- When the user says what happened, tick each step done with `task_checklist_update` (`done`
  true) and finish a finished card with `tasks_done_create`.
- A step that waits for an action ticks when the action is recorded, so record the action: a
  parcel that left or arrived is moved with `parcels_move_create`, one state at a time
  (`ready`, then `sent` with `sent_on`, then `delivered` with `delivered_on`); a purchase
  order is issued; artwork is approved.
- What was not done is re-dated with the user (`tasks_update`), and shows on the next brief.

## The project's papers

Two PDFs made from a project as it stands, each queued by its call (202) and then listed,
newest of each kind, by `projects_briefs_list` with an `attachment_id` for `download_file`:

- **Packaging brief**, for a printer or packer: `projects_packaging_brief_create` (optional
  `card_id`: one of the project's cards, whose open steps print as the flow). It prints each
  live SKU's spec, an approval checklist and the artwork. Get the artwork approved first
  (pl-projects): the approved version is the one to print.
- **Sourcing brief**, for a buyer: `projects_sourcing_brief_create`. It prints a buying list
  of every SKU's reference perfumes with boxes for Got and N/A, then a page per SKU comparing
  its target notes with its closest references.

## Pitfalls

- A card with no `contact_id`, on the card or a step, is not on that person's brief.
- A card linked to the project prints the project's name, not a parcel's address: link the
  delivery card to the parcel.
- A parcel's copied address does not follow later edits of the party's address: fix the
  party, then send `address_id` again with `parcels_partial_update`.
- Supplier names, prices and costs stay out of the `body`.

## Confirm with the user before

Creating the party or contact, each card and step (title, owner, outside person, date,
record), the parcel (contents, deliver-to, who carries it), SKU spec changes, making a brief
or a project paper (say which cards or SKUs it will print), ticking steps, moving a parcel
and finishing cards.
