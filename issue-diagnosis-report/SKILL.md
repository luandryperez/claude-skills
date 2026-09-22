---
name: issue-diagnosis-report
category: "Client"
description: >
  Use this skill when a client reports that an Odoo record looks wrong and you must
  decide whether it is a real bug before answering them.
  Investigate a potential bug or issue a client reported about Odoo records and
  produce a deliverable HTML diagnosis report (Vauxoo design). The skill reproduces
  the questioned behavior from data, cross-references the record chatter for the
  business "why", and renders a verdict — confirmed bug, expected behavior,
  data-entry issue, process issue, or configuration issue — plus options-by-need.
  Use this skill when a client asks "why is this record in this state?", "is this a
  bug?", "this looks wrong", "the invoiced quantity is 0 but there is an invoice", or
  escalates any unexpected Odoo behavior on a sale order, invoice, picking, payment,
  or task — even if they never say "bug".
  Triggers on: "is this a bug", "es un bug", "reporte de bug", "issue reportado",
  "por qué está en este estado", "why is it in this state", "diagnostica este caso",
  "diagnose this issue", "client reported", "unexpected behavior",
  "informe de diagnóstico", "diagnosis report".
metadata:
  domain: odoo
  output: html
  profile_aware: true
  verdicts: [bug, expected_behavior, data_entry, process, configuration]
  source: "adapted from git.vauxoo.com/NicolasOT/hive-mind-agents (skills/issue-diagnosis-report), original ecosystem by Nhomar Hernandez / Vauxoo — retargeted from the odoo-mcp CLI to the mcp__odoo__ tools available in this environment"
---

# Issue Diagnosis Report

Turn a vague client question ("is this a bug?") into an evidence-backed verdict and
a deliverable HTML report. The skill is **read-first and profile-aware**: it diagnoses
the client's own instance, never assuming a default profile and never writing without
explicit confirmation.

This is a **diagnostic** skill, not a fixer. It explains *what happened, why, whether
it is a defect, and what options exist* — it does not change records on its own.

---

## Setup

- **Profile (MANDATORY)**: this skill runs against the **client's instance**, not
  necessarily the default one. Call `mcp__odoo__list_available_profiles` first, match
  the request to a profile by url/database, and pass that `profile` on every
  `mcp__odoo__search_read` call. If the profile is ambiguous, **stop and ask** — never
  assume the default.
- **Template**: `assets/diagnostic-report.html` (in this skill folder). Ya incluye los
  `@media` responsivos; **no quitarlos** — el HTML debe verse bien en celular y
  computadora. No reescribir la capa de marca (`:root` + clases), solo rellenar
  contenido.
- **Marca (fuentes + logo reales)**: el template trae los marcadores `<!-- VX_FONTS -->`
  y `<!-- VX_LOGO official-logo-horizontal -->`. Después de rellenar los placeholders,
  correr `python3 ~/.claude/skills/_vx-shared/tools/vx_brand.py inline <archivo.html>`
  para embeber la tipografía Sora/Manrope real (base64, sin CDN) y el SVG oficial del
  logo. Requiere el plugin `agents-brand-guide-vauxoo@vauxoo-brand` instalado
  (`claude plugin list` para confirmar).
- **QA antes de entregar** (opcional pero recomendado): `python3
  ~/.claude/skills/_vx-shared/tools/html_report_lint.py <archivo.html>` valida que no
  queden hexes fuera de paleta, `box-shadow`, fuentes remotas ni placeholders sin
  rellenar; `python3 ~/.claude/skills/_vx-shared/tools/es_ortografia_lint.py
  <archivo.html>` valida ortografía en español.
- **Output language**: the SKILL is English, but the **rendered report MUST be in the
  client's language** (e.g. Spanish). Localize every heading and sentence at render time.

---

## The verdict framework

Every diagnosis resolves to exactly one primary verdict. Pick with evidence, not opinion.

| Verdict | Badge | When it applies |
|---|---|---|
| **Expected behavior** | `badge-green` | The system did exactly what its rules dictate. The "problem" is a misunderstanding of how a (usually computed) field works. Most common outcome. |
| **Data-entry issue** | `badge-yellow` | A record was captured manually in a way that produces the odd state (wrong quantity, missing link, free line not tied to its origin). |
| **Process issue** | `badge-yellow` | The sequence of legitimate actions caused it (e.g. a full-quantity credit note used to adjust a price; a line added after invoicing). |
| **Configuration issue** | `badge-blue` | A setting/policy explains it (invoice policy, fiscal position, rounding, sequence). |
| **Confirmed bug** | `badge-red` | Reproducible behavior that contradicts the framework/standard. Requires a minimal reproduction and a code/standard reference. |

