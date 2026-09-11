# Software Requirements Specification (SRS)
## Campus Merch Platform (Project Ebisu)

**Document Status:** v2.6 — Canonical Engineering Reference  
**Prepared For:** Full-Stack Engineering Team  

---

## 1. Introduction

### 1.1 Purpose
This specification translates the functional product requirements for **Project Ebisu** into an actionable, traceable technical blueprint. It prioritizes data integrity, operational speed at physical campus booths, and clean multi-tenant architecture.

### 1.2 Scope
The system is a multi-tenant campus commerce platform enabling accredited student organizations and councils to run **Batch Pre-Order campaigns** and **In-Stock Flash Drops**, receive direct buyer payments (GCash, Maya, Bank Transfer, Cash on Hand), unlock tailor production sheets through a volume-scaled platform service fee, and distribute merchandise at campus booths via an alphanumeric pick-up code dispatch terminal.

#### In Scope for v1:
* **Org-Scoped Campaigns:** Multi-attribute variant matrices (size, color, cut) and optional institutional ID gating.
* **Buyer Checkout:** Passwordless Buyer identity (email magic links), source-wallet tracking, and atomic inventory reservation with TTL.
* **Operational Verification Desk:** Triage queues for digital payment proofs (cross-wallet/InstaPay support) and walk-in cash settlement.
* **Audit Persistence:** Permanent transaction histories, downloadable digital receipts, and audit trails for buyers and student organizations.
* **MOQ Evaluation:** Automated target checking at campaign deadline, 48-hour Treasurer-gated override window, and a Batch Refund Queue.
* **Volume-Scaled Platform Toll-Gate:** 5.00 PHP/item fee (capped at 5% retail price) unlocking formula-sanitized Master Supplier Exports and tailor sizing matrices.
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
| **Reservation TTL** | Time-To-Live window during which inventory is held pending payment proof (30 mins digital) or physical cash settlement (24 hours). |

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
* Mobile-responsive web application (PWA-compatible) for buyers and booth officers.
* Booth dispatch terminal optimized for touchscreens and mobile browsers under variable campus cellular connectivity.
* Zero external scanner or printer hardware dependencies.

### 2.4 Design and Implementation Constraints
* **Zero Escrow:** Funds never pass through platform accounts; direct-to-merchant clearing only.
* **Permanent Transaction Auditability:** Order history, uploaded receipt screenshots, and payment records remain permanently accessible for self-auditing and dispute resolution.
* **Non-Blocking Distribution:** Unresolved platform fees or administrative audits must never disable booth pickup tools for items already manufactured and on campus.
* **Formula Injection Defense:** All spreadsheet exports are strictly sanitized (`'`, `=`, `+`, `-`, `@`) before generation.
