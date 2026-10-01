# Software Requirements Specification (SRS)
## Project Ebisu — Campus Merch Platform

**Document Status:** v2.9 — Canonical Engineering Reference (D-06 officer-only cash approved 2026-10-01, builds on D-05 stack reconciliation 2026-09-30)  
**Prepared For:** Full-Stack Engineering Team  

---

## 1. Introduction

### 1.1 Purpose
This specification translates the functional product requirements for **Project Ebisu** into an actionable, traceable technical blueprint. It prioritizes data integrity, operational speed at physical campus booths, and clean multi-tenant architecture.

### 1.2 Scope
The system is a multi-tenant campus commerce platform enabling accredited student organizations and councils to run **Batch Pre-Order campaigns** and **In-Stock Flash Drops**, receive direct buyer payments (GCash, Maya, Bank Transfer via buyer checkout; Cash on Hand via officer-only F2F terminal — D-06), unlock tailor production sheets through a banded platform service fee with a per-campaign minimum (D-04), and distribute merchandise at campus booths via an alphanumeric pick-up code dispatch terminal.

#### In Scope for v1:
* **Org-Scoped Campaigns:** Multi-attribute variant matrices (size, color, cut) and optional institutional ID gating.
* **Buyer Checkout:** Passwordless Buyer identity (Supabase Auth magic links via Google SMTP), source-wallet tracking, and atomic inventory reservation with TTL (30 mins, digital only — D-06). Cash has no buyer reservation/TTL; officer creates cash orders directly as `PAYMENT_VERIFIED`.
* **Operational Verification Desk:** Triage queues for digital payment proofs (cross-wallet/InstaPay support) and officer-only F2F cash terminal under an active register session (D-06).
* **Audit Persistence:** Permanent transaction histories, downloadable digital receipts, and audit trails for buyers and student organizations.
* **MOQ Evaluation:** Automated target checking at campaign deadline, 48-hour Treasurer-gated override window, and a Batch Refund Queue.
* **Banded Platform Toll-Gate (D-04):** Per-unit fee by retail price band — 5% for items under ₱100; flat ₱5 (₱100–₱499), ₱10 (₱500–₱999), ₱15 (₱1,000+) — with a ₱100 per-campaign minimum; unlocking formula-sanitized Master Supplier Exports and tailor sizing matrices.
* **Booth Dispatch Terminal:** Instantaneous alphanumeric pick-up code lookup (`CS-8419`) with atomic double-claim prevention.
* **Daily Register Sessions:** Day-long booth cash tracking ledger with end-of-day dual-officer reconciliation.
* **Superadmin & Governance Portal:** Org accreditation, fee settlement verification, and platform analytics.

#### Out of Scope for v1:
* Open peer-to-peer / student-to-student classified listings.
* Platform payment escrow or custodial holding of user funds.
* Hardware dependencies or camera/QR-based scanning hardware.
* Automated in-app regulatory tracking or compliance workflows.

---

### 1.3 Definitions, Acronyms, Abbreviations

| Term | Definition |
|---|---|
| **MOQ** | Minimum Order Quantity — supplier's required production run size. |
| **Org** | An accredited campus organization, department, or student council acting as a merchant. |
| **Buyer** | Any student, faculty member, alumnus, or campus visitor purchasing merchandise. |
| **Toll-Gate** | The platform-fee checkpoint unlocking master production exports and aggregated sizing matrices. |
| **Pick-up Code** | Human-readable alphanumeric code (e.g., `CS-8419`) identifying a verified order at booth dispatch. |
| **Master Supplier Export** | Formula-sanitized production spreadsheet grouping sizes, cuts, and custom personalization text for print shops. |
| **Daily Register Session** | A day-long cash tracking session allowing multiple rotating booth officers to record cash transactions under a single ledger. |
| **Buyer Account** | Lightweight, passwordless identity tied to email and mobile number, persisting order history and enforcing purchase caps. |
| **Magic Link** | One-time authentication link sent via email to access orders without password management. |
| **Reservation TTL** | Time-To-Live window during which digital inventory is held pending payment proof (30 mins, digital only). Cash orders skip reservation entirely and are created directly as `PAYMENT_VERIFIED` by officers (D-06/D-07). |

---

### 1.4 Engineering Team & Organizational Roles

| Team Member | Roles & Responsibilities | Core Workstream Ownership |
|---|---|---|
| **Jamir** | Project Manager / Tech Lead / Backend | Architecture, Sprint Planning, Core Services, Database & Security |
| **Dohn** | Backend Lead / Full-Stack | API Architecture, Supabase Schema, Concurrency & Integrations |
| **Jaaseia** | Frontend Lead / QA | Buyer PWA UI, Design System, Component Library & Test Scenarios |
| **Gabby** | QA Lead / Full-Stack | Test Strategy, Verification Desk, Cashier Terminals & E2E Testing |
| **Patrick** | Full-Stack / Frontend | UI Implementation, Order Tracker & Magic Link Auth Flows |
| **Xyryll** | Full-Stack / Backend | Production Export Toll-Gate, Register Sessions & Audit Logs |

---

## 2. Overall Description

