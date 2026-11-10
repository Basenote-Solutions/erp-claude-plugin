---
name: crm-and-quotes
description: Covers Base Note ERP sales work. That means leads (qualify, disqualify, convert), deals on the Private Label, Blend Engine, Alchemy Engine and Upsell tracks with their offered packages, moving deals to Won or Lost, activities, clients with their contacts, addresses and identifiers, client quotes (draft, Mark sent, accept, decline), client contracts made from house templates (edit, preview, email, signed copy, cancel, fee invoice), and the CRM settings (contract templates, picklists, tracks and stages). Use it when the user mentions a lead, prospect, deal, pipeline, stage, won, lost, upsell, an offered package or option, follow-up, call notes, a client contact or address, quoting a client, a contract, agreement, NDA, signed copy or contract template.
---

# CRM, deals, quotes and contracts

Base Note sells private-label perfume projects (the Private Label track), licences for its Blend
Engine and Alchemy Engine software, and upsells such as branding, packaging design or a website
(the Upsell track). In the CRM, **leads** are people who may buy, **deals** are what is being
sold (moving through a track's stages), **activities** record what was said, and **clients** are
parties with the client role. **Quotes** (in money) hold the price offered. **Contracts** are the
agreements a client signs, written from a house template and emailed as a PDF.

## Before any call

- Call `describe_operation` before you use an operation for the first time in a conversation, to
  get its exact body. Send decimals as JSON strings (`"1250.00"`). A value the user may not see
  (prices need `view_sales_price`) is missing from the response, never zero: say it is hidden.
- Find records with `search` (types `leads`, `deals`, `parties`, `quotes`, `contracts`) or the
  list operations. `get_record` returns one with its history, open tasks and `web_url`; give the
  user the link.
- Look up owners with `users_lookup_list` and picklists with `picklist_options_list` (kind
  `source`, `lost_reason` or `upsell_type`). Read `tracks_list` for the tracks and their stages
  (ids, order, outcome open, won or lost) instead of guessing names: settings can rename stages.
- Every edit sends the record's current `expected_revision` (a DELETE takes it in `query`). A 409
  `stale_revision` means someone changed it: read it again, apply the change again and confirm.
  A PUT replaces the whole record, so read it first and send every field back; a PATCH changes
  only the fields sent. Picklist options, tracks and stages take no revision.
- Operations marked [key] take an Idempotency-Key: pass `idempotency_key` (a new UUID you make
  per intent). After an unknown outcome (a timeout, `idempotency_in_flight` or
  `concurrency_conflict`), retry with the same key and the same body.

## Records and statuses

- **Lead**: New, Qualified, Disqualified or Converted. A converted lead links to its client and
  deal and never changes again.
- **Tracks and their seeded stages**:
  - Private Label: New lead → Discovery call → Explainer sent → Choices received → Quote sent →
    Sampling → Won or Lost.
  - Blend Engine: New lead → Demo → Proposal → Negotiation → Contract signed → Onboarding → Live
    or Lost. Contract signed, Onboarding and Live are all won stages.
  - Alchemy Engine: New lead → Demo → Pilot → Proposal → Contract signed → Live or Lost. Contract
    signed and Live are both won stages.
  - Upsell: Identified → Proposed → Won or Lost.
- **Deal**: numbered D-00001 onwards, with a value, currency and rate (giving Value GBP), an
  expected close date, a source, an owner and an outcome. Its **options** are the packages
  offered: alternatives, never summed.
- **Activity**: Call, Email, Meeting, Note or Sample. Activities are never edited or deleted;
  only a call's, meeting's or sample's status changes.
- **Quote**: Draft → Sent (numbered QT-, PDF made) → Accepted, Declined or Expired. An accepted
  quote becomes Superseded when another one is accepted for the same record.
- **Contract**: Draft → Sent (numbered AGR- at its first send) → Signed. A Draft or Sent one can
  be Cancelled. Its title and body are filled from a template when it is made and are then its
  own text. Each email of it is a row in `sends` (`state` `queued`, `sent` or `failed`).
- **Contract template**: house text with `{{placeholders}}`, archived and never deleted. Seeded:
  `development_agreement`, `manufacturing_order`, `nda` and `formula_licence`.

## Operations

- **Leads**: `leads_list` (filter by status, owner, source or `q`); `leads_create` (name and
  source required; Company decides the client); `leads_retrieve`; `leads_update` (a PUT, refused
  once converted); `leads_transition_create` [key] (status `new`, `qualified` or `disqualified`;
  disqualifying needs a `reason`); `leads_convert_create` [key] (needs `track`; `stage`, `client`
  and `title` optional).
- **Deals**: `deals_list` (filter by track, stage, outcome, owner, client, `parent_deal`, source
  or `q`); `deals_create` (an optional `follow_up` makes its first task and needs
  `manage_tasks`); `deals_retrieve`; `deals_update` (a PUT: title, contact, `via` (an agent, any
  `contacts_list` contact), `waiting_on` (us or them), owner, `estimated_value` with `currency`,
  expected close, source, upsell type, parent deal, notes); `deals_move_create` [key];
  `deals_summary_retrieve` and `tracks_board_retrieve` (counts and GBP totals per stage; null
  when nothing is priced; each card's follow-up: `chase`, `waiting_on`, last email, next task,
  latest call and sample).
- **Offered packages**: `deals_options_create` (`expected_revision`: the deal's; `name` and
  `currency` required; `development_cost`, `unit_price`, `quantity`, `moq`, `bottle_size_ml`,
  `payment_terms`, `delivery_included`, `vat_included`, `preferred`, `notes`);
  `deal_options_update` (a PUT) and `deal_options_destroy` (`expected_revision`: the option's
  own). Each answers with the whole deal, whose `options` carry `option_total`.
- **Activities**: `activities_list` (filter by deal, lead, client, project, kind or
  `created_by`), `activities_create` (an email needs `direction` sent or received, which updates
  the deal's or lead's follow-up; a call or meeting takes a `status`, booked when dated ahead,
  else held; kind `sample` needs `sample_type` existing or development),
  `activities_status_create` (`status`: a call's or meeting's `booked`, `held` or `rearrange`, a
  sample's `sent` or `feedback`), `activities_retrieve`.
- **Clients and contacts**: `parties_list` (filter by `search`, exact `name`, `role` or
  `status`), `parties_create` (`display_name`, `kind` `organisation` or `person`, `roles`
  including `client`; optional `payment_terms_days` and `vat_number`), `parties_retrieve`,
  `parties_update` (a PUT; `roles` replaces the set); `parties_contacts_create`,
  `party_contacts_update`, `party_contacts_destroy`; `contacts_list` (every party's contacts, by
  `search`).
- **Addresses and identifiers**: `parties_addresses_create`, `party_addresses_update`,
  `party_addresses_destroy` (`kind` `billing`, `shipping`, `registered` or `other`; one to four
  `lines`; `country_code` as two capital letters); `parties_identifiers_create`,
  `party_identifiers_update`, `party_identifiers_destroy` (`scheme`, an identifier scheme's code,
  and `value`, checked against the scheme's pattern).
- **Follow-ups**: `tasks_create` (title, `owner_id` and `due_date`; link it with `record_type`
  from `task_record_types_list` and `record_id`); `tasks_done_create`.
- **Quotes**: `quotes_list` (filter by status, `service`, client, `q`, or `source_type` with
  `source_id`); `quotes_create`; `quotes_retrieve`; `quotes_update` and `quotes_destroy` (drafts
  only); `quotes_send_create` (Mark sent); `quotes_accept_create`; `quotes_decline_create` (needs
  a `reason`); `quotes_invoice_create` (invoices an accepted quote: see the money skill).
- **Contracts**: `contracts_list` (filter by `status`, `client`, `q`, or `source_type` with
  `source_id`); `contracts_create` (`template` and `client` required; optional `contact`,
  `source_type` `crm.deal` or `projects.project` with `source_id`, `title`, `contract_date`,
  `fee` with `currency`, `package_lines`, `owner`, `notes`); `contracts_retrieve`;
  `contracts_partial_update` (a PATCH: the `body` and any of those particulars but the template,
  client and link); `contracts_preview_create` (no body); `contracts_send_create` [key]
  (`sent_to`; optional `cc` and `message`); `contracts_signed_copy_create` (`attachment_id`;
  optional `signed_on`); `contracts_cancel_create` [key] (needs a `reason`);
  `contracts_fee_invoice_create` [key].
- **Contract templates**: `contract_templates_list` (`archived` true adds the archived ones; each
  names the `placeholders` it uses), `contract_templates_retrieve`, `contract_templates_create`
  (`code`, `name`, `title`, `body`), `contract_templates_update` (a PUT: `name`, `title`,
  `body`), `contract_templates_archive_create`; `contract_placeholders_list` (every placeholder a
  template may name, with its meaning).
- **CRM settings**: `picklist_options_create` (`kind`, `code`, `name`, `sequence`),
  `picklist_options_update` (a PUT: `name` and `sequence`), `picklist_options_archive_create`;
  `tracks_create` (`code`, `name`, `sequence`), `tracks_update` (a PUT: `name` and `sequence`),
  `tracks_archive_create`; `stages_create` (`track`, `code`, `name`, `sequence`, `outcome`
  `open`, `won` or `lost`), `stages_update` (a PUT: `name`, `sequence`, `outcome`),
  `stages_archive_create`. Stages have no list of their own: read them in `tracks_list`.
- **Exchange rate**: `fx_rates_latest_retrieve` gives the default rate on a date.

## Rules that make writes succeed

- **Permissions**: CRM reads need `view_crm`, writes `manage_crm`. Writing an offered package,
  and reading or writing a contract (its body prints the fee), also need `view_sales_price`; the
  fee invoice needs `manage_money` too. Clients, contacts, addresses and identifiers need
  `view_parties` and `manage_parties`. Quotes need `view_money` to read and `manage_money` to
  write, with `view_sales_price` to create or edit one. Tasks need `manage_tasks`. Contract
  templates, picklists, tracks and stages are settings: anyone signed in reads them, changing
  them needs `manage_settings`.
- **Convert**: the lead must be New or Qualified (a Disqualified lead goes back to New first).
  - The client is the party already named like the lead's company (else its name), in any case.
    When none matches, a new client is made. When several match, the API returns 409
    `lead_match_ambiguous`: list the candidates, ask the user, and pass the one as `client`.
  - The lead becomes a contact of the client. The deal is owned by the lead's owner, who must be
    an active user.
- **New deal**: title, client and track are required, and client and track cannot change later.
  The deal starts at the track's first open stage. The rate defaults to the latest one;
  `fx_rate_missing` means the currency has no rate yet.
- **Move**: `stage` must be an active stage of the deal's own track (422 `deal_stage_invalid`
  otherwise). A lost stage needs `lost_reason`, an option of kind `lost_reason`
  (`deal_lost_reason_required` otherwise). Leaving Won or Lost for another outcome needs a
  `reason`.
- **What winning does** (once, in the same move): a Private Label deal makes its PL project at
  Choices received with its task list (see the pl-projects skill). A Blend Engine or Alchemy
  Engine deal drafts a licence with the deal's value as the fee (see the money skill). An Upsell
  deal makes nothing. On the two engine tracks the first won stage is Contract signed: moving the
  deal there is the win. Signing a contract record moves no deal.
- **Upsell**: `deals_create` on the Upsell track for the same client, with `parent_deal` and
  `upsell_type`.
- **Offered packages**: marking one `preferred` (one per deal; the last one marked wins) sets the
  deal's value to its total, the development cost plus the unit price times the quantity, in the
  option's currency. While a deal has a preferred option, `deals_update` cannot change its value
  or currency (422): edit or unmark the option. Dropping an option leaves the value as it was.
- **Activity**: needs kind, subject and one of deal, lead, client or project. One on a deal takes
  the deal's client.
- **Client records**: every contact, address and identifier write sends the party's own
  `expected_revision` and answers with the whole party. Quote and invoice PDFs print, and a new
  contract's address placeholder takes, the client's `billing` address, else its `registered`
  one, else its first, with the party's `vat_number`. A VAT number is that field, not an
  identifier. No operation lists the identifier schemes: an unknown `scheme` code answers 422.
- **New quote**: client, currency, `service` (`pl`, `blend`, `alchemy`, `upsell` or `other`) and
  lines (description, qty, unit_price, tax_rate; blank tax rate: the default; a negative line is a
  credit). Link it to its record with `source_type` and `source_id` (for example `crm.deal`,
  `projects.project` or `money.licence`). The contact must belong to the client. The server works
  out the totals.
- **Accept**: only while the quote is valid (`quote_expired` otherwise). Each record has one
  accepted quote: accepting a new one supersedes the old one and cancels its draft invoices,
  unless an invoice from the old one was already issued (409 `quote_already_accepted`).
- **Private Label quotes**: the quote that counts for a PL project's gates is the project's own
  (source `projects.project`), usually drafted from a costing snapshot. A quote on the deal does
  not move the project.
- **New contract**: `template` is an active template and `client` a party with the client role;
  the `contact` is one of the client's; a link names one of the client's own deals or projects. A
  `fee` needs a `currency`. The owner defaults to the user and the date to their today. The title
  and body are filled once, from the template's placeholders (the client, contact, company, date,
  fee, currency, package lines, linked record and owner); one with nothing to fill it is left
  blank.
- **Editing a contract**: while Draft or Sent (409 `contract_locked` once Signed or Cancelled).
  In the `body`, `## 1. Heading` opens a section, `- ` a bullet and a blank line ends a
  paragraph. Changing `fee`, `currency`, `package_lines` or `contact` later does not rewrite the
  body: edit its text to match.
- **Preview**: `contracts_preview_create` answers 202 with nothing and the worker renders the
  PDF. Read the contract again for `pdf_attachment_id` (its newest PDF, a preview's or a send's)
  and fetch it with `download_file`. A cancelled contract is refused (409
  `contract_transition_invalid`), and so is an empty body.
- **Send**: a Draft or Sent contract with a body (409 `contract_transition_invalid` once Signed
  or Cancelled). The first send numbers it AGR- and makes it Sent. The worker emails `sent_to`
  the contract PDF with the `message`; `cc` is copied in, and replies go to it. Read
  `sends[].state` later: `failed` carries an `error_code`, the contract stays Sent either way,
  and sending again makes a new send. Each send keeps the title and body as they stood.
- **Signed copy**: upload the client's signed file with `upload_file` (doc_type
  `contract_signed`; PDF, PNG or JPEG), then `contracts_signed_copy_create` with its
  `attachment_id`. Only a Sent contract can be signed (409 `contract_transition_invalid`).
  `signed_on` defaults to the upload's issue date, else today. A Signed contract changes no more.
