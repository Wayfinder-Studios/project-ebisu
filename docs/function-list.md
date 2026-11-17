# Project Ebisu — Lean Function List

**Version:** 1.1 · **Date:** 2026-11-18 · **Status:** Agreed V1 scope. Replaces the SRS (`project-spec.md`) and the schema doc (`database-schema.md`).

## 1. Purpose and boundaries

Ebisu helps student organizations at Visayas State University sell merch. It covers three things:

- **Inventory:** products, variants and stock.
- **Payment history:** one place for every payment a shop has received.
- **Buying:** a simple way for members and other buyers to find and buy products.

**Everyone must be signed in.** There are no guest or anonymous pages: browsing, cart, checkout and every shop or admin screen require a signed-in Google account. Signed-out visitors see only the sign-in page.

This list defines V1 scope, roles and business rules. It is not a backlog, screen list or schema.

- Keep function IDs `EB-001`–`EB-023` stable when reorganizing headings. Issues and PRs reference them.
- A function may appear on several screens; it is defined once here. A row does not require its own screen or API route.
- The schema lives in `supabase/migrations/` as runnable SQL once coding starts, not in Markdown.
- Anything not listed here is out of scope for V1 (see section 6).

## 2. Roles and access

Roles follow a normal selling platform, not student-org positions. Org titles (President, Treasurer…) are not roles.

| Role | Scope |
|---|---|
| Buyer | Every signed-in account. Browse, buy and see own orders. |
| Shop Staff | Day-to-day selling for one shop: products, payments, cash sales, pickup, history. |
| Shop Owner | **Everything Shop Staff can do**, plus shop profile, payment accounts and members. |
| Admin | Platform team. Creates and deactivates shops, manages the shared payment provider list. No selling authority inside a shop. |

### Assignment rules

- One person, one account, created by their first Google sign-in. Every account starts as a Buyer only.
- Owner and Staff are granted per shop, by invitation, to an **existing** account. A person must have signed in at least once before they can be invited or assigned as first Owner.
- Membership is per shop: a person can be Owner in one shop and Staff in another.
- A shop can have several Owners. A shop must always have at least one Owner.
- Officer handover each school year: the old Owner invites the new officers as Owner, then leaves or is removed.
- Enforce access on the server, never only by hiding buttons.

### Who changes membership?

| Change | Who |
|---|---|
| Create a shop, assign its first Owner | Admin |
| Invite, change role or remove Owner/Staff | Shop Owner |
| Leave a shop | The member (unless they are the last Owner) |

## 3. Function catalog

### Access and account

| ID | Function | Users | Description / scope |
|---|---|---|---|
| EB-001 | Sign in and sign out | Everyone | Google sign-in (Supabase Auth, Google OAuth). The first sign-in creates the account as a Buyer. No passwords, no email links. Required before any page except the sign-in page. |
| EB-002 | Edit own profile | Signed-in users | Name and mobile number (name prefilled from Google). Email comes from Google and is not edited here. |
| EB-003 | Accept shop invitation | Invited person | Pending invitations show in-app for the invited account. Accept to join the shop with the invited role, or decline. An invitation works once and can be cancelled by an Owner before use. |

### Browsing and buying

| ID | Function | Users | Description / scope |
|---|---|---|---|
| EB-004 | Browse shops and products | Buyer | List live products across all active shops. Search by name, filter by shop. Show price, photo, and stock or pre-order deadline. MOQ is never shown to buyers. |
| EB-005 | View product details | Buyer | Photos, description, variants (size/color), price, availability, and the product's custom fields (e.g. jersey name, student ID). Sold-out variants cannot be added. |
| EB-006 | Manage cart | Buyer | Add, change quantity and remove items. One cart per shop, because each shop has its own payment accounts. Stored in the browser; nothing is reserved until checkout. |
| EB-007 | Check out and submit payment | Buyer | Choose one of the shop's active payment accounts, enter the reference number and upload the transfer screenshot. The server re-checks price and stock, **blocks a reference number already used with the same payment provider**, on any account in any shop (see section 5), computes the total, reduces stock and creates the order with a pickup code in one transaction. Any failure creates nothing. |
| EB-008 | View my orders | Buyer | List of own orders with status, items, total, payment details and pickup state. Rejected orders show the reason. |
| EB-009 | Show pickup code | Buyer | Once an order is Paid, show its pickup code and items to present at the booth. Works as a screenshot. |