### 2.1 Product Perspective
Multi-tenant commerce application where each accredited Org operates as an independent merchant. Payment clearing is direct-to-org (no escrow). The platform acts as a trust ledger, operational triage desk, and production coordinator.

### 2.2 User Classes and Characteristics

| User Class | Operational Role | Key Capabilities |
|---|---|---|
| **Buyer** | Campus Customer | Browse merch drops, place custom orders, submit payment proof, receive claim pass, track order status, and retain permanent transaction receipts. |
| **Org Volunteer / Cashier** | Frontline Booth Staff | Verify incoming payments, log walk-in cash under active register session, look up pick-up codes at booth terminal, release items. |
| **Org Treasurer** | Financial Authority | Manage payment accounts, execute MOQ deficit overrides ("Proceed Anyway"), reconcile daily register sessions, settle platform fees, export supplier sheets. |
| **Platform Superadmin** | Governance & Operations | Review and accredit organizations, verify platform fee remittances, oversee platform volume and metrics. |
| **Platform Owner** | Founding Team / System Admins | Strict superset of Superadmin: full access to accreditation, fee clearance, monitoring, overrides, and account provisioning without target-org recusal blocks. |

### 2.3 Operating Environment
* Mobile-responsive web application (PWA-compatible) for buyers and booth officers, built with Next.js (App Router) + TypeScript + TailwindCSS and hosted on Vercel.
* Booth dispatch terminal optimized for touchscreens and mobile browsers under variable campus cellular connectivity.
* Zero external scanner or printer hardware dependencies.

### 2.4 Design and Implementation Constraints
* **Zero Escrow:** Funds never pass through platform accounts; direct-to-merchant clearing only.
* **Permanent Transaction Auditability:** Order history, uploaded receipt screenshots, and payment records remain permanently accessible for self-auditing and dispute resolution.
* **Non-Blocking Distribution:** Unresolved platform fees or administrative audits must never disable booth pickup tools for items already manufactured and on campus.
* **Formula Injection Defense:** All spreadsheet exports are strictly sanitized (`'`, `=`, `+`, `-`, `@`) before generation.

---

## 3. System Features & Use Cases

### 3.1 Module: Campaign Management

#### **UC-01: Create Campaign**
* **Actor:** Org Officer (Treasurer / Admin)
* **Precondition:** Org is accredited and active.
* **Main Flow:**
  1. Officer selects campaign type (`PRE_ORDER` or `FLASH`).
  2. Uploads mockups, sets title, description, and base price.
  3. Configures variant matrix (sizes, cuts, colors) and custom text personalization fields.
  4. Sets Student ID requirement policy (`MANDATORY` or `OPTIONAL`).
  5. For Pre-Orders: sets MOQ target count and campaign cutoff deadline.
  6. For Flash drops: sets initial inventory stock per variant and per-buyer purchase caps.
 7. Configures payment receiving channels (GCash, Maya, Bank details). Cash on Hand is an officer-terminal-only toggle (D-06): it enables F2F cash entry, never a buyer checkout choice; booth instructions render in the officer terminal header only.
 8. Publishes campaign to `LIVE` status.
* **Related Requirements:** `REQ-01` to `REQ-06`, `REQ-06A`

#### **UC-02: Edit Live Campaign**
* **Actor:** Org Officer
* **Precondition:** Campaign is `LIVE`.
* **Main Flow:** Officer modifies non-structural attributes (deadline extensions, announcements, photos).
* **Exception Flow:** Structural changes to variants with existing orders are blocked; variants can only be flagged as inactive for future orders.
* **Related Requirements:** `REQ-07`

---

### 3.2 Module: Buyer Ordering & Checkout

#### **UC-03A: Establish Buyer Account (Passwordless)**
* **Actor:** Buyer
* **Precondition:** Buyer provides full name, email address, and mobile number.
* **Main Flow:** Triggered during first checkout or when accessing order history -> system emails a magic link -> buyer clicks link -> establishes persistent authenticated session without passwords.
* **Related Requirements:** `REQ-08A`

#### **UC-03: Browse and Reserve Variant**
* **Actor:** Buyer
* **Precondition:** Campaign is `LIVE` and before deadline or has available stock.
* **Main Flow:** Buyer views drop feed, selects attributes (size, cut, color), fills required personalization fields (e.g., jersey back name, student ID if enforced) -> proceeds to checkout.
* **Postcondition:** For Flash drops, inventory is atomically decremented and placed under a Reservation TTL.
* **Related Requirements:** `REQ-08` to `REQ-11`

#### **UC-04: Checkout and Payment Submission (Digital Only — D-06)**
* **Actor:** Buyer
* **Precondition:** Valid reservation within TTL window (digital only).
* **Main Flow:**
  - *Digital Wallet / Bank Transfer:* Buyer selects `Source Channel` (GCash, Maya, GoTyme, BDO, BPI, Other) -> inputs transaction reference number / InstaPay Trace ID -> uploads receipt screenshot -> submits -> `PAYMENT_SUBMITTED`.
  - *Cash on Hand:* Not offered in buyer UI. Buyers paying cash transact F2F at the booth; the officer creates the order via UC-07. Any legacy `AWAITING_PAYMENT` cash-reservation path is retired.