- **Cancel a contract**: Draft or Sent only, with a `reason`.
- **Fee invoice**: `contracts_fee_invoice_create` [key] drafts a Standard invoice for the
  contract's client and contact: one line, the title with the package lines under it, quantity 1
  at the fee in the contract's currency, linked to its deal or project (else to the contract). It
  needs a fee and a currency (422 `contract_fee_missing`), is refused for a Cancelled contract,
  and is made once while one is live (409 `contract_invoice_exists`: cancel that invoice to draft
  another). The answer names the `invoice`; issue it as the money skill says.
- **Templates**: a title or body naming a placeholder the ERP does not know is refused (422):
  check the names with `contract_placeholders_list`. A template's `code` never changes. Editing
  or archiving one changes no contract already made; an archived one is refused for new ones.
- **Tracks, stages and picklists**: a `code` never changes. A track's `code` and `sequence`, a
  stage's within its track and an option's `code` within its kind must be unique (422). A stage's
  `outcome` changes only while no deal sits in it (409 `crm_stage_in_use`). An archived entry
  stays on its records and is refused for new use; there is no restore. Only the seeded track
  codes act on a win (`private_label`: a project; `blend_engine` and `alchemy_engine`: a
  licence).

## Common jobs

1. **New lead**: get the source option id, then `leads_create` with name, company, email, phone
   and source. Log the first call with `activities_create` (kind `call`, `lead`).