> Default skepticism: **assume "expected behavior" until the data forces otherwise.**
> A confirmed bug needs a concrete reproduction, not a feeling.

---

## Phase 1 — Intake

### Step 1: Capture the exact question

Record verbatim: the symptom the client observed, the field/value they expected, and the
record(s) involved. Quote it — it becomes §0 of the report.

### Step 2: Resolve profile and records

List profiles, select the client's instance, then locate the records by name/id with
`mcp__odoo__search_read`, e.g. model `sale.order`, domain
`[('name','in',['S03563','S03515'])]`, fields
`id,name,state,invoice_status,amount_total,invoice_ids,partner_id`.

Save the ids of the questioned records and every related record (invoices, credit notes,
pickings, payments, tasks).

---

## Phase 2 — Reproduce the state from data

Pull the records and the lines/links that drive the questioned field. Read the **computed
field inputs**, not just the field. For an invoicing question that means line quantities
and their `invoice_lines` / `sale_line_ids` links via `mcp__odoo__search_read`:

- Model `sale.order.line`, domain `[('order_id','in',[<ids>])]`, fields
  `id,order_id,product_id,product_uom_qty,qty_invoiced,qty_to_invoice,invoice_status,invoice_lines`.
- Model `account.move.line`, domain `[('id','in',[<line_ids>])]`, fields
  `move_id,move_name,parent_state,move_type,product_id,quantity,price_subtotal,sale_line_ids`.

Reconstruct the math by hand and confirm it matches what the client sees. **If you cannot
reproduce the value from the data, do not guess** — keep querying (related model, draft
records, reversals) until the number is fully explained.

---

## Phase 3 — Audit trail (the business "why")

Read the chatter of the questioned record **and** of every related document. This is where
"expected behavior" becomes a documented story instead of a bare assertion.

Model `mail.message`, domain
`[('model','=','sale.order'),('res_id','=',<id>),('message_type','in',['comment','email'])]`,
fields `date,author_id,subject,body`, order `date asc`. Repeat with `model = 'account.move'`
and the invoice/credit-note id for its chatter.

Extract dated, attributed evidence: who decided what and why (a refund agreed with the
client, a line added after invoicing, an amount corrected later). Each item becomes a row
in the record's traceability table. Cross-check `create_date` / `write_date` of suspect
lines against document dates to prove "added after the fact".

---

## Phase 4 — Classify the verdict

Apply the framework with this checklist:

- [ ] The questioned value is **fully reproduced** from data (Phase 2).
- [ ] The mechanism (computed-field rule / policy) is identified and stated.
- [ ] The chatter explains the human decision behind the state (Phase 3).
- [ ] The financial/operational reality is reconciled (e.g. "net invoiced = order total →
      fully collected").
- [ ] Only if the behavior **contradicts** the mechanism → escalate to **Confirmed bug**
      with a minimal reproduction and a documented standard/code reference.

State one primary verdict per record and one overall verdict for the report.

---

## Phase 5 — Options by need

Clients usually want an outcome (e.g. "move the status to Invoiced"), not just an
explanation. Offer options ordered **fastest → most robust**, each with its caveat:

1. **Quick override** (e.g. a server action that forces the state). Valid when reality is
   already settled. **Always state the caveat** when overriding a *stored computed field*:
   it can be recomputed/reverted when the record or its dependencies change — document the
   reason in the chatter.
2. **Root-cause fix** (re-link records, redo the document correctly). Permanent, more work.
3. **Do-not-do options** — explicitly flag any path that causes harm (e.g. re-invoicing an
   already-collected line → **double billing**). Naming the trap is part of the value.
4. **Prevention** — the practice that avoids the issue next time.

> Never recommend a write that the client has not asked for. Present options; let the human choose.

---

## Phase 6 — Render the HTML report

1. Read `assets/diagnostic-report.html`.
2. Replace every `{{PLACEHOLDER}}` (CLIENT, SUBJECT, DATE, ANALYST, VERSION, INSTANCE,
   DOC_TITLE).
3. Build the sections, **localized to the client's language**:
   - §0 Context & Query + executive verdict
   - §1 Is it a bug? — the mechanism table + verdict badge
   - §2 Detail by record — one `.phase-card` per record: cause, evidence table, warning
     alert, **traceability table from chatter**, reconciliation note
   - §3 Consolidated — one row per record (cause · settled? · bug?)
   - §4 Options by need — one `.pending-item` per option
4. Pick badges/alerts that match the verdict (see template header comment).
5. Run `vx_brand.py inline` on the output file to embed the real fonts and logo
   (see Setup). Optionally run the two lint gates.

Save to: `~/Downloads/Diagnosis-<SUBJECT>-<YYYY-MM-DD>.html`. Show the local path and
offer to open it for preview before sending it anywhere.

---

## Phase 7 — Deliver (optional)

- Draft a client-facing reply for the record chatter in the client's language (no
  internal jargon), post only on explicit confirmation (`mcp__odoo__write` on
  `mail.message`/`mail.thread` or the messaging path already in use for that client).