* **Exception Flow (Duplicate Reference):** Reference number already exists in database across any campaign -> entry rejected immediately (`REQ-14`; cash `NULL` references exempt).
* **Exception Flow (TTL Expiry):** Payment proof not submitted within 30 minutes -> reservation auto-cancels and variant inventory restores by reserved quantity.
* **Related Requirements:** `REQ-11`, `REQ-12` to `REQ-14`, `REQ-16`, `REQ-13B`

#### **UC-05: Track Order, View Receipts & Resubmit Payment**
* **Actor:** Buyer
* **Precondition:** Buyer authenticated via magic link.
* **Main Flow:** Buyer views permanent purchase history and real-time order status (digital: `Awaiting Payment` -> `Payment Verified` -> `In Production` -> `Ready for Pick-up` -> `Claimed`; officer-entered cash starts at `Payment Verified` — D-06) along with uploaded receipts (digital) and full item breakdowns.
* **Alternate Flow (Resubmission, digital only):** If a digital order is flagged `REJECTED`, buyer reviews the officer's rejection note, edits reference number/channel, uploads new proof, and resubmits without losing their reserved order item. Cash orders have no rejection/resubmission path.
* **Related Requirements:** `REQ-17`, `REQ-17A`

---

### 3.3 Module: Payment Verification

#### **UC-06: Verify Digital Payment (Cross-Wallet Support)**
* **Actor:** Org Officer (Treasurer or Assigned Volunteer)
* **Precondition:** Order is in `PAYMENT_SUBMITTED`.
* **Main Flow:** Officer opens Verification Desk -> reviews order card showing submitted receipt screenshot, timestamp, amount, source channel, reference number, and buyer details side-by-side -> cross-references incoming merchant SMS/wallet logs -> clicks **Verify** or **Reject** (with mandatory reason note).
* **Postcondition:** Order transitions to `PAYMENT_VERIFIED` (generating a pick-up code) or `REJECTED`.
* **Related Requirements:** `REQ-18` to `REQ-20`

#### **UC-07: Create-and-Verify F2F Cash Order (Officer-Only — D-06/D-07)**
* **Actor:** Org Officer (Cashier / Volunteer / Treasurer)
* **Precondition:** Active Daily Register Session open at booth (`status=OPEN`, same org); campaign is `LIVE`; buyer present F2F with cash.
* **Main Flow (unified form with campaign-type branching):** Officer selects campaign -> selects variant/qty + personalization/student-ID -> finds-or-creates buyer (name + **email required** + mobile; duplicate email reuses `buyers` row) -> accepts cash -> submits via `create_cash_order()` in one atomic transaction:
  - *PRE_ORDER:* no stock decrement; order created directly as `PAYMENT_VERIFIED` and counts toward MOQ immediately.
  - *FLASH:* atomic `stock_remaining` decrement (row `FOR UPDATE`) + purchase-cap check in the same transaction; sold-out variants disabled.
  System tags `register_session_id`, `verified_by_officer_id` (= collecting officer), `verified_at=now()`, `payment_method=CASH_ON_HAND`, `source_channel='Cash'`, generates `pickup_code`, writes `order_status_history`, and emails the buyer a magic-link tracker + claim-pass link (REQ-22A; email retry without rolling back the order).
* **Exception Flow:** No `OPEN` session, closed session mid-submit, sold-out race, cross-org campaign/variant/session, or missing email -> transaction aborts with explicit officer-facing error; cash handed back; nothing partially written.
* **Postcondition:** Order is `PAYMENT_VERIFIED` at creation (skips `AWAITING_PAYMENT`; no TTL). Retires the old "verify existing cash reservation / Quick Order Entry as secondary path" model.
* **Related Requirements:** `REQ-21`, `REQ-22`, `REQ-22A`, `REQ-13B`, `REQ-41`

---

### 3.4 Module: MOQ Safeguard & Overrides

#### **UC-08: Evaluate MOQ at Deadline**
* **Actor:** System (automated), Org Treasurer
* **Precondition:** Pre-Order deadline reached.
* **Main Flow:** System tallies total verified orders against `moq_target`.
  - *MOQ Met:* Auto-transitions to `CLOSED_PENDING_PRODUCTION`.
  - *MOQ Missed:* Transitions to `MOQ_FAILED`; opens a 48-hour decision grace period.
* **Alternate Flow (Treasurer Override):** Only an officer tagged as **Treasurer** can tap **"Proceed Anyway (Cover Deficit)"** within 48 hours -> campaign advances to `CLOSED_PENDING_PRODUCTION`.
* **Alternate Flow (Default Refund Trigger):** Grace period expires with no action, or Treasurer triggers cancellation -> campaign auto-cancels and populates the Batch Refund Queue.
* **Related Requirements:** `REQ-23` to `REQ-27`

#### **UC-09: Process Batch Refund**
* **Actor:** Org Officer
* **Precondition:** Campaign in cancelled state with active Refund Queue.
* **Main Flow:** Officers issue manual refunds (GCash/cash) to buyers using listed account details -> upload refund proof reference -> marks order `REFUNDED`.
* **Related Requirements:** `REQ-28`