### Products

| ID | Function | Users | Description / scope |
|---|---|---|---|
| EB-010 | Manage products | Staff | Add, edit and archive products: name, description, photos, price, variants, optional stock per variant (empty = pre-order/unlimited), optional deadline and MOQ (internal target, staff-only), optional custom fields for buyers to fill in. Variants that already have orders can be deactivated, not deleted. |
| EB-011 | Update pre-order progress | Staff | Shop-side view of each pre-order: paid units vs MOQ (overall and per variant), pending unpaid units, and time to deadline. Staff decide; nothing happens automatically. After the deadline: **Proceed** or **Cancel**. Cancel marks paid orders Refund due. Later mark the product **Ready for pickup** so buyers see it on their orders. |

### Orders and payments

| ID | Function | Users | Description / scope |
|---|---|---|---|
| EB-012 | Verify or reject transfer payment | Staff | Queue of pending orders showing screenshot, reference number, amount, destination account and buyer. Verify → Paid. Reject needs a reason and returns stock. Duplicate reference numbers for the same provider never reach this queue (blocked at EB-007). |
| EB-013 | Record cash order | Staff | Take cash for in-stock items at the booth **or for a pre-order**: pick products, variants, quantities and custom fields, find the buyer's account by email, take cash. Creates the order directly as Paid with a pickup code, reduces stock and records who took the cash. In-stock lines can be handed out right away (EB-016); pre-order lines are picked up once the product is Ready for pickup. |
| EB-014 | Cancel order | Staff | Cancel an order that was never paid (e.g. buyer never sent money). Returns stock. |
| EB-015 | Mark refunded | Staff | For orders in Refund due (or a paid order the shop chooses to refund): record that the money was returned outside the app, with an optional note. |
| EB-016 | Hand out items | Staff | Look up a pickup code, see items and custom fields, mark each line picked up. Records who and when. A line already picked up shows who released it and cannot be released again. |
| EB-017 | View payment history | Staff | Every payment for the shop: date, buyer, amount, method (transfer/cash), destination account, verified/recorded by, and status. Filter by date, method, account and staff. Export CSV. |
| EB-018 | View and export orders | Staff | Orders per product with variant and custom-field details. Export CSV of the order list and size/variant counts for the supplier. |

### Shop administration

| ID | Function | Users | Description / scope |
|---|---|---|---|
| EB-019 | Edit shop profile | Owner | Name, description, logo and banner. |
| EB-020 | Manage payment accounts | Owner | Add, edit and archive the shop's receiving accounts shown at checkout. Each account picks a **payment provider** from the shared list (e.g. GCash, Maya, BPI) and adds the account name and number. Owners cannot add providers; if theirs is missing they ask the Admin outside the app (EB-023). Archived accounts are hidden from checkout, but past payments still show the account they went to. Archive, never delete. |
| EB-021 | Manage members | Owner | Invite by email as Owner or Staff, cancel pending invites, change a member's role, remove a member. The last Owner cannot be removed or demoted. Removing someone keeps the history of what they did. |

### Platform

| ID | Function | Users | Description / scope |
|---|---|---|---|
| EB-022 | Manage shops | Admin | Create a shop when an org asks (by message or form outside the app) and invite its first Owner. Deactivate a shop: its products leave the catalog, but its orders and history stay. |
| EB-023 | Manage payment providers | Admin | Add, rename and retire the shared provider list. Seeded before launch with GCash, Maya and the common banks; further providers are added when an Owner asks (by message, outside the app). Retired providers can't be picked for new accounts; existing accounts keep working. |

## 4. Business states

These are business meanings, not a requirement for one database enum each.

| Record | States | Rules |
|---|---|---|
| Shop | Active / Deactivated | Only Active shops appear in the catalog. |
| Membership | Invited / Active / Removed | Invite is one-time. At least one Active Owner per shop. |
| Product | Draft / Live / Closed / Archived | Live products are buyable. Closed = deadline passed or sales stopped. Archived hides it but keeps orders. |
| Pre-order progress | Open / Proceeding / Ready for pickup / Cancelled | Only for products with a deadline. Set by EB-011. |
| Order | Pending payment / Paid / Rejected / Cancelled | Transfer orders start Pending payment; cash orders (in-stock or pre-order) start Paid. Rejected and Cancelled return stock once and are final; the buyer can simply order again. |
| Order refund | — / Refund due / Refunded | Independent of Order status. Refund due comes from a cancelled pre-order. |
| Order line pickup | Not picked up / Picked up | Order is **Completed** when every line is picked up (calculated, not stored). |
| Payment provider | Active / Retired | Shared by all shops, managed only by Admin. |
| Payment account | Active / Archived | Belongs to one shop and one provider. Archived accounts are hidden from checkout only. |

