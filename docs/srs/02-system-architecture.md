# System Architecture & Functional Requirements
## Project Ebisu — SRS Section 3 & Section 4

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
  7. Configures payment receiving channels (GCash, Maya, Bank details, or Cash on Hand instructions).
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
* **Main Flow:** Triggered during first checkout or when accessing order history → system emails a magic link → buyer clicks link → establishes persistent authenticated session without passwords.
* **Related Requirements:** `REQ-08A`

#### **UC-03: Browse and Reserve Variant**
* **Actor:** Buyer
* **Precondition:** Campaign is `LIVE` and before deadline or has available stock.
* **Main Flow:** Buyer views drop feed, selects attributes (size, cut, color), fills required personalization fields (e.g., jersey back name, student ID if enforced) → proceeds to checkout.
* **Postcondition:** For Flash drops, inventory is atomically decremented and placed under a Reservation TTL.
* **Related Requirements:** `REQ-08` to `REQ-11`

#### **UC-04: Checkout and Payment Submission**
* **Actor:** Buyer
* **Precondition:** Valid reservation within TTL window.
* **Main Flow:**
  - *Digital Wallet / Bank Transfer:* Buyer selects `Source Channel` (GCash, Maya, GoTyme, BDO, BPI, Other) → inputs transaction reference number / InstaPay Trace ID → uploads receipt screenshot → submits.
  - *Cash on Hand:* Buyer reserves order in `AWAITING_PAYMENT` with clear campus booth location and payment deadline instructions.
* **Exception Flow (Duplicate Reference):** Reference number already exists in database across any campaign → entry rejected immediately (`REQ-14`).
* **Exception Flow (TTL Expiry):** Payment proof not submitted within TTL window (30 minutes for digital, 24 hours for cash) → reservation auto-cancels and variant inventory increments back (`+1`).
* **Related Requirements:** `REQ-11`, `REQ-12` to `REQ-14`, `REQ-16`

#### **UC-05: Track Order, View Receipts & Resubmit Payment**
* **Actor:** Buyer
* **Precondition:** Buyer authenticated via magic link.
* **Main Flow:** Buyer views permanent purchase history and real-time order status (`Awaiting Payment` → `Payment Verified` → `In Production` → `Ready for Pick-up` → `Claimed`) along with uploaded receipts and full item breakdowns.
* **Alternate Flow (Resubmission):** If an order is flagged `REJECTED`, buyer reviews the officer's rejection note, edits reference number/channel, uploads new proof, and resubmits without losing their reserved order item.
* **Related Requirements:** `REQ-17`, `REQ-17A`

---

### 3.3 Module: Payment Verification

#### **UC-06: Verify Digital Payment (Cross-Wallet Support)**
* **Actor:** Org Officer (Treasurer or Assigned Volunteer)
* **Precondition:** Order is in `PAYMENT_SUBMITTED`.
* **Main Flow:** Officer opens Verification Desk → reviews order card showing submitted receipt screenshot, timestamp, amount, source channel, reference number, and buyer details side-by-side → cross-references incoming merchant SMS/wallet logs → clicks **Verify** or **Reject** (with mandatory reason note).
* **Postcondition:** Order transitions to `PAYMENT_VERIFIED` (generating a pick-up code) or `REJECTED`.
* **Related Requirements:** `REQ-18` to `REQ-20`

#### **UC-07: Verify Walk-In Cash Payment**
* **Actor:** Org Officer (Cashier / Volunteer)
* **Precondition:** Active Daily Register Session open at booth; buyer presents cash.
* **Main Flow:** Officer accepts cash → searches reservation by buyer name, phone, or student ID → marks order `Paid (Cash Verified)`. If buyer is an unregistered walk-in, officer uses Quick Order Entry to generate and verify the order on the spot.
* **Postcondition:** Order transitions to `PAYMENT_VERIFIED` and transaction is tagged with the collecting officer's ID and active Register Session ID.
* **Related Requirements:** `REQ-21`, `REQ-22`

---

### 3.4 Module: MOQ Safeguard & Overrides

#### **UC-08: Evaluate MOQ at Deadline**
* **Actor:** System (automated), Org Treasurer
* **Precondition:** Pre-Order deadline reached.
* **Main Flow:** System tallies total verified orders against `moq_target`.
  - *MOQ Met:* Auto-transitions to `CLOSED_PENDING_PRODUCTION`.
  - *MOQ Missed:* Transitions to `MOQ_FAILED`; opens a 48-hour decision grace period.
* **Alternate Flow (Treasurer Override):** Only an officer tagged as **Treasurer** can tap **"Proceed Anyway (Cover Deficit)"** within 48 hours → campaign advances to `CLOSED_PENDING_PRODUCTION`.
* **Alternate Flow (Default Refund Trigger):** Grace period expires with no action, or Treasurer triggers cancellation → campaign auto-cancels and populates the Batch Refund Queue.
* **Related Requirements:** `REQ-23` to `REQ-27`