---

### 3.5 Module: Toll-Gate & Production Handoff

#### **UC-10: Settle Platform Fee (Banded with Campaign Minimum — D-04)**
* **Actor:** Org Treasurer, Platform Superadmin
* **Precondition:** Campaign in `CLOSED_PENDING_PRODUCTION`.
* **Main Flow:** System calculates total fee per verified unit quantity:
  $$\text{Unit Fee} = \begin{cases} \text{Unit Retail Price} \times 5\% & \text{if price} < ₱100 \\ ₱5 & \text{if } ₱100 \le \text{price} \le ₱499 \\ ₱10 & \text{if } ₱500 \le \text{price} \le ₱999 \\ ₱15 & \text{if price} \ge ₱1000 \end{cases}$$
  $$\text{Total Platform Fee} = \max(₱100.00, \sum (\text{Unit Fee} \times \text{Verified Quantity}))$$
  The 5% band is rounded to the nearest centavo. Band boundaries are inclusive as stated. Examples: a ₱20 sticker incurs ₱1.00/unit; a ₱250 shirt incurs ₱5/unit; a ₱750 hoodie incurs ₱10/unit; a ₱1,200 jersey incurs ₱15/unit. A campaign with 5 × ₱250 shirts subtotals ₱25 and is floored to the ₱100 minimum.
  Org transfers fee to Superadmin account and uploads proof -> Superadmin verifies receipt and marks fee `CLEARED`.
* **Postcondition:** Master Supplier Export and aggregated tailor sizing matrices are unlocked.
* **Related Requirements:** `REQ-29` to `REQ-31`

#### **UC-11: Generate Master Supplier Export**
* **Actor:** Org Officer
* **Precondition:** Platform fee is `CLEARED`.
* **Main Flow:** Officer downloads production document grouping items by cut, fabric, size counts, and sanitized custom name rosters -> officer submits file to garment tailor and updates campaign to `IN_PRODUCTION`.
* **Exception Flow (Formula Injection Defense):** System sanitizes all free-text fields (`'`, `=`, `+`, `-`, `@`) prior to export generation to prevent spreadsheet execution exploits.
* **Related Requirements:** `REQ-32`, `REQ-33`

---

### 3.6 Module: Booth Dispatch & Distribution

#### **UC-12: Access Digital Claim Pass**
* **Actor:** Buyer
* **Precondition:** Order is `PAYMENT_VERIFIED` and campaign is `READY_FOR_PICKUP`.
* **Main Flow:** Buyer opens Order Tracker -> displays high-contrast digital claim pass featuring alphanumeric Pick-up Code (e.g., `CS-8419`), verified variant specifications, and buyer name -> taps **Save to Gallery** for offline presentation.
* **Related Requirements:** `REQ-34`, `REQ-35`

#### **UC-13: Dispatch Merchandise at Booth (Online Terminal)**
* **Actor:** Org Officer
* **Precondition:** Campaign is `READY_FOR_PICKUP`; officer terminal has active internet connectivity.
* **Main Flow:** Officer asks for Pick-up Code -> types code into auto-focused search bar -> order record loads with item specs, customization details, and verified payment badge -> officer checks physical university/government ID against order name -> taps **Confirm Release**.
* **Exception Flow (Double Claim):** Order already marked `CLAIMED` -> system displays prominent red blocking warning with original release timestamp and releasing officer name.
* **Postcondition:** Order marked `CLAIMED` via atomic update (`UPDATE orders SET status = 'CLAIMED' ...`) with immutable timestamp and officer attribution.
* **Related Requirements:** `REQ-36` to `REQ-40`, `REQ-48`

---

### 3.7 Module: Cash Registers & Anti-Fraud

#### **UC-14: Manage Daily Register Session & End-of-Day Sign-Off**
* **Actor:** Org Officers, Reviewing Officer (Treasurer / Auditor)
* **Precondition:** Booth physical operations active.
* **Main Flow:** Officer opens a Daily Register Session for the booth -> rotating officers log cash transactions under that active register, with each order storing `collected_by_officer_id` -> at day's end, register closes and displays:
  $$\text{Expected Cash} = \sum (\text{Verified Cash Orders in Session})$$
  Officers count physical bills.
  - If Counted = Expected: Register marks `CLOSED_BALANCED`.
* **Exception Flow (Cash Discrepancy):** Counted cash does not match expected total -> system requires a mandatory explanation note from the closing cashier. Reviewing officer reviews variance and note. Both officers tap digital sign-off. Register closes as `CLOSED_WITH_DISCREPANCY` without halting distribution operations.
* **Related Requirements:** `REQ-41` to `REQ-43`

#### **UC-15: Cross-Platform Payment Reference Deduplication**
* **Actor:** System (automated)
* **Precondition:** Reference number submitted at checkout.
* **Main Flow:** System checks reference string against the unique index across all campaigns and orgs -> rejects duplicate submission immediately.
* **Related Requirements:** `REQ-14`

---

### 3.8 Module: Superadmin & Governance (v2.6 Superset Rules, retained in v2.8)