**Stock:** reduced at checkout or cash order; returned when an order is Rejected or Cancelled. The reduce step must be a single atomic database update (`stock >= qty` check in the same statement) so two buyers can't buy the last item.

## 5. Essential safeguards

Keep these. They are small and prevent real damage.

- **Signed in everywhere.** Every route and every database call requires an authenticated session; there is no anonymous access, including the catalog.
- **Server computes everything that matters.** Buyers never write totals, prices or statuses directly. Orders are created only through the checkout and cash-order functions.
- **No duplicate reference numbers per provider.** A provider (GCash, Maya, a bank) issues each reference once, so the same number twice is the same transfer claimed twice. Normalize the reference (trim, remove spaces and dashes, uppercase) and enforce it with a database unique index on (provider, normalized reference) over orders that are not Rejected or Cancelled, so a buyer can resubmit after a rejection. This applies across all accounts and all shops. The error tells the buyer only that the reference was already used, never which shop or order. Cash orders have no provider or reference.
- **One record per provider.** Only Admin writes providers, plus a unique index on the normalized name (lowercase, spaces and punctuation removed), so the reference check can't be split across near-duplicates.
- **Private payment screenshots.** Cloudflare R2 private bucket; the server issues short-lived presigned URLs for upload and viewing, only to the buyer and that shop's staff. Product and shop images go in a public R2 bucket.
- **Image uploads:** JPG/PNG/WebP only (no SVG), with a size limit.
- **CSV exports:** escape cells that start with `=`, `+`, `-` or `@` so spreadsheets don't run them as formulas.
- **Who did what:** verify, reject, cash order, refund and pickup each record the staff member and time.
- **Shop isolation:** staff only see and change their own shop's data (row-level security).

## 6. Not in V1

Decided out, so it doesn't creep back:

- Platform fees, fee settlement, export locks
- Cash register sessions, two-person sign-off, cash reconciliation workflow
- Org accreditation workflow, Platform Owner vs Superadmin tiers
- Treasurer/Cashier/Volunteer roles, custom permissions
- Buyer-facing MOQ progress, automatic MOQ evaluation, override windows, refund queues
- Reservation timers and auto-expiry of unpaid orders
- Payment gateways (PayMongo, Xendit…), installments, partial refunds
- Email notifications for order status or invitations (buyers check EB-008; invites show in-app)
- Email/password or email-link sign-in; guest browsing or guest checkout
- Platform analytics dashboard
- Per-variant images, image galleries beyond a few product photos

## 7. Decisions

| Date | Decision |
|---|---|
| 2026-11-15 | Replaced the SRS v2.12 and schema doc with this function list. The old docs stay in git history. |
| 2026-11-15 | Roles are Buyer, Shop Staff, Shop Owner (superset of Staff) and Admin. |
| 2026-11-15 | Storage moves to Supabase Storage (public + private buckets); Cloudflare R2 is dropped. |
| 2026-11-15 | Duplicate reference numbers show a warning; staff decide. |
| 2026-11-15 | Pickup code is created at checkout and shown once the order is Paid. |
| 2026-11-18 | Storage is Cloudflare R2 (private + public buckets); replaces the 2026-11-15 Supabase Storage decision. |
| 2026-11-18 | Payment providers are shared platform-wide and separate from shop payment accounts. Only Admin manages the list (EB-023); Owners request missing providers outside the app. |
| 2026-11-18 | Duplicate reference numbers are blocked per provider across all accounts and shops; replaces the 2026-11-15 warning decision. |
| 2026-11-18 | Sign-in is Google OAuth only. All accounts start as Buyers; Owner/Staff are invited onto existing accounts. |
| 2026-11-18 | All pages require sign-in; no guest browsing. |
| 2026-11-18 | Cash orders cover pre-orders as well as booth sales (EB-013 renamed Record cash order). |
| 2026-11-18 | MOQ is a staff-only target shown in EB-011; buyers never see it. |

**Implementation details, not scope questions:** pickup code format, CSV column layout, max product photos, presigned URL lifetime, and whether invitations expire.
