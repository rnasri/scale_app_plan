# PR #69 — Scale Service (Field Service / AMC Module)

**Code Review · Prepared for Muhammad Parkar**

**Repo:** ehiddenbrain/scale · **Branch:** `feature/scale_services` → `main` · **Status:** Open, marked `CONFLICTING` on GitHub

This is a plain-language write-up of a review focused on one specific question: **does anything in this PR rebuild something Scale/ERPNext already has, instead of extending it?** All 32 new DocTypes in the `scale_service` app were read field-by-field and checked against the native Frappe/ERPNext doctypes already in this bench, and against how this same codebase already solves the same problem elsewhere (e.g. `scale_purchase`'s use of Material Request, `scale_sales`'s Quotation workflow). Every finding below cites the actual native fields found in the installed ERPNext source, not a general impression.

The module itself is a real, substantial piece of work — a genuine field-service/AMC business capability ERPNext doesn't natively cover well — and one part of it (repairable spare tracking) is a good example of extending rather than duplicating. The issue isn't the module's existence, it's that roughly a fifth of the new DocTypes rebuild something already in the foundation.

## Summary

| Total new DocTypes reviewed | Duplicate/overlaps a native DocType | Legitimately new | Other flags |
|---|---|---|---|
| 32 | 6 | 24 | 1 naming/origin concern, 1 likely scaffold, 1 merge conflict |

| Finding | Duplicates | Overlap strength |
|---|---|---|
| 1. Installed Base / Installed Base Unit | `Serial No` | High — same purpose, same fields, re-implemented |
| 2. Service Contract Billing Schedule | `Subscription` | High — same purpose (recurring billing), hand-rolled |
| 3. Service Team / Service Team Member | `Asset Maintenance Team` / `Maintenance Team Member` | High — near-identical shape |
| 4. Service Estimate | `Quotation` | Medium — same lifecycle, different object |
| 5. Service Item Request / Line | `Material Request` | Medium — same purpose, already the house standard |
| 6. Equipment Schedule / PMS Checklist Item | `Maintenance Schedule` / `Maintenance Schedule Item` | Medium — real gap exists (Contract vs. Sales Order), base concept overlaps |

---

## 🔴 Finding 1 — Installed Base / Installed Base Unit rebuilds native Serial No

**Overlap: High** · `scale_service/doctype/installed_base` and `installed_base_unit`

**What's happening**

`Installed Base` (Customer, Item, model, `serial_no` as a plain **Data** field, `warranty_start_date`, `warranty_expiry`, `commissioning_date`, `purchased_from`) and its child `Installed Base Unit` (unit_type, tag_no, model, `serial_no` again as Data, capacity, refrigerant, location) together model exactly one thing: *the AMC/warranty status of a specific serialized unit sitting at a specific customer's site.*

ERPNext's native `Serial No` doctype already exists for exactly this. Checked directly in the installed source (`erpnext/stock/doctype/serial_no/serial_no.json`):

```
maintenance_status   Select: Under Warranty / Out of Warranty / Under AMC / Out of AMC
warranty_expiry_date Date
amc_expiry_date      Date
customer             Link → Customer
warehouse            Link → Warehouse
purchase_document_no Data
```

That's the same data model, with the AMC/warranty distinction already built in as a first-class status value. The new doctypes duplicate it as free text (`serial_no` is `Data`, not a `Link` to the real Serial No record), so there's no way to ever join a Service Call back to the item's real stock/purchase/warranty history — the two systems can't talk to each other.

**Why it matters**

Every other module in this codebase (Stock, Purchase, Manufacturing) already treats Serial No as the source of truth for a unit's identity and history. A Service Call raised against an `Installed Base Unit` has no queryable link to that unit's actual purchase record, its batch, or any stock movement — the two worlds are permanently disconnected unless someone manually keeps the free-text `serial_no` fields in sync by hand.

**Suggested fix**

