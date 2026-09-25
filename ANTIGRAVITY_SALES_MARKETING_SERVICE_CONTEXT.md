# Scale — Sales, Marketing & Service — Feature Context for Antigravity

Self-contained handoff brief on three modules of **Scale** (the ERP suite built on Frappe/ERPNext by
Pixaflip × SIL). Written from the actual code in `Claude/code/scale-bench/apps/` — every feature below
was verified against the real source, not assumed from a spec sheet. Two important corrections are
flagged inline where a feature already exists in the codebase rather than needing to be built.

---

## 1. Marketing (`scale_sales_marketing` app)

WhatsApp is the primary marketing/engagement channel. Everything below lives under
`scale_sales_marketing/scale_sales_marketing/whatsapp/`.

### 1.1 WhatsApp Appointment Management

A full booking system layered on WhatsApp, not just a chat log:

- **Booking Settings per WhatsApp number** (`bookings.py`) — services offered, duration per service,
  working hours per day of week, holidays, custom per-service hours, slot duration, a Google Meet link,
  and per-status message templates (Confirmed / Completed / Cancelled / Rescheduled / Reminder).
- **Slot logic** — `open_dates()` / `open_slots()` compute genuinely free slots against existing bookings
  and working hours; `assert_slot_free()` guards every create/reschedule against double-booking.
- **Appointment lifecycle** (`appointments.py`, backed by `Scale WhatsApp Appointment` doctype) —
  Confirmed → Completed / Cancelled / Rescheduled, with day-view and month-view calendar APIs
  (`list_day`, `list_month`) for a staff-facing calendar UI.
- **Automatic WhatsApp notifications on every status change** — booked, completed, cancelled, rescheduled
  each fire a templated WhatsApp message. Two send paths: a plain session message if the contact messaged
  in the last 24 hours, otherwise a pre-approved WhatsApp **template** message (Meta's 24-hour session
  rule) — this fallback is handled automatically (`_notify_whatsapp`).
- Booking can be **started conversationally** — a customer typing a "booking" keyword in WhatsApp chat
  triggers a guided flow (service → date → time) via the bot automation layer (below), and can also be
  created manually by staff (`create_manual`).

### 1.2 Automated Templated Drip Campaigns

`drips.py`, backed by `Scale WhatsApp Drip` (campaign definition with an ordered list of template steps,
each with its own delay-hours) and `Scale WhatsApp Drip Enrollment` (per-contact progress state):

- **Enrollment** — a contact is enrolled automatically when they come in from a Meta lead-gen ad
  (`enroll_from_leadgen`, matched to a specific campaign via `meta_ad_id`), or manually
  (`enroll_conversation`). Falls back to a designated "default" campaign, then any active campaign, if no
  ad-specific match exists.
- **Step execution** (`process_pending`, intended to run on a schedule) — sends the next template in
  sequence once its delay has elapsed, advances `current_step`, and marks the enrollment `Completed` once
  all steps are sent.
- **Opt-out respected** — if the contact's conversation is unsubscribed, the enrollment stops immediately
  instead of sending.
- **Failure handling** — a failed step marks the enrollment `Failed` with the error captured, rather than
  retrying silently or blocking other enrollments.

### 1.3 The bot layer these two features sit on top of (context, not a separate feature)

`automations.py` is the inbound-message router: keyword-triggered opt-out/opt-in, keyword-triggered
booking start, a full node-based conversational **Flow** builder (buttons/lists/free-text input, with
booking-specific node types that plug directly into the appointment slots above), and ad-click-to-WhatsApp
automation (Meta ad referral → auto-reply → auto-enroll into a drip campaign). This is the mechanism that
makes 1.1 and 1.2 self-service instead of staff-triggered only.

**Already documented elsewhere in this repo:** `Claude/implementation/scale-marketing-module.html` and
`Claude/TECHNICAL_CONTEXT_SCALE_MARKETING.md` (multi-number routing, permission model, Meta Ads
architecture) — cross-check those before treating this section as the only source.

---

## 2. Sales (`scale_sales` app)

### 2.1 Lead → Opportunity → Customer workflow

⚠️ **Correction: this already exists, built on Frappe/ERPNext's native CRM conversion flow (Lead →
Opportunity → Customer → Quotation), not something to build from scratch.** What Scale adds on top is an
AI layer via `doc_events` hooks (`hooks.py`):

- **Lead** (`lead_hooks.py`) — blocks duplicate Leads on matching mobile/email at insert time; auto-scores
  every Lead on insert and re-scores on any BANT-field change (`scale_sales_budget`,
  `scale_sales_authority_level`, `scale_sales_need_description`, `scale_sales_expected_closure`) into an
  AI lead score + a priority/heat category (`ai_utils.score_lead` / `heat_category`).
- **Opportunity** (`opportunity_hooks.py`) — recomputes a win-probability on every save
  (`ai_utils.win_probability`, fed by days-since-last-activity computed from real
  Communication/Comment timestamps), writes a rule-based "next best action" recommendation, and — only at
  the moment status flips to Lost — runs an LLM-based lost-deal analysis (loss category, competitor
  identified, AI summary).