#### **UC-16: Vet Organization Accreditation**
* **Actor:** Platform Superadmin, Platform Owner
* **Main Flow:** Review council accreditation credentials -> grant active merchant status.

#### **UC-17: Clear Platform Fee Settlement**
* **Actor:** Platform Superadmin, Platform Owner
* **Main Flow:** Review remittance proof -> unlock UC-11 export for that campaign.

#### **UC-18: Monitor Platform Metrics**
* **Actor:** Platform Superadmin, Platform Owner
* **Main Flow:** Monitor platform-wide GMV, active campaigns, volume, and fee collections.

#### **UC-19: Administrative Overrides**
* **Actor:** Platform Superadmin, Platform Owner
* **Main Flow:** Execute emergency campaign status adjustments or dispute resolutions.

#### **UC-20: System Account Provisioning & Fee Exemptions**
* **Actor:** Platform Owner exclusively
* **Main Flow:** Provision Superadmin accounts, toggle `fee_exempt` status, and attach official Memorandum of Agreement (`moa_reference_url`).

---

## 4. Requirement Traceability Matrix

| REQ-ID | Requirement Statement | Module | Use Case | Priority |
|---|---|---|---|---|
| REQ-01 | Support creation of Pre-Order and Flash drop campaigns | Campaign Mgmt | UC-01 | Must |
| REQ-02 | Configure multi-attribute variant matrix and custom text personalization fields | Campaign Mgmt | UC-01 | Must |
| REQ-03 | Enforce mandatory MOQ target and deadline for Pre-Order campaigns | Campaign Mgmt | UC-01 | Must |
| REQ-04 | Enforce fixed initial stock inventory per variant for Flash campaigns | Campaign Mgmt | UC-01 | Must |
| REQ-05 | Enforce per-campaign buyer purchase caps with optional per-variant overrides | Campaign Mgmt | UC-01 | Should |
| REQ-06 | Support independent configuration of GCash, Maya, Bank Transfer channels (buyer checkout) plus Cash as officer-terminal-only toggle (D-06) | Campaign Mgmt | UC-01 | Must |
| REQ-06A | Allow orgs to configure whether Student ID is mandatory or optional per drop | Campaign Mgmt | UC-01 | Should |
| REQ-07 | Restrict structural variant deletion on campaigns with active orders | Campaign Mgmt | UC-02 | Must |
| REQ-08 | Display discovery feed with real-time countdowns and MOQ progress indicators | Ordering | UC-03 | Must |
| REQ-08A | Support passwordless buyer authentication via emailed magic link | Ordering | UC-03A | Must |
| REQ-09 | Provide variant selectors and text inputs for personalization | Ordering | UC-03 | Must |
| REQ-10 | Disable cart addition for sold-out Flash drop variants | Ordering | UC-03 | Must |
| REQ-11 | Decrement Flash inventory atomically at order placement with reservation TTL (digital 30m only); restore reserved quantity on cancel; officer flash-cash decrements atomically inside `create_cash_order()`; preorder cash has no stock op | Ordering | UC-03, UC-04, UC-07 | Must |
| REQ-12 | Capture buyer source wallet/bank channel, reference number, and receipt screenshot (digital checkout only) | Ordering | UC-04 | Must |
| REQ-13 | ~~Allow Cash-on-Hand order reservation with configurable payment instructions~~ — RETIRED in v2.9 (D-06). Buyer cash reservation path removed; see REQ-13B/REQ-21/REQ-22. | Ordering | UC-04 | — |
| REQ-13B | Forbid buyer-created `CASH_ON_HAND` orders; cash creatable only via officer `create_cash_order()` under `OPEN` same-org register session | Ordering / Anti-Fraud | UC-07 | Must |
| REQ-14 | Enforce database-level uniqueness constraint on payment reference numbers across all tenants (cash `NULL` exempt) | Ordering / Anti-Fraud | UC-04, UC-15 | Must |
| REQ-16 | Enforce per-buyer purchase limits server-side across orders | Ordering | UC-04, UC-07 | Should |
| REQ-17 | Provide real-time buyer order status timeline, item details, and permanent receipt view (cash starts at `Payment Verified`) | Ordering | UC-05 | Must |
| REQ-17A | Allow buyers to resubmit corrected payment details on rejected digital orders without duplicate order creation (digital only) | Ordering | UC-05 | Must |
| REQ-18 | Display triage queue of pending payments showing receipt screenshot, amount, source, and reference | Verification | UC-06 | Must |
| REQ-19 | Allow officers to Approve or Reject payments with mandatory rejection reason notes | Verification | UC-06 | Must |
| REQ-20 | Record immutable audit log entries for all status changes with actor ID and timestamp | Verification | UC-06 | Must |
| REQ-21 | Allow officers to create-and-verify F2F cash orders directly as `PAYMENT_VERIFIED` under an `OPEN` register session (no buyer reservation) | Verification | UC-07 | Must |
| REQ-22 | Provide unified officer cash terminal for walk-in cash buyers (find-or-create buyer + order + verify in one transaction, with preorder/flash branching) | Verification | UC-07 | Must |
| REQ-22A | Require buyer email on officer cash entry; send magic-link tracker + claim-pass link via Google SMTP (retry without rolling back order) | Verification | UC-07, UC-05 | Must |
| REQ-23 | Automatically evaluate campaign MOQ against verified orders at deadline | MOQ Safeguard | UC-08 | Must |
| REQ-24 | Auto-transition campaign to `CLOSED_PENDING_PRODUCTION` when MOQ target is met | MOQ Safeguard | UC-08 | Must |
| REQ-25 | Auto-transition campaign to `MOQ_FAILED` with 48-hour grace window when MOQ is missed | MOQ Safeguard | UC-08 | Must |
| REQ-26 | Restrict "Proceed Anyway (Cover Deficit)" action exclusively to Org Treasurer role | MOQ Safeguard | UC-08 | Must |
| REQ-27 | Auto-cancel campaign and generate Batch Refund Queue if override grace window expires | MOQ Safeguard | UC-08 | Must |
| REQ-28 | Track manual refund disbursements with proof attachment until order reaches terminal `REFUNDED` | MOQ Safeguard | UC-09 | Must |
| REQ-29 | Calculate banded per-unit platform fee (5% under ₱100; ₱5 / ₱10 / ₱15 by band) on verified quantity with ₱100 per-campaign minimum; lock export until CLEARED | Toll-Gate | UC-10 | Must |
| REQ-30 | Provide Superadmin interface to review and confirm platform fee payment receipts | Toll-Gate | UC-10 | Must |
| REQ-31 | Automatically unlock Master Supplier Export for campaign upon fee clearance | Toll-Gate | UC-10 | Must |
| REQ-32 | Generate consolidated supplier export grouping garment cuts, sizes, and print specs | Toll-Gate | UC-11 | Must |
| REQ-33 | Sanitize all free-text fields in exports to neutralize spreadsheet formula injection | Toll-Gate | UC-11 | Must |
| REQ-34 | Render ticket-style claim pass with short alphanumeric Pick-up Code (e.g., `CS-8419`) | Distribution | UC-12 | Must |
| REQ-35 | Support client-side image saving of claim pass for offline presentation | Distribution | UC-12 | Must |
| REQ-36 | Provide auto-focused search field for instant Pick-up Code query at booth terminal | Distribution | UC-13 | Must |
| REQ-37 | Display order details, customization specs, and payment status on lookup | Distribution | UC-13 | Must |
| REQ-38 | Prompt officer to verify buyer physical university/national ID prior to release | Distribution | UC-13 | Must |
| REQ-39 | Prevent double-claiming with atomic update and persistent blocking warning | Distribution | UC-13 | Must |
| REQ-40 | Record releasing officer ID and timestamp on claim confirmation | Distribution | UC-13 | Must |
| REQ-41 | Tag officer-created cash orders at creation with active Daily Register Session ID and collecting officer ID (`verified_by_officer_id` = collector) | Anti-Fraud | UC-14, UC-07 | Must |
| REQ-42 | Automatically calculate total expected shift cash based on verified transactions | Anti-Fraud | UC-14 | Must |
| REQ-43 | Require dual-officer sign-off for register closure; require explanatory note for cash variances (`CLOSED_WITH_DISCREPANCY`) without halting operations | Anti-Fraud | UC-14 | Must |
| REQ-45 | Support Superadmin review and onboarding of student organization credentials | Superadmin | UC-16 | Must |
| REQ-46 | Provide Superadmin triage queue to approve platform fee settlement remittances | Superadmin | UC-17 | Must |
| REQ-47 | Provide Superadmin visibility over platform GMV, volume, and active campaigns | Superadmin | UC-18 | Should |
| REQ-48 | Isolate booth distribution tools so fee disputes never block merch releases on campus | Cross-cutting | UC-13 | Must |
| REQ-80 | `PLATFORM_OWNER`'s capability set is a strict superset of `SUPERADMIN` | Governance | UC-16 - UC-20 | Must |
| REQ-81 | The self-dealing recusal rule applies only to `SUPERADMIN`; `PLATFORM_OWNER` actions are never blocked by target org | Governance | UC-16 - UC-19 | Must |

