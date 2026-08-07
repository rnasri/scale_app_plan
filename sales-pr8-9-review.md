# Sales PRs #8 & #9 — Review Context (PENDING RE-REVIEW)

**Status:** Review done, feedback sent to both authors, **waiting on their updates before re-review/approval.**
**Date reviewed:** 2026-08-06
**Reviewer:** Claude (this session), at Raghib's request — "review for sales integration, is it safe, nothing will break, aligned with task list, nothing misimplemented, can I approve?"
**Repo:** `ehiddenbrain/scale`

## How to resume this thread in a new session

1. Re-fetch both PRs to see if they've changed:
   ```
   gh pr view https://github.com/ehiddenbrain/scale/pull/9 --json title,body,state,headRefName,updatedAt
   gh pr view https://github.com/ehiddenbrain/scale/pull/8 --json title,body,state,headRefName,updatedAt
   gh pr diff https://github.com/ehiddenbrain/scale/pull/9
   gh pr diff https://github.com/ehiddenbrain/scale/pull/8
   ```
2. Check whether the specific issues below were addressed (diff the new head against what's summarized here).
3. **Re-run the empirical merge-conflict test** (this is the critical check — don't skip it just because individual diffs look fixed):
   ```
   cd Claude/code/scale-bench
   git fetch origin refs/pull/9/head:pr9-scratch refs/pull/8/head:pr8-scratch
   git checkout -b integration-test-scratch main
   git merge pr9-scratch --no-edit
   git merge pr8-scratch --no-edit   # look for CONFLICT lines
   git diff --name-only --diff-filter=U   # lists any still-conflicting files
   # cleanup after, regardless of outcome:
   git merge --abort   # only if mid-conflict
   git checkout main
   git branch -D integration-test-scratch pr9-scratch pr8-scratch
   ```
4. If clean, do a final read-through against the "Issues found" lists below, then give Raghib an updated approve/hold verdict.

---

## PR #9 — Archit (`archit-jais`), branch `sales-verify`, title "scale sales-archit"

**What it does:** Customer Risk Automation (T18) — new `Sales Config` Single DocType (risk rule toggles/thresholds + inert AI-threshold fields for a future AI team integration), `customer_risk.py` rule engine (overdue invoice / competitor signal / no-activity checks, respects manual overrides, auto-clear), wired into Sales Invoice + Payment Entry doc events + a daily scheduled job. Also adds 3 "Lost Deal" fields to Opportunity (Loss Category, Competitor Identified, AI Loss Summary) per SALES-15.

**Verified correct:**
- `customer_risk.py` rule logic — sound, SQL properly parameterized, guards against overwriting Churned/Inactive status.
- Idempotency guards throughout `demo_setup.py` additions.

**Issues found (sent to Archit):**
1. **Opportunity Lost-Deal fields appear unwired** — 3 new fields shipped, but no code in this diff (`_run_lost_analysis()` or equivalent) actually populates them, and nothing added to `opportunity_hooks.py`'s `doc_events`. Asked Archit to confirm whether this lands in a follow-up or was missed.
2. `on_payment_received` calls `sync_collection_status_for_invoice()` directly **and** enqueues the same function again with `now=True` right after — double-runs per invoice on every payment. Harmless (idempotent) but looks like leftover debug code.
3. `debug_customer_risk.py` and `diagnose_risk_fields.py` — personal debug scripts with hardcoded invoice numbers from Archit's local site (`SINV-26-00009`, `SINV-26-00006`). Asked him to drop these before merge.
4. New root-level `Agents.md` (115 lines) duplicates the purpose of the existing `docs/SCALE_TEAM_DEVELOPMENT_GUIDE.md` / `docs/SCALE_DOMAIN_APP_PLAYBOOK.md`. Risk of two competing "rules" docs. Asked him to drop it or fold anything new into the existing guide.
5. `_make_room_after_customer_details()` / `_fix_customer_field_collisions()` reorder Customer form fields via raw SQL on `tabDocField`/`tabCustom Field` idx columns — works, not wired into the default install path (`run_tier2`) so not a live risk, but non-standard vs. normal Frappe `insert_after` convention. Flagged as a pattern not to repeat, not a blocker.

## PR #8 — Muhammad (`muhammadparkar`), branch `feature/sales-02-muhammad`, title "Feature/sales 02 muhammad"

**What it does:** Distributor Onboarding DocType (SALES-07) — full application pipeline (Draft → Documents Pending → Under Review → Approved/Rejected → Active); on Approve + linked Customer, sets Customer Group = Distributor (verified in code). Also T19 — `collection_tasks.py` daily job: T-7/T-0 email reminders + auto-creates/submits a Dunning document at T+15 via native `create_dunning()`. Plus Notification rules and Net 30/Net 60 Payment Terms Templates.

**Verified correct:**
- Distributor → Customer Group logic matches PR description exactly.
- `create_dunning(invoice.name, ignore_permissions=True)` call checked directly against the real signature in `erpnext/accounts/doctype/sales_invoice/sales_invoice.py` (`create_dunning(source_name, target_doc=None, ignore_permissions=False)`) — matches, not a bug.
- Also independently fixes the `setup_dunning_types()` crash-on-missing-Company bug (same issue raised earlier this session from Ahmad's teammate's report) — this PR closes that gap.

**Issues found (sent to Muhammad):**
1. `daily_collection_reminders()` catches and logs every per-invoice failure via `frappe.log_error` and continues — good for isolation, but a systemic failure (e.g. email server down) would fail silently for every invoice, every day, with nothing visible except buried error logs. Flagged as a follow-up, not urgent.
2. Branch predates the Quotation Discount Approval workflow fix (commit `c7a323c`, pushed earlier this session) — doesn't touch the same lines so won't undo the fix on merge, but his own local testing is against the old buggy version. Asked him to `git pull origin main` before further testing.

---

## THE CRITICAL FINDING — both PRs conflict with each other

Empirically verified via real `git merge` (not inferred from reading diffs): merging both PRs together — in either order — produces real conflicts in **5 files**:
```
apps/scale_sales/TESTING.md
apps/scale_sales/scale_sales/collection_hooks.py
apps/scale_sales/scale_sales/demo_setup.py
apps/scale_sales/scale_sales/hooks.py
apps/scale_sales/scale_sales/public/js/demo_ui.js
```

**Root cause:** both Archit and Muhammad independently built "Partial payment status" handling into `collection_hooks.py::on_payment_received()` as a side effect of their separate actual tasks (T18 and T19 respectively) — same idea, two different implementations, off the same starting code. Also both added `"Partial"` to `scale_sales_collection_status`'s Select options in `demo_setup.py` with identical text.

- PR #9's version: pulled into a reusable `sync_collection_status_for_invoice()` helper, wrapped in `frappe.db.after_commit(...)`, plus a new `on_invoice_update()` hook.
- PR #8's version: smaller, self-contained, same Partial/Collected logic written inline.

**Recommendation given to both:** they need to coordinate directly on whose `collection_hooks.py` version survives (leaning toward Archit's, since it's the reusable one), then whoever's second should **rebase on top of the agreed version**, not blind-resolve the textual conflict when merging.

---

## Messages already sent (2026-08-06, via WhatsApp)

Both Archit and Muhammad were sent full, specific writeups (in plain language) covering their own PR's issues above **and** the cross-PR conflict with each other — asking them to sync directly on the `collection_hooks.py` question. Per Raghib's choice, the messages **flagged the conflict without an explicit "don't merge yet" instruction** — timing was left to their judgment.

## What's blocking approval right now

Nothing is approved. Waiting on:
1. Archit ↔ Muhammad to agree on and reconcile the `collection_hooks.py`/Partial-status overlap.
2. Archit: confirm/fix the unwired Lost-Deal fields, drop the debug files + `Agents.md` (or justify keeping them).
3. Muhammad: pull `main`, re-test with the discount-workflow fix in place.
4. Re-run the empirical merge test (see "How to resume" above) once both are updated — that's the actual pass/fail gate, not just re-reading the diffs.
