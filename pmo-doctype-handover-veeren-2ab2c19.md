# PMO DocType Handover — for Veeren's AI Team (commit `2ab2c19`)

**Date:** 26 Aug 2026
**Why this doc exists:** four schema-level changes landed on `main` outside the regular T00–T20 task list — owner-raised additions (the "Expert's Advice" track, ET-01 through ET-04), not sourced from the Excel/PDF scope. Veeren's team should know about these before planning against `scale_project_management`'s schema, especially the skill-data change below, which is directly relevant to V03 (AI Resource Optimization).

**Status of the underlying code:** commit **[`2ab2c19`](https://github.com/ehiddenbrain/scale/commit/2ab2c19)** — **pushed to `main`** (26 Aug 2026, direct push, no PR — a single squashed commit covering 4 small features built the same day). Everything below is live on `main` — pull latest and it's there, no branch switching needed.

**This is the first of a series.** More commits will come with their own handover doc as they land — don't assume this is the last one for the PMO module.

---

## 1. Modified native DocType: `Project`

3 new Custom Fields:

| Field | Type | Notes |
|---|---|---|
| `project_manager` | Link → Employee | **Not required.** Picker is filtered to Employees whose Designation is "Project Manager"; same rule re-checked server-side (`project_hooks.enforce_project_manager_designation`) so it can't be bypassed via API/import. Started life as a required field, then corrected the same day — most existing Projects don't have one set, don't assume every Project has a `project_manager` value. |
| `project_manager_name` | Data, read-only | Fetched from `project_manager.employee_name`. Purely cosmetic — don't read this for logic, read `project_manager` and resolve the Employee yourself if you need more than the name. |
| `project_risk_summary` | Text | Labeled "Key Risks Summary" on the form (deliberately not "Project Risk", to avoid confusion with the doctype below). Free-text narrative, capped at 500 words (`project_hooks.enforce_risk_summary_word_limit`). **Not structured data** — if you need per-risk signals (probability/impact/category) for any AI feature, use the `Project Risk` doctype instead, not this field. This is a summary a human wrote, nothing to parse reliably.

Also: `override_doctype_dashboards` now adds a "Risk" group to Project's Connections sidebar, linking to `Project Risk`. Chained on top of `hrms`'s existing override (which adds "Claims") — no doctype/schema impact, just a desk UI convenience. Mentioned here only so you're not surprised if you see `project_hooks.add_risk_to_project_dashboard` in the codebase and wonder what it's for.

---

## 2. Modified native child DocType: `Project User` (Project's `users` table)

1 new Custom Field:

| Field | Type | Notes |
|---|---|---|
| `major_skill` | Link → Employee Skill Map, read-only | Auto-set on every Project save (`project_hooks.link_major_skill_for_project_users`): matches the row's `user` to an Employee via `user_id`, then links to that Employee's Employee Skill Map. |

**Relevant to your track — read this carefully if V03 (AI Resource Optimization) uses skill data:**

- **An empty Employee Skill Map is now auto-created** the first time an Employee joins a project's team, if they don't already have one. This means `Employee Skill Map` record *count* is no longer a clean signal of "how many people have documented skills" — most will be empty shells with zero rows in `employee_skills`. Check `employee_skills` (the child table), not just record existence.
- Don't assume every `Project User` row has a `major_skill` value — it's blank for anyone whose `user` doesn't resolve to an Employee at all (external/portal users).

---

## 3. Modified native DocType (HRMS): `Employee Skill Map`

1 new Custom Field, plus a **global title change** you need to know about:

| Field | Type | Notes |
|---|---|---|
| `top_skill` | Data, read-only | Auto-computed on every save (`skill_hooks.compute_top_skill`): the skill with the highest `proficiency` rating in `employee_skills`, formatted `"<skill> (<stars>/5)"`. `"No skills added yet"` if the map is empty. |

**Global side effect, done deliberately (owner chose this over a narrower option after seeing both mocked up):** two Property Setters make `top_skill` this doctype's `title_field`, with `show_title_field_in_link` turned on. **This means `Employee Skill Map`'s displayed title — everywhere it's referenced across the whole bench, including its own page title/breadcrumb/list view, and any Link field pointing to it — now shows the computed skill text, not the employee's name.** If any of your AI tooling displays, logs, or links to an `Employee Skill Map` record by its default title, expect skill text like `"Python (4/5)"`, not `"Rohan Kapoor"`. If you need the employee's name, read the `employee_name` field directly, don't rely on the record's title/label.

**Real, useful data source for V03:** `top_skill` (and the full `employee_skills` child table behind it) is the first place in this codebase where an Employee's skills are tied back to Project staffing at all — worth considering as an input signal for resource-matching/optimization, same way T15's `nps_score` was flagged as an unclaimed signal in the earlier handover doc. Nothing currently reads this data for AI purposes; it's clean and available.

**Caveat, same as ET-02's own task note:** only a handful of employees on this bench have any real skill data entered — most `Employee Skill Map` records that exist today (including the newly auto-created ones) are empty. Don't build against this expecting rich, populated data yet; it's a data-entry gap, not a schema gap.

---

## Open items, unchanged by this commit

- The "Business definition of impact" question from the earlier handover doc is still open — this commit doesn't touch the Project Impact Report.
- Per-project access control (restricting who can edit a Project to its actual `project_manager`) is explicitly **not** built — the field is informational only, access is still governed by the blanket `Projects Manager`/`Projects User` roles site-wide. Flagged in the code comments if you ever need to reason about who "owns" a project programmatically.

---

**Full technical detail** (exact hook names, the property-setter chaining mechanics, the auto-create decision history) is in `pmo-module.html`'s ET-01 through ET-04 task cards if you need more than this summary.