---

## 5. External Interface Requirements

### 5.1 User Interfaces
* **Buyer UI:** Mobile-first responsive web application (TailwindCSS). Includes campaign discovery feed, multi-step variant selection, digital-only checkout with wallet source selector, permanent order tracking timeline, receipt locker, and offline-downloadable claim ticket. No cash option in buyer checkout (D-06).
* **Org Merchant Portal:** Responsive desktop and tablet dashboard. Includes Drop Builder, Verification Desk, officer-only F2F Cash Terminal (unified preorder/flash form, D-06), Daily Register manager, fast-lookup booth dispatch terminal, and gated Production Hub.
* **Superadmin UI:** Administrative web portal for vetting organizations, verifying fee payments, and reviewing platform metrics.

### 5.2 Hardware Interfaces
Zero dedicated hardware interfaces. The system operates on commodity smartphones, tablets, and laptops without external scanners or printers.

### 5.3 Software Interfaces
* **Object Storage:** Cloudflare R2 only for persistent receipt screenshots, product mockups, and refund receipts.
* **Transactional Email:** Supabase Auth sending via Google SMTP (direct sender) to deliver passwordless magic links and status notifications.
* **Database & Persistence:** Supabase (PostgreSQL) with atomic transactional updates, row-level security (RLS), strict uniqueness constraints, and append-only audit tables.