2. **Qualify and convert**: `leads_retrieve` for its revision, `leads_transition_create` [key] to
   `qualified`, then `leads_convert_create` [key] with the track id from `tracks_list`. Report
   the client and deal it made.
3. **Move a deal**: `deals_retrieve` for its revision and track, `tracks_list` for the target
   stage id, then `deals_move_create` [key]. For Lost, ask the reason and map it to a
   `lost_reason` option.
4. **Quote a deal** (Blend, Alchemy or Upsell): `quotes_create` with the deal's client,
   `source_type` `crm.deal` and the lines; show the totals; on the user's yes, `quotes_send_create`.
   Fetch the PDF with `download_file` from the quote's `pdf_attachment_id`, which can be empty for
   a moment while the worker renders it.
5. **The client's answer**: `quotes_accept_create` or `quotes_decline_create` with the quote's
   revision.
6. **Pipeline at a glance**: `deals_summary_retrieve`, or `run_report` `pipeline` (see the reports
   skill).
7. **Offer packages on a deal**: `deals_options_create` once per alternative, each with the
   `revision` of the deal as last answered; mark the one the client leans to `preferred`.
8. **Draft and send a contract**: pick the template from `contract_templates_list`;
   `contracts_create` for the client, linked to its deal or project; go through the filled `body`
   with the user and correct it with `contracts_partial_update`; `contracts_preview_create`, then
   show the PDF; on the user's yes to the recipient, CC and message,
   `contracts_send_create` [key]; read the contract again for the send's `state`.
