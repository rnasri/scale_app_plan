# Claude / Implementation

This folder contains development artifacts produced during the AIWS ERP implementation program:
scripts, configuration files, code snippets, migration tools, and working notes that do not belong in the portal or implementation sub-site.

## What goes here

| Subfolder | Contents |
|---|---|
| `frappe/` | Custom app skeletons, bench setup scripts, fixture files, patch scripts |
| `control-plane/` | FastAPI service stubs, DB migration scripts, provisioner logic |
| `infra/` | Terraform/CDK, Kubernetes manifests, Dockerfiles, CI/CD configs |
| `data/` | Data migration helpers, import templates, seed data |
| `scripts/` | One-off utility scripts used during implementation |

## Governing documents

Before writing any code, read:
- `../../AIWS_CONTEXT.md` — decisions log and scope boundary (source of truth)
- `../../implementation/` — the engineering blueprint (architecture, DB, infra, roadmap)
- `../../AIWS_ERP_MODULES_CONTEXT.md` — module requirements and feature coverage matrix
- Memory files in Claude Code memory (session context, WIP tracker, working rules)

## Standing rules

1. **Never touch Frappe core** — all customizations go in `aiws_core`, `aiws_align`, or `aiws_erp` custom apps only.
2. **Bridge/Brain deferred** — no AIB or Strategy Engine integration work here; only seam-keeping (Config Profile + event outbox).
3. **AWS ap-south-1** is the cloud target; use managed services (RDS/ElastiCache/EKS/S3/MSK).
4. **MariaDB per Frappe site** for tenant data; **PostgreSQL** for the control plane.
5. **Team: 4–6 engineers** — module waves run serially; one platform track alongside.