### 5.4 Communications Interfaces
Secure HTTPS web communication with JSON payloads. Polling or lightweight real-time subscriptions for verification queue refreshes.

---

## 6. Non-Functional Requirements (Qualitative)

| Dimension | Expectation |
|---|---|
| **Performance** | Pick-up Code lookups at the booth terminal must execute in sub-second time to prevent distribution line congestion. |
| **Concurrency & Integrity** | Flash drop checkouts must use atomic decrement operations to prevent overselling. Multiple officers dispatching items simultaneously must never result in duplicate item releases. |
| **System Availability** | Booth distribution interfaces must remain completely operational during events, decoupled from administrative or fee-clearance modules. |
| **Data Security** | Payment receipts and transaction identifiers must be protected against tampering. Production spreadsheet exports must be sanitized against formula injection vulnerabilities (`'`, `=`, `+`, `-`, `@`). Order history and uploaded proofs persist permanently. |
| **Usability** | Verification Desk and Booth Dispatch interfaces must be optimized for fast operation by non-technical student volunteers in noisy, crowded campus settings. |
| **Financial Auditability** | Every transaction, payment approval, refund, cash handover, and item release must maintain complete chronological attribution to a specific officer. |

---

## 7. Data Requirements (Entity Overview)

| Entity | Key Attributes | Structural Purpose |
|---|---|---|
| `Organization` | `id`, `name`, `accreditation_status`, `is_active` | Multi-tenant parent entity owning campaigns |
| `User` (Officer/Admin) | `id`, `org_id`, `name`, `email`, `role_tag` (`TREASURER`, `CASHIER`, `VOLUNTEER`, `SUPERADMIN`, `PLATFORM_OWNER`) | Authorizes operational actions and gates Treasurer overrides |
| `Buyer` | `id`, `name`, `email`, `mobile_number`, `created_at` | Passwordless identity for purchase caps and order tracking |
| `Campaign` | `id`, `org_id`, `type` (`PRE_ORDER`, `FLASH`), `status`, `moq_target`, `deadline_at`, `fee_policy_version`, `fee_status`, `require_student_id` | Central campaign coordinator and lifecycle machine |
| `ProductVariant` | `id`, `campaign_id`, `sku`, `attributes` (size/color), `price`, `stock_remaining`, `variant_purchase_cap` | Defines purchasable options and tracks inventory |
| `Order` | `id`, `campaign_id`, `buyer_id`, `status`, `total_amount`, `payment_method`, `source_channel`, `payment_reference`, `receipt_image_url`, `pickup_code`, `reservation_expires_at`, `claimed_at`, `claimed_by_officer_id` | Core transactional record. Persists permanently to allow customer and merchant self-auditing |
| `OrderItem` | `id`, `order_id`, `variant_id`, `customization_data` (printed name/number) | Granular line items and custom personalization specs |
| `OrderStatusHistory` | `id`, `order_id`, `from_status`, `to_status`, `actor_id`, `notes`, `occurred_at` | Immutable chronological audit trail |
| `RegisterSession` | `id`, `org_id`, `session_name`, `opened_by_officer_id`, `closed_by_officer_id`, `cosigned_by_officer_id`, `status` (`OPEN`, `CLOSED_BALANCED`, `CLOSED_WITH_DISCREPANCY`), `expected_cash`, `counted_cash`, `variance_note` | Supports flexible daily cash drawer tracking and reconciliation |
| `Refund` | `id`, `order_id`, `refund_method`, `refund_reference`, `proof_url`, `processed_by_officer_id` | Tracks refund fulfillment on cancelled campaigns |
| `FeeSettlement` | `id`, `campaign_id`, `total_units_sold`, `subtotal_fee`, `total_fee_due` (`max(subtotal, 100)`), `fee_breakdown`, `fee_policy_version`, `minimum_applied`, `payment_proof_url`, `cleared_by_admin_id`, `cleared_at` | Gates Master Supplier Export |

---

## 8. Constraints and Out-of-Scope

### 8.1 Hard Constraints
* **Zero Platform Escrow:** Platform never receives, routes, or holds buyer funds.
* **Fulfillment Independence:** Outstanding platform fees or billing disagreements can lock production exports, but must never disable pickup verification for merchandise that has arrived on campus.
* **No Specialized Hardware:** The entire platform must operate using standard responsive web browsers on commodity smartphones, tablets, and laptops.
* **Audit Persistence:** Transaction records, receipts, and order histories are permanent and never auto-purged.

### 8.2 Out of Scope for v1
* Open peer-to-peer / student-to-student resale marketplaces.
* Automated payment gateway integrations (PayMongo, Xendit, Stripe).
* Camera-based QR/barcode scanning modules.
* In-app regulatory tracking or compliance workflows.

---

## 9. Decisions Log