9. **Record the signature**: `upload_file` the signed copy as `contract_signed`, then
   `contracts_signed_copy_create`. For a Blend Engine or Alchemy Engine deal, then move the deal
   to Contract signed with `deals_move_create` [key].
10. **Bill the contract's fee**: `contracts_fee_invoice_create` [key], then issue the draft
    invoice (see the money skill).

## Pitfalls

- Mark sent (a quote) emails nothing: the user sends the PDF to the client. Sending a contract
  does email the client.
- A deal's value in another currency needs an exchange rate. Value GBP uses the latest rate unless
  a rate is typed.
- Moving a deal to the stage it is already at changes nothing.
- A contact that a deal, project, quote, invoice or contract names cannot be removed (409
  `party_child_in_use`).
- Search before `parties_create` so you don't make a duplicate client.
- The seeded templates leave the deal's own details as blanks in square brackets (`[number]`,
  `[days]`): fill them in the `body` before sending.
- A contract is filled when it is made. A contact, address or fee added to the client or the
  contract afterwards is not in the body until you write it there.
- A failed send still leaves the contract Sent and numbered. Read the send's `error_code`
  (`email_rejected`: the provider refused the message), put it right and send again.

## Confirm with the user before

Converting or disqualifying a lead; creating, editing or moving a deal (above all to Won, which
creates a project or licence, or to Lost); adding, changing or dropping an offered package (a
preferred one resets the deal's value); sending, accepting, declining or deleting a quote;
creating, editing, sending, signing or cancelling a contract, or drafting its fee invoice;
removing a contact, address or identifier; changing a template, picklist, track or stage. State
the record, the change and its consequence, then wait for a yes.

Sending a contract emails a client and cannot be recalled: show the recipient, the CC and the
message word for word, and have the user approve the previewed PDF. Put nothing in the body or
message the user did not approve, and never costs, supplier names or formula percentages.