- **Quotation** — has its own existing approval workflow (discount-threshold-based Manager/VP approval,
  documented separately in `sales-tracker.html`'s T08 finding) that this doesn't touch.

### 2.2 Field Sales Visit — geo check-in / check-out by salesperson

⚠️ **Correction: this already exists in full, not something to add.** `Field Visit` doctype +
`field_sales_person.py` + `field_visit_history.py`:

- **Check-in / check-out with geolocation** — each visit captures a check-in and (separately) a check-out
  event, each with latitude/longitude, resolved address/street/city/state/pincode/country, a location
  name, and a timestamp.
- **Distance calculation** — a haversine-distance helper computes the straight-line distance between
  check-in and check-out points per visit, and aggregates total distance across a filtered set of visits.
- **Role-scoped access** (`field_sales_person.py`) — three-tier role model: **Sales Person** (field rep,
  sees/creates only their own visits — enforced via both `permission_query_conditions` and
  `has_permission`, not just UI hiding), **Sales Manager / Sales User** (see every salesperson's visits),
  **System Manager** overrides all. Resolution path is User → Employee → Sales Person master.
- **Field Visit Summary / History** (`get_field_visit_summary`) — a map-ready API: per-visit check-in/out
  coordinates with a distinct route color per visit (for a multi-visit route map), filterable by
  salesperson / date range / customer, with rollup stats (total visits, completed, unique customers,
  mapped visits, total distance). A Sales Person calling this is force-scoped to their own data even if
  they pass a different `sales_person` filter (`assert_summary_access`).

**Recommendation for Antigravity:** if the ask driving this doc is a UI/UX pass, the map-based Field Visit
Summary (2.2) and the day/month appointment calendar (1.1) are the two views most likely to benefit from
real visual design work — both already have complete backend APIs, no new endpoints needed.

---

## 3. Service (`scale_service` app — PR #69, Muhammad Parkar, **open, not yet merged**)

A new field-service / AMC (Annual Maintenance Contract) module — 32 new DocTypes, 49 commits, 223 files.
Not part of `main` yet; GitHub shows it `CONFLICTING` against `main` (traced to one trivial, non-semantic
line conflict in `apps/scale/scale/hooks.py`'s `app_include_js` list — just needs a rebase).

### 3.1 What it covers

- **Installed Base** — tracks serialized equipment installed at a customer site (warranty/AMC dates,
  commissioning date, model, capacity, refrigerant type for HVAC-style equipment).
- **Service Contract** with a **Billing Schedule** — AMC contracts with Annual/Quarterly/Monthly billing.
- **Service Call** — the core AMC/repair-visit ticket.
- **Service Estimate** — a priced, customer-facing document with an accept/reject lifecycle for chargeable
  repair work.
- **Service Item Request** — spares/parts consumption tracking against a Service Call.
- **Service Team** — the roster of field engineers/technicians who do the work.
- **Equipment Schedule / PMS Checklist** — preventive-maintenance scheduling against installed equipment.
- Recent commits (today, 15 Sep) add: a **Scale Service Coordinator** role, real role differentiation
  between **Field Engineer** vs **Service Coordinator**, a "Billing Mix" arc-gauge chart (ported from
  Sales' own win-rate gauge pattern), and demo-data/address-prefill polish. These are UI/role/demo-data
  changes only — they don't touch any of the doctype-level findings below.

### 3.2 Review finding — 6 of the 32 new DocTypes duplicate something already native

A field-by-field review (`Claude/implementation/pr69-scale-service-doctype-review.md`, full detail there)
found 6 of the 32 new DocTypes rebuild a native ERPNext capability instead of extending it:

| New DocType | Duplicates | Overlap |
|---|---|---|
| Installed Base / Installed Base Unit | `Serial No` | High — same purpose; `serial_no` is a free-text field, not linked to the real Serial No record, so it can't join back to real stock/purchase/warranty history |
| Service Contract Billing Schedule | `Subscription` | High — a second, independent recurring-billing engine |
| Service Team / Service Team Member | `Asset Maintenance Team` / `Maintenance Team Member` | High — near-identical shape |
| Service Estimate | `Quotation` | Medium — same priced-document lifecycle, misses Sales' existing discount-approval workflow |
| Service Item Request | `Material Request` | Medium — spares consumption won't appear in existing procurement/MRP reporting |
| Equipment Schedule / PMS Checklist | `Maintenance Schedule` | Medium — a real structural gap also exists here (Contract-anchored vs. native's Sales-Order-anchored), not a clean duplicate call |

Also flagged (not duplication): `Service Call`'s field names (`companyid`, `isdeleted`,
`inswizarddeliverynote`) look ported wholesale from an external legacy AMC tool rather than designed fresh
against this codebase's own naming conventions; and a single-field `Project Demo Machine` doctype looks
like leftover scaffolding from a "TEMP DEMO" commit.

**Status:** review shared with Muhammad, not yet addressed in the branch — none of today's new commits
touch the 6 findings above. Not a blocker list to clear item-by-item; more a design conversation before
merge (see the full doc for the suggested fix per finding).

---

*Sources: direct reads of `scale_sales_marketing/whatsapp/{appointments,bookings,drips,automations}.py`,
`scale_sales/{lead_hooks,opportunity_hooks,field_sales_person,field_visit_history}.py`, `scale_sales/hooks.py`,
and `Claude/implementation/pr69-scale-service-doctype-review.md`, plus `gh pr view 69` for latest commit
state, as of 2026-09-15.*