1. **Cross-Wallet / InstaPay References:** Resolved — Buyer selects `Source Channel` at checkout. Reference number string remains strictly unique across the platform to prevent duplicate slip submissions. The Verification Desk presents screenshots, timestamps, amounts, and source channels so treasurers verify via transaction logs.
2. **Register Sessions vs. Rigid Shifts:** Resolved — Decoupled cash drawer tracking from individual volunteer shifts. Orgs open a Daily Register Session per booth. Rotating volunteers process cash transactions tagged with their individual officer IDs under the active session. Cash counting and dual sign-off occur once at the end of the day.
3. **General Buyer Model:** Resolved — Core identity captures Name, Email (magic link), and Mobile Number. Student ID is an optional campaign-level toggle configured by the org depending on drop restrictions.
4. **Inventory Concurrency & Reservation TTL:** Resolved, amended by D-06/D-07 — Stock decrements atomically at digital checkout submission to prevent overselling. Digital orders have a 30-min Reservation TTL. Cash orders skip reservation/TTL and are created directly as `PAYMENT_VERIFIED` by officers. Unpaid digital orders auto-cancel, restoring reserved quantity. Rejected digital orders follow UC-05 resubmission (retains hold per F02 remediation); cash has no rejection path.
5. **Platform Toll-Gate Economics & Low-Ticket Cap:** Superseded by D-04 (Item 11) on 2026-09-21 — Former volume-scaled model (`min(₱5.00, 5%)`) is no longer in effect. Retained here for history only.
11. **Banded Platform Fee with Campaign Minimum (D-04):** Resolved 2026-09-21 (per `docs/moms/2026-09-21-platform-pricing-approval.md`) — Per-unit fee by retail band: 5% for items under ₱100 (rounded to centavo); flat ₱5 (₱100–₱499), ₱10 (₱500–₱999), ₱15 (₱1,000+). Campaign total is `max(₱100, Σ unit fee × verified quantity)`. The toll-gate locks the consolidated Master Supplier Export and aggregated tailor sizing matrix.
6. **Cash Discrepancy Protocol:** Resolved — Variances mandate an explanatory note and dual-officer digital sign-off (`CLOSED_WITH_DISCREPANCY`). Administrative resolutions remain internal to the organization, avoiding booth halts.
7. **Legal & Compliance Boundary:** Resolved — Out of Scope for Codebase. Corporate formation and statutory compliance are offline organizational matters managed by founders and legal counsel. The platform contains no compliance tables or regulatory status workflows.
8. **Permanent Audit Trail (No Data Purging):** Resolved — All transaction records, order statuses, and receipt screenshots are retained permanently.
9. **Booth Dispatch Network Model (Online-Only Terminal):** Resolved — The Buyer Claim Pass is cached client-side for offline presentation, but the Officer Dispatch Terminal is strictly online-only to prevent split-brain double-release risks. Enforces live atomic database updates (`UPDATE orders SET status = 'CLAIMED' WHERE pickup_code = $1 AND status = 'READY_FOR_PICKUP'`).
10. **Platform Owner Superset (v2.6):** Resolved — `PLATFORM_OWNER` is a strict superset of `SUPERADMIN`, able to perform all governance, accreditation, fee clearance, and monitoring actions without self-dealing recusal blocks.
12. **Tech-Stack Reconciliation (D-05):** Resolved 2026-09-30 (per `docs/moms/2026-09-30-stack-reconciliation.md`) — v1 stack locked to Next.js (App Router) + TypeScript + TailwindCSS on Vercel; Supabase (PostgreSQL + RLS); Cloudflare R2 only for object storage; Supabase Auth sending via Google SMTP (direct) for magic links and notifications. Affirms Kickoff D-01; removes R2/S3 and Tailwind/Vanilla ambiguity.
13. **Officer-Only F2F Cash (D-06):** Resolved 2026-10-01 — Buyer cash-on-hand choice removed from buyer checkout to eliminate 24h unverified holds, MOQ/stock distortion, and expiry-job complexity for preorders/flash. All cash is F2F via officer Cash Terminal under an `OPEN` same-org register session. Retires REQ-13 buyer-reservation path; adds REQ-13B (officer-only creation) and REQ-22A (email-required magic-link tracker).
14. **Cash Skips AWAITING_PAYMENT (D-07):** Resolved 2026-10-01 — Officer cash orders are created directly as `PAYMENT_VERIFIED` with `pickup_code` issued at creation. No TTL, no `AWAITING_PAYMENT`, no rejection/resubmission for cash. Preorder cash counts to MOQ immediately; flash cash decrements stock atomically in the same transaction.
15. **Cash Creation Guardrails (D-08):** Resolved 2026-10-01 — `CASH_ON_HAND` enum retained for audit clarity but buyer `INSERT` denied via RLS; only `create_cash_order()` (officer roles `CASHIER`/`TREASURER`/`VOLUNTEER`, same-org, `OPEN` session) may create cash orders. `verified_by_officer_id` is defined as the collecting officer. Email send failure retries out-of-band and never rolls back the verified sale.
16. **Unified Cash Form (D-09):** Resolved 2026-10-01 — One officer Cash Terminal handles both `PRE_ORDER` (no stock op) and `FLASH` (atomic decrement + cap check) via branching. Booth `payment_instructions` render in officer header only.