#### **UC-09: Process Batch Refund**
* **Actor:** Org Officer
* **Precondition:** Campaign in cancelled state with active Refund Queue.
* **Main Flow:** Officers issue manual refunds (GCash/cash) to buyers using listed account details → upload refund proof reference → marks order `REFUNDED`.
* **Related Requirements:** `REQ-28`

---

### 3.5 Module: Toll-Gate & Production Handoff

#### **UC-10: Settle Platform Fee (Volume-Scaled with Cap)**
* **Actor:** Org Treasurer, Platform Superadmin
* **Precondition:** Campaign in `CLOSED_PENDING_PRODUCTION`.
* **Main Flow:** System calculates total fee:
  $$\text{Unit Fee} = \min(₱5.00, \text{Unit Retail Price} \times 5\%)$$
  $$\text{Total Platform Fee} = \sum (\text{Unit Fee} \times \text{Verified Units})$$
  Org transfers fee to Superadmin account and uploads proof → Superadmin verifies receipt and marks fee `CLEARED`.
* **Postcondition:** Master Supplier Export and aggregated tailor sizing matrices are unlocked.
* **Related Requirements:** `REQ-29` to `REQ-31`

#### **UC-11: Generate Master Supplier Export**
* **Actor:** Org Officer
* **Precondition:** Platform fee is `CLEARED`.
* **Main Flow:** Officer downloads production document grouping items by cut, fabric, size counts, and sanitized custom name rosters → officer submits file to garment tailor and updates campaign to `IN_PRODUCTION`.
* **Exception Flow (Formula Injection Defense):** System sanitizes all free-text fields (`'`, `=`, `+`, `-`, `@`) prior to export generation to prevent spreadsheet execution exploits.
* **Related Requirements:** `REQ-32`, `REQ-33`

---

### 3.6 Module: Booth Dispatch & Distribution

#### **UC-12: Access Digital Claim Pass**
* **Actor:** Buyer
* **Precondition:** Order is `PAYMENT_VERIFIED` and campaign is `READY_FOR_PICKUP`.
* **Main Flow:** Buyer opens Order Tracker → displays high-contrast digital claim pass featuring alphanumeric Pick-up Code (e.g., `CS-8419`), verified variant specifications, and buyer name → taps **Save to Gallery** for offline presentation.
* **Related Requirements:** `REQ-34`, `REQ-35`

#### **UC-13: Dispatch Merchandise at Booth (Online Terminal)**
* **Actor:** Org Officer
* **Precondition:** Campaign is `READY_FOR_PICKUP`; officer terminal has active internet connectivity.
* **Main Flow:** Officer asks for Pick-up Code → types code into auto-focused search bar → order record loads with item specs, customization details, and verified payment badge → officer checks physical university/government ID against order name → taps **Confirm Release**.
* **Exception Flow (Double Claim):** Order already marked `CLAIMED` → system displays prominent red blocking warning with original release timestamp and releasing officer name.
* **Postcondition:** Order marked `CLAIMED` via atomic update (`UPDATE orders SET status = 'CLAIMED' ...`) with immutable timestamp and officer attribution.
* **Related Requirements:** `REQ-36` to `REQ-40`, `REQ-48`

---

### 3.7 Module: Cash Registers & Anti-Fraud

#### **UC-14: Manage Daily Register Session & End-of-Day Sign-Off**
* **Actor:** Org Officers, Reviewing Officer (Treasurer / Auditor)
* **Precondition:** Booth physical operations active.
* **Main Flow:** Officer opens a Daily Register Session for the booth → rotating officers log cash transactions under that active register, with each order storing `collected_by_officer_id` → at day's end, register closes and displays:
  $$\text{Expected Cash} = \sum (\text{Verified Cash Orders in Session})$$
  Officers count physical bills.
  - If Counted = Expected: Register marks `CLOSED_BALANCED`.
* **Exception Flow (Cash Discrepancy):** Counted cash does not match expected total → system requires a mandatory explanation note from the closing cashier. Reviewing officer reviews variance and note. Both officers tap digital sign-off. Register closes as `CLOSED_WITH_DISCREPANCY` without halting distribution operations.
* **Related Requirements:** `REQ-41` to `REQ-43`

#### **UC-15: Cross-Platform Payment Reference Deduplication**
* **Actor:** System (automated)
* **Precondition:** Reference number submitted at checkout.
* **Main Flow:** System checks reference string against the unique index across all campaigns and orgs → rejects duplicate submission immediately.
* **Related Requirements:** `REQ-14`

---

### 3.8 Module: Superadmin & Governance (v2.6 Superset Rules)

#### **UC-16: Vet Organization Accreditation**
* **Actor:** Platform Superadmin, Platform Owner
* **Main Flow:** Review council accreditation credentials → grant active merchant status.

