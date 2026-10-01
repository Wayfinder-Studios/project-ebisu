# Design Specification

*(Draft in progress — D-05 stack note: v1 tokens and components assume TailwindCSS.)*

## v2.9 — Officer-Only F2F Cash Terminal (D-06/D-09)

Stack: Next.js App Router + TS + Tailwind on Vercel; Supabase PG + RLS; R2-only; Supabase Auth via Google SMTP. Buyer checkout is digital-only; all cash via this officer terminal.

### 1. Cash Terminal screen (Org Portal, mobile-first, `/org/cash`)

Header: org + booth `session_name` + `OPEN` badge + `Expected Cash` live total + booth `payment_instructions` (location/hours). Block entire form when no `OPEN` session with "Open a register session first" CTA.

Form (single page, unified preorder/flash — D-09):
1. Campaign picker (only `LIVE`, same org; shows `PRE_ORDER`/`FLASH` badge + MOQ progress or stock left).
2. Variant picker (only `is_active`; flash sold-out disabled with "Sold out" tag) + qty stepper + unit price (read-only, server-frozen) + running total.
3. Personalization fields per campaign schema + student-ID input when `require_student_id`.
4. Buyer fields: full name (required), email (required, format-validated, duplicate reuses account), mobile (required). Inline errors; email explain: "Tracker + claim pass sent here."
5. Tendered cash display + change due (client-computed, server ignores).
6. Submit `Log Cash Sale (Verified)` → calls `create_cash_order()` → success panel with big `pickup_code`, order total, buyer email, "Resend tracker email" + "New sale" (autofocus reset). Failure panel with specific reason (no OPEN session / sold out / cross-org / missing email) and "cash handed back" guidance.

States: loading (disabled submit + spinner), empty (no LIVE campaigns / no OPEN session), permission-denied (non-officer role), offline (block submit, "terminal is online-only"), uncertain-result (after timeout: "Do not retake cash — re-check order by pickup code/buyer email before retrying"; idempotency via buyer-email + variant + qty re-query).

A11y/perf: touch targets ≥44px, autofocus campaign→variant→email flow, labels for all inputs, status not by color alone, sub-second pickup-code render after commit.

### 2. Buyer checkout change (D-06)

Remove cash radio, booth copy, 24h TTL text. Digital form unchanged (`Source Channel + reference + R2 receipt`). Cash buyers see booth poster/QR directing to F2F terminal instead.

### 3. Tracker email template (REQ-22A, Google SMTP via Supabase Auth)

Subject: `[Ebisu] Your order {pickup_code} is verified — {campaign_title}`

Body:
- Greeting `{buyer_name}`, org + campaign + variant/qty + total paid (cash) + `pickup_code` (large).
- Magic-link button `Track my order` (expiry/replay per auth config) + plain URL fallback.
- "Show this code + school ID at pickup. We'll email when Ready for Pick-up."
- Footer: booth contact, receipt note ("Cash collected F2F under {session_name}"), no-reply guidance.

Send: queued post-commit (not in DB txn). On SMTP fail: log + show "email pending" on officer success panel with retry button; never roll back the verified sale (D-08).

### 4. v2.9 acceptance checklist (maps to DEV-02/03/08/11)

- [ ] Buyer cannot `INSERT CASH_ON_HAND` (RLS negative test) and sees no cash option.
- [ ] Officer preorder cash → `PAYMENT_VERIFIED` + MOQ count +1×qty + email queued + history row.
- [ ] Officer flash cash → stock −qty atomically + cap enforced; last-unit race yields exactly one success.
- [ ] Closed/missing session, cross-org campaign/session/officer, missing/invalid email → abort with no partial rows.
- [ ] Duplicate buyer email reuses `buyers` row; duplicate `pickup_code` rejected by `UNIQUE`, issuance retries.
- [ ] `confirm_order_pickup` rejects `PAYMENT_VERIFIED` (must be `READY_FOR_PICKUP` + campaign ready); fee `UNPAID` still releasable (REQ-48).
- [ ] Same officer cannot dual-sign close (`register_signers_differ`); mismatch requires `variance_note`.
- [ ] Magic-link resolves cash order starting at `Payment Verified`; no `Awaiting/Resubmit` UI for cash.
- [ ] Offline terminal blocks submit; timeout recovery re-queries before retaking cash.
