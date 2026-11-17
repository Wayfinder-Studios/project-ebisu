# Project Ebisu

> **Campus Merch Platform** for student organizations at Visayas State University.

---

## Overview

Ebisu lets student orgs sell merch without spreadsheets and chat threads:
- **Inventory:** products with variants (size/color), stock or pre-order with deadline, plus staff-only MOQ tracking.
- **Payment history:** every GCash/Maya/bank transfer and cash sale in one place, filterable and exportable.
- **Buying:** buyers sign in with Google, browse all shops, check out with a transfer screenshot (or pay cash to staff), and pick up with a code at the booth.

Every user must be signed in; there is no guest access.

---

## Documentation

```
docs/
├── function-list.md        # V1 scope: roles, functions (EB-xxx), business rules
├── design-spec.md          # UI notes (draft)
└── moms/                   # Meeting notes
```

The database schema lives in `supabase/migrations/` once coding starts.

---

## Tech Stack & Infrastructure

- **Frontend / Framework:** Next.js (App Router), TypeScript, TailwindCSS
- **Hosting:** Vercel (Free Tier)
- **Backend & Database:** Supabase (PostgreSQL with Row Level Security, Triggers & Atomic Functions)
- **Auth:** Supabase Auth with Google OAuth only (every account starts as a Buyer)
- **Storage:** Cloudflare R2 (private bucket for payment screenshots via presigned URLs, public bucket for product and shop images)

---

## Engineering & Tracking Workflow

To keep code, documentation, and task tracking strictly synchronized:

1. **Task Intake:**
   - Tasks and features are created via [GitHub Issue Forms](.github/ISSUE_TEMPLATE/task.yml).
   - Every issue references the function ID it implements (e.g., `EB-007`).
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