Make `Installed Base Unit.serial_no` a `Link` to `Serial No` instead of `Data`. Move `warranty_start_date`/`warranty_expiry`/`amc_owner` onto `Serial No` as custom fields (prefixed `service_`, per the project's `scale_`-field convention) instead of a parallel table, and drive `Serial No.maintenance_status` from the AMC contract state instead of maintaining a separate status on `Installed Base`. `Installed Base` itself can stay — it's a legitimate "site + customer" grouping concept Serial No doesn't have — but its units should point at real Serial No records, not re-describe them.

---

## 🔴 Finding 2 — Service Contract Billing Schedule rebuilds native Subscription

**Overlap: High** · `scale_service/doctype/service_contract_billing_schedule`

**What's happening**

This child table (`period_no`, `from_date`, `to_date`, `invoice_date`, `amount`, `status: Pending/Invoiced/Suspended`, `sales_invoice` link) is a hand-built recurring-billing schedule for AMC contracts.

ERPNext already ships `Subscription`/`Subscription Plan` — a recurring-billing engine with its own period tracking (`current_invoice_start`/`current_invoice_end`, `days_until_due`, `cancel_at_period_end`) and automatic Sales Invoice generation on schedule.

**Why it matters**

Two independent recurring-billing systems now exist on the same bench. Any future improvement to invoice generation, proration, or dunning has to be built (and tested) twice, and Finance-side reporting that assumes `Subscription` is the one place recurring revenue lives will silently miss every AMC contract.

**Suggested fix**

Evaluate whether `Subscription` (possibly with a custom `Subscription Plan` per AMC billing frequency — Annual/Quarterly/Monthly already matches `Service Contract`'s own `billing_schedule` Select options almost 1:1) can drive AMC invoicing directly, with `Service Contract` linking to a `Subscription` instead of carrying its own schedule table. If Subscription's period/proration model genuinely can't express AMC-specific billing, that's a reasonable finding — but it should be a documented decision, not a default.

---

## 🟡 Finding 3 — Service Team / Service Team Member duplicates Asset Maintenance Team

**Overlap: High** · `scale_service/doctype/service_team`, `service_team_member`

**What's happening**

`Service Team` (team_name, lead_engineer, members table) and `Service Team Member` are structurally almost identical to ERPNext's native `Asset Maintenance Team` (`maintenance_team_name`, `maintenance_manager`, `maintenance_team_members` table) and `Maintenance Team Member` (`team_member`, `full_name`, `maintenance_role`).

**Why it matters**

Lower stakes than Findings 1–2 (no cross-module data-integrity risk), but it's still a second "team of people who do maintenance work" concept on the same bench, with its own permissions and its own place to look someone up.

**Suggested fix**

Either rename/reuse `Asset Maintenance Team` directly with a `service_zone`/`max_calls_per_day` custom field for the Field-Service-specific bits, or — if a separate doctype is kept — document explicitly why (e.g. Asset Maintenance Team is scoped to internally-owned Assets, not customer-site equipment, if that distinction turns out to matter).

---

## 🟡 Finding 4 — Service Estimate re-implements Quotation's lifecycle

**Overlap: Medium** · `scale_service/doctype/service_estimate`, `service_line_item`

**What's happening**

`Service Estimate` has the exact same shape and status lifecycle as native `Quotation`: Customer, dated, line items, totals, `Draft → Submitted → Sent To Customer → Accepted/Rejected/Cancelled`. `scale_sales` in this same codebase already has a mature Quotation-based approval workflow (discount thresholds, Manager/VP approval — see `sales-tracker.html`'s T08 finding for the existing design).

**Why it matters**

A second "customer-facing priced document with an accept/reject flow" means Sales reporting, discount-approval rules, and any future Quotation-level automation don't apply here automatically — someone has to remember to build (and maintain) each one twice.

**Suggested fix**

Consider a `Quotation` with a custom `Service Call` link field and a `quotation_type` or similar flag, reusing the existing approval workflow, instead of a parallel document type. If Service Estimate's chargeable/free-line split (`total_free`/`total_chargeable`/`has_free_exception`) doesn't map cleanly onto Quotation Item, that's a real reason to diverge — but worth confirming rather than assuming.

---

## 🟡 Finding 5 — Service Item Request duplicates Material Request

**Overlap: Medium** · `scale_service/doctype/service_item_request`, `service_item_request_line`

**What's happening**

Warehouse, requested items, `Draft → Approved → Issued → Returned → Cancelled` — this is Material Request's job, and Material Request is already the house standard: `scale_purchase` and `scale_stock` both build on it extensively elsewhere in this codebase (Material Request's own PO-approval and MRP flows are the very mechanism a previous connected-demo build in this project deliberately exercised).

**Why it matters**

Spares consumed on a Service Call won't show up in any of the existing Material Request-driven reporting (procurement planning, stock shortfall visibility) that the rest of the ERP already relies on for exactly this kind of "who requested what and when" tracking.

**Suggested fix**

Use `Material Request` with `material_request_type` or a custom field tagging it as service-originated, linked back to the `Service Call`. This also gets AMC spare requests into the same MRP/procurement pipeline PUR-13 and MFG-15 already established, for free.

---

## 🟢 Finding 6 — Equipment Schedule / PMS Checklist Item partially overlaps Maintenance Schedule

**Overlap: Medium, real gap also exists** · `scale_service/doctype/equipment_schedule`, `pms_checklist_item`

**What's happening**

Native `Maintenance Schedule`/`Maintenance Schedule Item` already models "scheduled preventive visits against equipment with due dates," but it's tied to a Sales Order, not a standalone Contract — so there's a genuine structural gap here, unlike Findings 1–5. This one is lower-confidence as a "just reuse the native one" case.

**Suggested fix**

Worth a design conversation rather than an assumption either way: can `Maintenance Schedule` be generated from a `Service Contract` instead of Sales Order, or is a Contract-anchored schedule different enough to justify staying separate? Flagging for a decision, not asserting the native one is a drop-in fit.

---

## Also worth flagging — not duplication, but real

### `Service Call`'s fields look ported wholesale from an external legacy system

`Service Call`'s field list (`companyid`, `customerid`, `numberingseriesmasterid`, `isdeleted`, `whatsappno`, and one field literally named `inswizarddeliverynote` → labeled **"Installation Wizard Delivery Note"**) doesn't follow Frappe/Scale's own naming conventions anywhere else in this codebase (snake_case business names, `scale_`-prefixed custom fields, no raw DB-style `xxxid` suffixes). This reads like a schema ported directly from an existing external AMC/service-management tool (the field name names it: "Installation Wizard") rather than designed fresh against what Scale/ERPNext already offers — which is consistent with the duplication findings above. Worth confirming with Muhammad whether this was a deliberate client-data-migration shortcut (reasonable, if flagged) or an unreviewed carry-over.

### `Project Demo Machine` — likely scaffolding, not a real feature

A single-field doctype (`item_code` only), named "demo," matching commit `4832cc1 scale_service: TEMP DEMO - prefill address/machinery from Project`. Worth confirming with Muhammad whether this is meant to ship or should be dropped before merge.

### Merge conflict

GitHub marks this PR `CONFLICTING`. Traced directly (via `git merge-tree`): one conflict, in `apps/scale/scale/hooks.py`'s `app_include_js` list — both this branch and already-merged work added new entries to the same list around the same spot. Trivial textual conflict, not semantic — just needs a rebase against current `main` before it can merge.

---

## Suggested next step

Not a blocker list to clear item-by-item — more a design conversation before merge: for Findings 1, 2, and 5 especially (Serial No, Subscription, Material Request), extending the native doctype is very likely the right call given how central those three already are elsewhere in this codebase. Findings 3 and 4 are lower-stakes but worth a quick decision either way. Finding 6 genuinely needs a real design call, not a default answer. None of this is a rewrite — most of it is redirecting a few `Link`/foreign-key fields at the right target before the module goes further.
