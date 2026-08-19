# PMO DocType Handover — for Veeren's AI Team

**Date:** 19 Aug 2026
**Why this doc exists:** Veeren's team is building AI features (V01–V03: Delay Prediction, Resource Optimization, Budget Forecasting) against `scale_project_management` as it stood *before* PR #34. This doc lists exactly what changed at the DocType/schema level in that PR, so the AI track's plan can account for it.

**Status of the underlying code:** `PMO_T15_T17_impact_library` branch, PR **[#34](https://github.com/ehiddenbrain/scale/pull/34)** against `main` — **pushed, not yet merged.** Everything below exists on that branch; if you build against current `main` today, none of it is there yet. Confirm with Raghib whether to build against the PR branch directly or wait for merge.

---

## 1. New DocType: `Project Stakeholder Survey`

Captures client NPS feedback at project close. Fully new, no existing doctype affected.

| Field | Type | Notes |
|---|---|---|
| `project` | Link → Project | Required |
| `respondent_name` | Data | Optional (web form submissions are often anonymous) |
| `respondent_email` | Data (Email) | Optional |
| `survey_date` | Date | Defaults to today |
| `nps_score` | Int | **0–10**, required. Industry-standard NPS scale — the source Excel row named "NPS" with no scale of its own |
| `nps_category` | Select | `Promoter` / `Passive` / `Detractor` — **computed automatically** in the doctype's `validate()`, not user-set: 9–10 → Promoter, 7–8 → Passive, 0–6 → Detractor |
| `feedback` | Text | Free-text comments |

**How records get created:**
- Directly on the desk (Projects Manager / PMO Director), or
- Through a new public web form at `/project-nps-survey` (no login required, `doc_type: Project Stakeholder Survey`) — this is what the automated email below links to.

**Relevant to your track:** this is the real data source behind PMO-20's "Customer Satisfaction Score" tile (see §3) and could be a useful additional signal for an "AI Project Health Score" if you want client sentiment folded in alongside your delay/budget/resource scores — nothing currently reads `nps_score` from any AI code, it's a clean, unclaimed data source.

---

## 2. Modified native DocType: `File`

4 new Custom Fields added to Frappe's native `File` doctype (not a new doctype — File already exists, this just extends it):

| Field | Type | Notes |
|---|---|---|
| `pmo_project_tag` | Link → Project | Which project this file is relevant to — independent of the file's actual `attached_to_doctype`/`attached_to_name` (a file can be attached to a Task but tagged to the parent Project) |
| `pmo_industry_tag` | Data | Free text, no master list |
| `pmo_technology_tag` | Data | Free text, no master list |
| `pmo_is_confidential` | Check | See permission model below |

**New permission behavior on File** (relevant if any AI tool ever reads File records): a File with `pmo_is_confidential=1` is only readable by that Project's own team (Project User table) or a Projects Manager / PMO Director — enforced via `has_permission`/`permission_query_conditions` hooks in `library_hooks.py`, additive on top of native File permissions. **If you write any FAC tool or AI script that queries `File` directly, use `frappe.get_list` (permission-checked), not `frappe.get_all` (bypasses permissions) — otherwise a confidential file could leak into an AI tool's output.**

---

## 3. New Report: `PMO Project Impact Report`

Query Report, `ref_doctype: Project`. Per-project rollup — not a new doctype, but a new *data surface* worth knowing about:

| Column | Source |
|---|---|
| Project / Project Name / Status | Native Project fields |
| Goal (Success Criteria) | Native `Project.success_criteria` |
| Realized Risks | Count of `Project Risk` rows for this project |
| Budget Planned / Budget Actual | Sum from `Project Budget` |
| Avg NPS Score | Average of `Project Stakeholder Survey.nps_score` for this project |
| NPS Responses | Count of survey rows for this project |

## 4. Executive Dashboard change

`executive_dashboard_hooks.get_executive_dashboard_data()` (whitelisted method backing the PMO Executive Dashboard page) now returns a live `customer_satisfaction_score` key — portfolio-wide average NPS across all `Project Stakeholder Survey` records. It was previously in that method's `pending` list with the note *"Depends on T15 — not built yet."* That dependency is now resolved.

---

## Open item, unchanged by this PR

Excel's own note on the Project Impact Report row still stands: **"Business definition of 'impact' still requested."** The report above shows the 4 available signals (goals, risks, budget, NPS) side by side — there is no single combined "impact score." If your V05 AI Project Health Score work wants to define one, that's still an open business question, not something this PR resolved either way.

---

**Full technical detail** (exact hook wiring, permission-contract bug caught during testing, print format, web form) is in `pmo-module.html`'s T15 and T17 task cards if you need more than this summary.
