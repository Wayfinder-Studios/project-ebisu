# Project Ebisu

> **Campus Merch Platform** — Multi-tenant e-commerce and operational logistics platform tailored for university student organizations, batch pre-orders, flash drops, and fast booth distribution.

---

## Overview

Project Ebisu addresses the specific operational frictions faced by university councils, student organizations, and campus merchants when running merchandise campaigns:
- **Zero Platform Escrow:** Direct-to-merchant payments (GCash, Maya, Bank Transfer, Walk-in Cash) with proof verification desk.
- **Batch Pre-Orders & MOQ Safeguards:** Enforce Minimum Order Quantities with automated deadlines, 48-hour Treasurer deficit override windows, and refund queues.
- **Toll-Gate & Master Supplier Export:** Formula-sanitized tailor production spreadsheets unlocked upon platform fee settlement.
- **Booth Dispatch Terminal:** Touchscreen-friendly, sub-second alphanumeric Pick-up Code lookups (`CS-8419`) with double-claim prevention.
- **Daily Cash Registers:** Multi-officer cash drawer tracking with end-of-day dual-sign-off reconciliation.

---

## Documentation Architecture

All engineering specifications, database schemas, and meeting records live directly in this repository as the **single source of truth**:

```
docs/
├── srs/
│   ├── 01-introduction.md          # Purpose, scope, user classes, operational constraints
│   ├── 02-system-architecture.md   # Functional use cases (UC-01 - UC-20), modules, governance
│   └── 03-api-endpoints.md         # Interfaces, data requirements, decisions log
├── schemas/
│   └── supabase-schema.md          # PostgreSQL / Supabase schema, RLS policies & tables
└── moms/
    ├── template.md                 # Standard Minutes of Meeting markdown template
    └── 2026-09-10-kickoff.md       # Project kickoff meeting notes
```

---

## Tech Stack & Infrastructure

- **Frontend / Framework:** Next.js (App Router), TypeScript, TailwindCSS / Vanilla CSS
- **Backend & Database:** Supabase (PostgreSQL with Row Level Security, Triggers & Atomic Functions)
- **Object Storage:** Cloudflare R2 / AWS S3 (Receipt screenshots, product mockups)
- **Transactional Notifications:** Resend (Email magic links & status updates)

---

## Engineering & Tracking Workflow

To keep code, documentation, and task tracking strictly synchronized:

1. **Task Intake:**
   - Tasks and features are created via [GitHub Issue Forms](.github/ISSUE_TEMPLATE/task.yml).
   - Every issue references the relevant SRS section (e.g., `Section 3.2 UC-03`).
2. **Project Board Tracking:**
   - Issues route automatically to the Organization Kanban Board (`To Do` -> `In Progress` -> `In Review` -> `Done`).
   - Tagged with custom fields for **Epic / Feature**, **Priority**, and **Estimate**.
3. **Meeting Minutes (MoMs):**
   - Meeting notes are recorded in `docs/moms/YYYY-MM-DD-<topic>.md`.
   - Action items reference their tracked GitHub issues:
     ```markdown
     - [ ] @username to implement Supabase RLS policies for booth dispatch #12
     ```
4. **Automated PR Closure:**
   - When submitting Pull Requests, reference the issue in the description:
     ```markdown
     Closes #12
     ```
   - Merging the PR will automatically transition the issue to `Done`.

---

## License

Proprietary and confidential. See [LICENSE](LICENSE) for terms.
