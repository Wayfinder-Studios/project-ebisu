# Design Specification

*(Draft in progress — D-05 stack note: v1 tokens and components assume TailwindCSS.)*

## v2.10 — Campaign-Scoped Bundle Cart + Partial Pickup (D-10/D-11)

Stack: Next.js App Router + TS + Tailwind on Vercel; Supabase PG + RLS; R2-only; Supabase Auth via Google SMTP. Buyer checkout is digital-only bundle; all cash via officer terminal (now multi-line).

### 1. Buyer flow: campaign grid → variant detail → guest cart → Purchase → single proof (D-10/D-11)

**Campaign page (`/c/[slug]`, guest allowed):**
1. Grid of `is_active` variants: thumbnail, title/attributes, price (read-only), stock badge (`Only 3 left` / `Sold out` disabled) or MOQ progress (`142/200`) + countdown. No auth required.
2. Click variant → detail sheet: image, variant picker (if landing on family), qty stepper (1..cap), per-line personalization fields + student-ID when `require_student_id`, `Add to Cart` → toast + cart badge `N items · ₱est`.
3. Cart stored `localStorage["ebisu:cart:{campaign_id}"]` (max 10 lines / 99 units, guest, no TTL, prices advisory). Cart drawer on campaign page: line rows (edit qty/remove), estimated total, `Checkout / Purchase`. Switching campaigns keeps separate keys; UI warns “Cart is per-campaign — switching starts a new cart” (never merges cross-campaign).
4. Purchase: if guest → magic-link auth wall (name/email/mobile capture) → `reserve_cart(p_lines)` single txn → 30:00 TTL banner + bundle summary (frozen `unit_price` per line, server total).
5. Payment form (digital only): single `Source Channel` + single reference/Trace ID + single R2 receipt for the bundle total → `PAYMENT_SUBMITTED`. Duplicate ref rejected globally. Expiry restores each line qty once.
6. Tracker (`/orders/[id]`): order timeline + per-line dots (`Ready` / `Claimed`), single receipt view, `REJECTED` → edit ref/channel + re-upload (same `order_id`, lines retained). Claim pass shows 1 big `pickup_code` + per-line `Claimed x/y`.

States: empty grid, empty cart, campaign-switched, TTL expiring (<5:00 warning) / expired (restore notice + “Rebuild cart”), resubmit, offline (browse cached, block Purchase with “online-only checkout”).

A11y/perf: ≥44px targets, focus grid→detail→cart→Purchase, labels, non-color status, optimistic drawer (<100ms local), server reserve p95 noted for pilot test.

### 2. Cash Terminal screen (Org Portal, mobile-first, `/org/cash` — multi-line D-10)

Header: org + booth `session_name` + `OPEN` badge + `Expected Cash` live total + booth `payment_instructions`. Block entire form when no `OPEN` session with "Open a register session first" CTA.

Form (single page, unified preorder/flash, 1–10 lines same campaign):
1. Campaign picker (only `LIVE`, same org; `PRE_ORDER`/`FLASH` badge + MOQ/stock). Locks bundle scope once first line added.
2. Repeatable line rows: variant picker (only `is_active`; flash sold-out disabled) + qty stepper + unit price (read-only, server-frozen) + per-line personalization + `Remove`. `+ Add line` (max 10). Running bundle total.
3. Student-ID per line when `require_student_id`.
4. Buyer fields: full name (required), email (required, format-validated, duplicate reuses account), mobile (required). Hint: "Tracker + claim pass sent here."
5. Tendered cash + change due (client-computed, server ignores).
6. Submit `Log Cash Sale (Verified, N lines)` → `create_cash_order(p_lines JSONB)` → success panel with big `pickup_code`, bundle total, line count, buyer email, "Resend tracker email" + "New sale". Failure panel (no OPEN session / sold out / cap hit / cross-org-campaign / missing email / >10 lines) with "cash handed back, nothing written" guidance.

States: loading, empty, permission-denied, offline block, uncertain-result (“Do not retake cash — re-check by pickup code/email”; idempotency via buyer-email + lines re-query).

A11y/perf: touch ≥44px, autofocus campaign→line1→email, sub-second code render.

### 3. Dispatch terminal — partial pickup (D-11, `/org/dispatch`)

Lookup `pickup_code` → bundle card: buyer + ID-check prompt (once per visit) + per-line rows with checkbox (only `READY_FOR_PICKUP` selectable; `CLAIMED` rows red-locked with timestamp + officer) + `Confirm Release (selected)` → `confirm_order_pickup(code, officer, item_ids[])`. Progress `2 of 3 claimed`; order flips `PARTIALLY_CLAIMED` → `CLAIMED` when zero lines remain. Fee `UNPAID` never blocks (REQ-48).

### 4. Tracker email template (REQ-22A, Google SMTP via Supabase Auth)

Subject: `[Ebisu] Your order {pickup_code} is verified — {campaign_title}`

Body:
- Greeting `{buyer_name}`, org + campaign + N-line bundle table (variant/qty/price) + total paid + `pickup_code` (large).
- Per-line claim hint (“You can pick up lines across visits — code stays the same until fully claimed”).
- Magic-link button `Track my order` + plain URL fallback.
- "Show this code + school ID at pickup. We'll email when Ready for Pick-up."
- Footer: booth contact, receipt note ("Cash collected F2F under {session_name}" for cash; "Ref {source_channel} {masked_ref}" for digital), no-reply guidance.

Send: queued post-commit. On SMTP fail: log + "email pending" with retry; never roll back sale (D-08).

### 5. v2.10 acceptance checklist (maps to DEV-02/03/08/11 + F04/F11)

- [ ] Guest builds 3-line cart offline-local; auth only at Purchase; server freezes prices/totals.
- [ ] Purchase 2-variant FLASH bundle for last units → exactly one winner; loser rolls back whole bundle, no partial hold.
- [ ] TTL expiry restores each line qty exactly once; re-Purchase works with fresh TTL.
- [ ] Per-variant + per-campaign caps enforced on digital `reserve_cart()` and cash `create_cash_order(p_lines)`.
- [ ] Cross-campaign / cross-org lines rejected; >10 lines / >99 units rejected; single-variant callers pass 1-line bundle.
- [ ] Cross-campaign cart switch warns, never merges; one ref/proof/code per bundle; duplicate ref rejected.
- [ ] Officer multi-line preorder cash → `PAYMENT_VERIFIED` + MOQ `SUM(qty)`; flash cash → per-line stock decrement.
- [ ] Partial pickup: claim line 1 → `PARTIALLY_CLAIMED`; re-claim line 1 blocked red; claim rest → `CLAIMED`. `PAYMENT_VERIFIED` never releasable; fee `UNPAID` still releasable.
- [ ] Buyer cannot `INSERT CASH_ON_HAND`; sees no cash option; magic-link tracker shows per-line dots starting at correct state.
