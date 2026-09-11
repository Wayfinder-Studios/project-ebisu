# System Interfaces & Decisions Log
## Project Ebisu — SRS Section 5 through Section 9

---

## 5. External Interface Requirements

### 5.1 User Interfaces
* **Buyer UI:** Mobile-first responsive web application. Includes campaign discovery feed, multi-step variant selection, checkout with wallet source selector, permanent order tracking timeline, receipt locker, and offline-downloadable claim ticket.
* **Org Merchant Portal:** Responsive desktop and tablet dashboard. Includes Drop Builder, Verification Desk, Daily Register manager, fast-lookup booth dispatch terminal, and gated Production Hub.
* **Superadmin UI:** Administrative web portal for vetting organizations, verifying fee payments, and reviewing platform metrics.

### 5.2 Hardware Interfaces
Zero dedicated hardware interfaces. The system operates on commodity smartphones, tablets, and laptops without external scanners or printers.

### 5.3 Software Interfaces
* **Object Storage:** Cloud object store (Cloudflare R2 / AWS S3) for persistent receipt screenshots, product mockups, and refund receipts.
* **Transactional Email:** Transactional email API (Resend) to deliver passwordless magic links and status notifications.
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
| `Campaign` | `id`, `org_id`, `type` (`PRE_ORDER`, `FLASH`), `status`, `moq_target`, `deadline_at`, `per_item_fee`, `fee_status`, `require_student_id` | Central campaign coordinator and lifecycle machine |
| `ProductVariant` | `id`, `campaign_id`, `sku`, `attributes` (size/color), `price`, `stock_remaining`, `variant_purchase_cap` | Defines purchasable options and tracks inventory |
| `Order` | `id`, `campaign_id`, `buyer_id`, `status`, `total_amount`, `payment_method`, `source_channel`, `payment_reference`, `receipt_image_url`, `pickup_code`, `reservation_expires_at`, `claimed_at`, `claimed_by_officer_id` | Core transactional record. Persists permanently to allow customer and merchant self-auditing |
| `OrderItem` | `id`, `order_id`, `variant_id`, `customization_data` (printed name/number) | Granular line items and custom personalization specs |
| `OrderStatusHistory` | `id`, `order_id`, `from_status`, `to_status`, `actor_id`, `notes`, `occurred_at` | Immutable chronological audit trail |
| `RegisterSession` | `id`, `org_id`, `session_name`, `opened_by_officer_id`, `closed_by_officer_id`, `cosigned_by_officer_id`, `status` (`OPEN`, `CLOSED_BALANCED`, `CLOSED_WITH_DISCREPANCY`), `expected_cash`, `counted_cash`, `variance_note` | Supports flexible daily cash drawer tracking and reconciliation |
| `Refund` | `id`, `order_id`, `refund_method`, `refund_reference`, `proof_url`, `processed_by_officer_id` | Tracks refund fulfillment on cancelled campaigns |
| `FeeSettlement` | `id`, `campaign_id`, `total_units_sold`, `fee_per_unit`, `total_fee_due`, `payment_proof_url`, `cleared_by_admin_id`, `cleared_at` | Gates Master Supplier Export |

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
4. **Inventory Concurrency & Reservation TTL:** Resolved — Stock decrements atomically at checkout submission to prevent overselling. Orders have an automated Reservation TTL (30 mins for digital upload, 24 hours for cash payment). Unpaid or rejected orders auto-cancel, restoring inventory back to available stock.
5. **Platform Toll-Gate Economics & Low-Ticket Cap:** Resolved — Volume-scaled service fee: ₱5.00 per item sold, capped at a maximum of 5% of the item's retail price (e.g., a ₱20 sticker incurs a ₱1.00 fee instead of ₱5.00). The toll-gate locks the consolidated Master Supplier Export and aggregated tailor sizing matrix.
6. **Cash Discrepancy Protocol:** Resolved — Variances mandate an explanatory note and dual-officer digital sign-off (`CLOSED_WITH_DISCREPANCY`). Administrative resolutions remain internal to the organization, avoiding booth halts.
7. **Legal & Compliance Boundary:** Resolved — Out of Scope for Codebase. Corporate formation and statutory compliance are offline organizational matters managed by founders and legal counsel. The platform contains no compliance tables or regulatory status workflows.
8. **Permanent Audit Trail (No Data Purging):** Resolved — All transaction records, order statuses, and receipt screenshots are retained permanently.
9. **Booth Dispatch Network Model (Online-Only Terminal):** Resolved — The Buyer Claim Pass is cached client-side for offline presentation, but the Officer Dispatch Terminal is strictly online-only to prevent split-brain double-release risks. Enforces live atomic database updates (`UPDATE orders SET status = 'CLAIMED' WHERE pickup_code = $1 AND status = 'READY_FOR_PICKUP'`).
10. **Platform Owner Superset (v2.6):** Resolved — `PLATFORM_OWNER` is a strict superset of `SUPERADMIN`, able to perform all governance, accreditation, fee clearance, and monitoring actions without self-dealing recusal blocks.