- Uploading to Drive is out of scope for this adapted version (no Drive integration
  configured) — deliver the local HTML file directly.

---

## Gotchas

- **`invoice_status` is computed from QUANTITY, not amount.** A line can be fully paid in
  money yet show `to invoice` if the invoiced *quantity* nets to zero.
- **`qty_invoiced` only counts linked invoice lines.** An invoice line with empty
  `sale_line_ids` does **not** raise the sale line's invoiced quantity, even if it bills the
  same product. Always check the link, not just the presence of an invoice.
- **A full-quantity credit note reverses the invoiced quantity.** Used to adjust a *price*,
  it zeroes `qty_invoiced` (qty in − qty out = 0) and re-opens the line as `to invoice`,
  even though the customer was net-charged correctly.
- **Stored computed fields recompute.** Forcing `invoice_status` (or similar) via a server
  action is an override that Odoo may revert on the next recompute. Say so.
- **Amount matching ≠ quantity matching.** Reconcile both: "order total = net invoiced"
  proves collection; it does not fix the status by itself.
- **Profile safety.** Diagnosing the wrong instance is worse than not answering. Verify
  url/database before querying; pass the resolved `profile` on every call.
- **No secrets in the report.** Never paste credentials, tokens, or PII from the chatter
  into the deliverable.

---

## Usage Examples

### Example 1 — "Why are these orders 'To Invoice'? Is it a bug?"

1. Reproduce: `mcp__odoo__search_read` on `sale.order.line`, domain
   `[('order_id','in',[3565,3517]),('product_id','=',4254)]`, fields
   `id,qty_invoiced,qty_to_invoice,invoice_status,invoice_lines`, profile `stiloconcepto`
   — the line shows `qty_invoiced` 0.
2. Inspect the linked move lines (invoice + credit note): `mcp__odoo__search_read` on
   `account.move.line`, domain `[('id','in',[118321,160961])]`, fields
   `move_name,move_type,parent_state,quantity,price_subtotal,sale_line_ids`.
3. Read the chatter for the "why": `mcp__odoo__search_read` on `mail.message`, domain
   `[('model','=','sale.order'),('res_id','=',3565),('message_type','in',['comment','email'])]`,
   fields `date,author_id,body`, order `date asc`.

Verdict: **expected behavior** — a full-quantity credit note (price adjustment / refund)
zeroed `qty_invoiced`; the order is fully collected. Options: server-action override (with
recompute caveat) or re-link/root-cause fix. Never re-invoice (double billing).

### Example 2 — Single record by model/ids

`mcp__odoo__search_read` on `stock.picking`, domain `[('id','=',<id>)]`, fields
`name,state,move_ids_without_package`, profile `<profile>`.

---

## Business rules

- **Read-only by default** — never modify records; present options for the human to choose.
- **Reproduce before concluding** — no verdict without the value explained from data.
- **Use the chatter as evidence** — a documented decision beats an assertion.
- **Skeptical of "bug"** — confirmed bug needs a reproduction + standard/code reference.
- **Localize the deliverable** — report in the client's language; SKILL stays English.
- **Preview before sending** — never post to chatter without confirmation.