#### **UC-17: Clear Platform Fee Settlement**
* **Actor:** Platform Superadmin, Platform Owner
* **Main Flow:** Review remittance proof → unlock UC-11 export for that campaign.

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
| REQ-06 | Support independent configuration of GCash, Maya, Bank Transfer, and Cash channels | Campaign Mgmt | UC-01 | Must |
| REQ-06A | Allow orgs to configure whether Student ID is mandatory or optional per drop | Campaign Mgmt | UC-01 | Should |
| REQ-07 | Restrict structural variant deletion on campaigns with active orders | Campaign Mgmt | UC-02 | Must |
| REQ-08 | Display discovery feed with real-time countdowns and MOQ progress indicators | Ordering | UC-03 | Must |
| REQ-08A | Support passwordless buyer authentication via emailed magic link | Ordering | UC-03A | Must |
| REQ-09 | Provide variant selectors and text inputs for personalization | Ordering | UC-03 | Must |
| REQ-10 | Disable cart addition for sold-out Flash drop variants | Ordering | UC-03 | Must |
| REQ-11 | Decrement Flash inventory atomically at order placement with reservation TTL; restore on cancel | Ordering | UC-03, UC-04 | Must |
| REQ-12 | Capture buyer source wallet/bank channel, reference number, and receipt screenshot | Ordering | UC-04 | Must |
| REQ-13 | Allow Cash-on-Hand order reservation with configurable payment instructions | Ordering | UC-04 | Must |
| REQ-14 | Enforce database-level uniqueness constraint on payment reference numbers across all tenants | Ordering / Anti-Fraud | UC-04, UC-15 | Must |
| REQ-16 | Enforce per-buyer purchase limits server-side across orders | Ordering | UC-04 | Should |
| REQ-17 | Provide real-time buyer order status timeline, item details, and permanent receipt view | Ordering | UC-05 | Must |
| REQ-17A | Allow buyers to resubmit corrected payment details on rejected orders without duplicate order creation | Ordering | UC-05 | Must |
| REQ-18 | Display triage queue of pending payments showing receipt screenshot, amount, source, and reference | Verification | UC-06 | Must |
| REQ-19 | Allow officers to Approve or Reject payments with mandatory rejection reason notes | Verification | UC-06 | Must |
| REQ-20 | Record immutable audit log entries for all status changes with actor ID and timestamp | Verification | UC-06 | Must |
| REQ-21 | Allow officers to verify Cash-on-Hand reservations at the booth | Verification | UC-07 | Must |
| REQ-22 | Provide Quick Order Entry for walk-in cash buyers without prior online reservations | Verification | UC-07 | Must |
| REQ-23 | Automatically evaluate campaign MOQ against verified orders at deadline | MOQ Safeguard | UC-08 | Must |
| REQ-24 | Auto-transition campaign to `CLOSED_PENDING_PRODUCTION` when MOQ target is met | MOQ Safeguard | UC-08 | Must |
| REQ-25 | Auto-transition campaign to `MOQ_FAILED` with 48-hour grace window when MOQ is missed | MOQ Safeguard | UC-08 | Must |
| REQ-26 | Restrict "Proceed Anyway (Cover Deficit)" action exclusively to Org Treasurer role | MOQ Safeguard | UC-08 | Must |
| REQ-27 | Auto-cancel campaign and generate Batch Refund Queue if override grace window expires | MOQ Safeguard | UC-08 | Must |
| REQ-28 | Track manual refund disbursements with proof attachment until order reaches terminal `REFUNDED` | MOQ Safeguard | UC-09 | Must |
| REQ-29 | Calculate platform fee as flat fee per verified item sold (capped at 5% of item retail price); lock export until CLEARED | Toll-Gate | UC-10 | Must |
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
| REQ-41 | Tag verified cash orders with active Daily Register Session ID and collecting officer ID | Anti-Fraud | UC-14 | Must |
| REQ-42 | Automatically calculate total expected shift cash based on verified transactions | Anti-Fraud | UC-14 | Must |
| REQ-43 | Require dual-officer sign-off for register closure; require explanatory note for cash variances (`CLOSED_WITH_DISCREPANCY`) without halting operations | Anti-Fraud | UC-14 | Must |
| REQ-45 | Support Superadmin review and onboarding of student organization credentials | Superadmin | UC-16 | Must |
| REQ-46 | Provide Superadmin triage queue to approve platform fee settlement remittances | Superadmin | UC-17 | Must |
| REQ-47 | Provide Superadmin visibility over platform GMV, volume, and active campaigns | Superadmin | UC-18 | Should |
| REQ-48 | Isolate booth distribution tools so fee disputes never block merch releases on campus | Cross-cutting | UC-13 | Must |
| REQ-80 | `PLATFORM_OWNER`'s capability set is a strict superset of `SUPERADMIN` | Governance | UC-16 - UC-20 | Must |
| REQ-81 | The self-dealing recusal rule applies only to `SUPERADMIN`; `PLATFORM_OWNER` actions are never blocked by target org | Governance | UC-16 - UC-19 | Must |
