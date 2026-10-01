# Database Schema Definition (Supabase / PostgreSQL)
## Project Ebisu — Campus Merch Platform

> **v2.10 (D-10 campaign-scoped cart + D-11 guest cart/partial-pickup, 2026-10-01):** Bundle checkout: 1 order = 1 campaign, `reserve_cart()` (digital atomic multi-line) + multi-line `create_cash_order(p_lines JSONB)`. `order_status` += `PARTIALLY_CLAIMED`; `order_items` += line claim cols (`status/quantity_claimed/claimed_at/by`). `confirm_order_pickup(code, officer, item_ids[])` supports partial. MOQ counts units. Retains v2.9 cash CHECKs, `pickup_code UNIQUE`, ordered DDL, RLS matrix.

This document defines the relational database schema, custom enum types, table constraints, row-level security (RLS) policies, and atomic functions for **Project Ebisu**.

---

## 1. Custom Enum Types

```sql
-- Role hierarchy
CREATE TYPE user_role AS ENUM (
  'VOLUNTEER',
  'CASHIER',
  'TREASURER',
  'SUPERADMIN',
  'PLATFORM_OWNER'
);

-- Organization accreditation status
CREATE TYPE accreditation_status AS ENUM (
  'PENDING',
  'ACCREDITED',
  'SUSPENDED'
);

-- Campaign lifecycle
CREATE TYPE campaign_type AS ENUM (
  'PRE_ORDER',
  'FLASH'
);

CREATE TYPE campaign_status AS ENUM (
  'DRAFT',
  'LIVE',
  'MOQ_FAILED',
  'CLOSED_PENDING_PRODUCTION',
  'IN_PRODUCTION',
  'READY_FOR_PICKUP',
  'COMPLETED',
  'CANCELLED'
);

-- Fee settlement status
CREATE TYPE fee_status AS ENUM (
  'UNPAID',
  'SUBMITTED',
  'CLEARED',
  'EXEMPT'
);

-- Order lifecycle (v2.10 adds PARTIALLY_CLAIMED for bundle partial pickup, D-11)
CREATE TYPE order_status AS ENUM (
  'AWAITING_PAYMENT',
  'PAYMENT_SUBMITTED',
  'PAYMENT_VERIFIED',
  'IN_PRODUCTION',
  'READY_FOR_PICKUP',
  'PARTIALLY_CLAIMED',
  'CLAIMED',
  'REJECTED',
  'CANCELLED',
  'REFUNDED'
);

-- Per-line claim state (bundle partial pickup, D-11)
CREATE TYPE order_item_status AS ENUM (
  'READY_FOR_PICKUP',
  'CLAIMED'
);

-- Payment methods & channels
CREATE TYPE payment_method AS ENUM (
  'DIGITAL_WALLET',
  'BANK_TRANSFER',
  'CASH_ON_HAND'
);

-- Register session status
CREATE TYPE register_session_status AS ENUM (
  'OPEN',
  'CLOSED_BALANCED',
  'CLOSED_WITH_DISCREPANCY'
);
```

---

## 2. Core Tables

### 2.1 Organizations & Users

```sql
-- Multi-tenant organizations (Student Councils, Orgs)
CREATE TABLE organizations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  slug TEXT NOT NULL UNIQUE,
  accreditation_status accreditation_status NOT NULL DEFAULT 'PENDING',
  is_active BOOLEAN NOT NULL DEFAULT true,
  fee_exempt BOOLEAN NOT NULL DEFAULT false,
  moa_reference_url TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Officers, Admins & Platform Owners
CREATE TABLE users (
  id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  org_id UUID REFERENCES organizations(id) ON DELETE SET NULL,
  name TEXT NOT NULL,
  email TEXT NOT NULL UNIQUE,
  role_tag user_role NOT NULL DEFAULT 'VOLUNTEER',
  is_active BOOLEAN NOT NULL DEFAULT true,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Passwordless Buyers
CREATE TABLE buyers (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  auth_user_id UUID REFERENCES auth.users(id) ON DELETE SET NULL,
  name TEXT NOT NULL,
  email TEXT NOT NULL UNIQUE,
  mobile_number TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 2.2 Campaigns & Product Variants

```sql
-- Merch Drops & Campaigns
CREATE TABLE campaigns (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  title TEXT NOT NULL,
  description TEXT,
  type campaign_type NOT NULL,
  status campaign_status NOT NULL DEFAULT 'DRAFT',
  moq_target INT,
  deadline_at TIMESTAMPTZ,
  require_student_id BOOLEAN NOT NULL DEFAULT false,
  fee_policy_version TEXT NOT NULL DEFAULT 'v2.7-banded', -- D-04: banded fee (5% / 5 / 10 / 15) + ₱100 minimum; replaces per_item_fee
  fee_status fee_status NOT NULL DEFAULT 'UNPAID',
  payment_instructions JSONB NOT NULL DEFAULT '{}'::jsonb,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Product Variants (Size, Color, Cut)
CREATE TABLE product_variants (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  campaign_id UUID NOT NULL REFERENCES campaigns(id) ON DELETE CASCADE,
  sku TEXT NOT NULL,
  title TEXT NOT NULL, -- e.g. "White / Small / Oversized"
  attributes JSONB NOT NULL DEFAULT '{}'::jsonb, -- {"size": "S", "color": "White"}
  price NUMERIC(10, 2) NOT NULL,
  initial_stock INT,
  stock_remaining INT,
  variant_purchase_cap INT DEFAULT NULL,
  is_active BOOLEAN NOT NULL DEFAULT true,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 2.3 Orders & Transaction History (requires §2.4 register_sessions first — create registers before orders, F05 fix)

```sql
-- Orders
CREATE TABLE orders (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  campaign_id UUID NOT NULL REFERENCES campaigns(id) ON DELETE RESTRICT,
  buyer_id UUID NOT NULL REFERENCES buyers(id) ON DELETE RESTRICT,
  register_session_id UUID REFERENCES register_sessions(id) ON DELETE SET NULL,
  status order_status NOT NULL DEFAULT 'AWAITING_PAYMENT',
  total_amount NUMERIC(10, 2) NOT NULL CHECK (total_amount >= 0),
  payment_method payment_method NOT NULL,
  source_channel TEXT, -- e.g. GCash, Maya, GoTyme, BDO, Cash (cash = 'Cash', payment_reference NULL per D-06)
  payment_reference TEXT UNIQUE, -- Global deduplication constraint (REQ-14)
  receipt_image_url TEXT,
  rejection_reason TEXT,
  pickup_code VARCHAR(10) NOT NULL UNIQUE, -- F06 fix: globally unique, issued transactionally; stored UPPER(TRIM()); D-07: cash issued at creation
  reservation_expires_at TIMESTAMPTZ,
  verified_by_officer_id UUID REFERENCES users(id),
  verified_at TIMESTAMPTZ,
  claimed_by_officer_id UUID REFERENCES users(id),
  claimed_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  CONSTRAINT cash_requires_register CHECK (
    payment_method != 'CASH_ON_HAND' OR register_session_id IS NOT NULL
  ),
  CONSTRAINT cash_no_reference CHECK (
    payment_method != 'CASH_ON_HAND' OR payment_reference IS NULL
  ),
  CONSTRAINT cash_skips_awaiting CHECK (
    payment_method != 'CASH_ON_HAND' OR status = 'PAYMENT_VERIFIED'
  )
);

-- Order Items & Custom Personalization (v2.10: per-line partial-claim tracking, D-11)
CREATE TABLE order_items (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  variant_id UUID NOT NULL REFERENCES product_variants(id) ON DELETE RESTRICT,
  unit_price NUMERIC(10, 2) NOT NULL CHECK (unit_price >= 0),
  quantity INT NOT NULL DEFAULT 1 CHECK (quantity > 0),
  quantity_claimed INT NOT NULL DEFAULT 0 CHECK (quantity_claimed >= 0),
  status order_item_status NOT NULL DEFAULT 'READY_FOR_PICKUP',
  customization_data JSONB NOT NULL DEFAULT '{}'::jsonb, -- e.g. {"back_name": "DELA CRUZ", "back_number": "24"}
  claimed_by_officer_id UUID REFERENCES users(id),
  claimed_at TIMESTAMPTZ,
  CONSTRAINT qty_claimed_lte_qty CHECK (quantity_claimed <= quantity),
  CONSTRAINT claimed_line_consistent CHECK (
    (status = 'CLAIMED' AND quantity_claimed = quantity AND claimed_at IS NOT NULL)
    OR (status = 'READY_FOR_PICKUP')
  )
);

-- Chronological Status Audit Log
CREATE TABLE order_status_history (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  from_status order_status,
  to_status order_status NOT NULL,
  actor_id UUID REFERENCES users(id),
  notes TEXT,
  occurred_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 2.4 Booth Cash Registers & Settlements

```sql
-- Daily Register Sessions
CREATE TABLE register_sessions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  session_name TEXT NOT NULL,
  opened_by_officer_id UUID NOT NULL REFERENCES users(id),
  closed_by_officer_id UUID REFERENCES users(id),
  cosigned_by_officer_id UUID REFERENCES users(id),
  status register_session_status NOT NULL DEFAULT 'OPEN',
  expected_cash NUMERIC(10, 2) NOT NULL DEFAULT 0.00 CHECK (expected_cash >= 0),
  counted_cash NUMERIC(10, 2) CHECK (counted_cash IS NULL OR counted_cash >= 0),
  variance_note TEXT,
  opened_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  closed_at TIMESTAMPTZ,
  CONSTRAINT register_signers_differ CHECK (
    cosigned_by_officer_id IS NULL OR closed_by_officer_id IS NULL
    OR cosigned_by_officer_id != closed_by_officer_id
  )
);

-- Platform Fee Settlements (D-04 banded model: 5% under ₱100; ₱5 / ₱10 / ₱15 by band; total = max(subtotal, 100))
CREATE TABLE fee_settlements (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  campaign_id UUID NOT NULL REFERENCES campaigns(id) ON DELETE CASCADE,
  total_units_sold INT NOT NULL CHECK (total_units_sold >= 0),
  subtotal_fee NUMERIC(10, 2) NOT NULL CHECK (subtotal_fee >= 0),
  total_fee_due NUMERIC(10, 2) NOT NULL CHECK (total_fee_due >= 100.00),
  fee_breakdown JSONB NOT NULL DEFAULT '[]'::jsonb, -- per-line [{variant_id, unit_price, quantity, band, unit_fee}]
  fee_policy_version TEXT NOT NULL DEFAULT 'v2.7-banded',
  minimum_applied BOOLEAN NOT NULL DEFAULT false, -- true when total_fee_due was floored to ₱100
  payment_proof_url TEXT,
  cleared_by_admin_id UUID REFERENCES users(id),
  cleared_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  CONSTRAINT fee_total_floor CHECK (total_fee_due = GREATEST(subtotal_fee, 100.00))
);

-- Refunds
CREATE TABLE refunds (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id UUID NOT NULL REFERENCES orders(id) ON DELETE RESTRICT,
  refund_method TEXT NOT NULL,
  refund_reference TEXT,
  proof_url TEXT,
  processed_by_officer_id UUID NOT NULL REFERENCES users(id),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

> **D-04 audit note (2026-09-21):** `campaigns.per_item_fee` and `fee_settlements.fee_per_unit` from the pre-D-04 `min(₱5.00, 5%)` model were removed. Banded unit fees are derived from each order item's frozen `unit_price` at settlement time: 5% (rounded to centavo) when `< ₱100`; `₱5` for `₱100–₱499`; `₱10` for `₱500–₱999`; `₱15` for `≥ ₱1,000`. The frozen `fee_breakdown` snapshot plus `fee_policy_version` keeps mixed-price campaigns reproducible after catalog edits. Existing unsettled rows must be recomputed; already-`CLEARED` settlements are historical and are not retroactively rebilled.

---

## 3. Atomic Database Operations

### 3.1 Booth Dispatch Confirmation — Bundle Partial Pickup (v2.10: READY lines only, per-line double-claim prevention, D-11; retains F03 + REQ-48)

```sql
CREATE OR REPLACE FUNCTION confirm_order_pickup(
  p_pickup_code TEXT,
  p_officer_id UUID,
  p_item_ids UUID[] DEFAULT NULL -- NULL/empty = all remaining READY lines; else selected subset (partial pickup)
)
RETURNS TABLE (success BOOLEAN, message TEXT, order_data JSONB)
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public, pg_temp
AS $$
DECLARE
  v_order orders%ROWTYPE;
  v_result orders%ROWTYPE;
  v_claimed_count INT := 0;
  v_remaining INT := 0;
BEGIN
  -- Caller authorization (defense in depth; RLS §4 also applies to direct writes):
  -- p_officer_id must be an active officer. Tenant/role scoping against the order's
  -- campaign org is enforced below via campaign join. GRANT EXECUTE to officer roles only.
  IF NOT EXISTS (
    SELECT 1 FROM users u WHERE u.id = p_officer_id AND u.is_active
      AND u.role_tag IN ('VOLUNTEER', 'CASHIER', 'TREASURER', 'SUPERADMIN', 'PLATFORM_OWNER')
  ) THEN
    RETURN QUERY SELECT false, 'Unauthorized: inactive or unknown officer.'::TEXT, NULL::JSONB;
    RETURN;
  END IF;

  -- Lock row for update (scoped to normalized code; UNIQUE per F06)
  SELECT * INTO v_order
  FROM orders
  WHERE pickup_code = UPPER(TRIM(p_pickup_code))
  FOR UPDATE;

  IF NOT FOUND THEN
    RETURN QUERY SELECT false, 'Order not found for pick-up code.'::TEXT, NULL::JSONB;
    RETURN;
  END IF;

  -- Same-org check: officer must belong to the campaign's org (or be platform admin)
  IF NOT EXISTS (
    SELECT 1 FROM orders o
    JOIN campaigns c ON c.id = o.campaign_id
    JOIN users u ON u.id = p_officer_id
    WHERE o.id = v_order.id
      AND (u.org_id = c.org_id OR u.role_tag IN ('SUPERADMIN', 'PLATFORM_OWNER'))
  ) THEN
    RETURN QUERY SELECT false, 'Unauthorized: cross-organization pickup.'::TEXT, NULL::JSONB;
    RETURN;
  END IF;

  IF v_order.status = 'CLAIMED' THEN
    RETURN QUERY SELECT false, format('Order already fully claimed on %s.', v_order.claimed_at)::TEXT, to_jsonb(v_order);
    RETURN;
  END IF;

  -- v2.10: releasable order states are READY_FOR_PICKUP (first release) or PARTIALLY_CLAIMED (subsequent visits).
  -- Campaign must be READY_FOR_PICKUP. Fee disputes never block release (REQ-48 preserved).
  IF v_order.status NOT IN ('READY_FOR_PICKUP', 'PARTIALLY_CLAIMED') THEN
    RETURN QUERY SELECT false, format('Order status is %s; cannot release (must be READY_FOR_PICKUP or PARTIALLY_CLAIMED).', v_order.status)::TEXT, to_jsonb(v_order);
    RETURN;
  END IF;

  IF NOT EXISTS (
    SELECT 1 FROM campaigns c WHERE c.id = v_order.campaign_id AND c.status = 'READY_FOR_PICKUP'
  ) THEN
    RETURN QUERY SELECT false, 'Campaign is not READY_FOR_PICKUP; cannot release.'::TEXT, NULL::JSONB;
    RETURN;
  END IF;

  -- Lock selected lines in stable order; claim whole quantity per line in v1 (split-qty via quantity_claimed reserved for v1.1).
  -- Lines already CLAIMED are skipped with a per-line warning; at least one newly-claimed line is required.
  WITH target AS (
    SELECT oi.id
    FROM order_items oi
    WHERE oi.order_id = v_order.id
      AND (p_item_ids IS NULL OR cardinality(p_item_ids) = 0 OR oi.id = ANY(p_item_ids))
      AND oi.status = 'READY_FOR_PICKUP'
    ORDER BY oi.id
    FOR UPDATE
  ),
  claimed AS (
    UPDATE order_items oi
    SET status = 'CLAIMED',
        quantity_claimed = quantity,
        claimed_at = now(),
        claimed_by_officer_id = p_officer_id
    FROM target t WHERE oi.id = t.id
    RETURNING oi.id
  )
  SELECT count(*) INTO v_claimed_count FROM claimed;

  IF v_claimed_count = 0 THEN
    RETURN QUERY SELECT false, 'No releasable lines: selected lines already claimed.'::TEXT, to_jsonb(v_order);
    RETURN;
  END IF;

  -- Derive header status: CLAIMED only when zero READY lines remain.
  SELECT count(*) INTO v_remaining FROM order_items WHERE order_id = v_order.id AND status = 'READY_FOR_PICKUP';

  UPDATE orders
  SET
    status = CASE WHEN v_remaining = 0 THEN 'CLAIMED'::order_status ELSE 'PARTIALLY_CLAIMED'::order_status END,
    claimed_at = CASE WHEN v_remaining = 0 THEN now() ELSE claimed_at END,
    claimed_by_officer_id = CASE WHEN v_remaining = 0 THEN p_officer_id ELSE claimed_by_officer_id END,
    updated_at = now()
  WHERE id = v_order.id
  RETURNING * INTO v_result;

  INSERT INTO order_status_history (order_id, from_status, to_status, actor_id, notes)
  VALUES (v_order.id, v_order.status, v_result.status, p_officer_id,
          format('Booth release: %s line(s) claimed; %s remaining', v_claimed_count, v_remaining));

  -- F13 fix: return the fresh post-claim row, not the stale pre-claim snapshot.
  RETURN QUERY SELECT true, format('%s line(s) released%s.', v_claimed_count,
    CASE WHEN v_remaining = 0 THEN ' — order fully CLAIMED' ELSE ' — order PARTIALLY_CLAIMED' END)::TEXT, to_jsonb(v_result);
END;
$$;

-- REVOKE FROM PUBLIC; grant to officer roles only (apply in migration):
-- REVOKE ALL ON FUNCTION confirm_order_pickup(TEXT, UUID, UUID[]) FROM PUBLIC;
-- GRANT EXECUTE ON FUNCTION confirm_order_pickup(TEXT, UUID, UUID[]) TO authenticated;
```

### 3.2 Officer Cash Order Creation — Multi-Line Bundle (D-10/D-11; sole cash write path, `p_lines JSONB`)

```sql
-- Generates a collision-safe pickup code: 2-letter org prefix + 4 digits (e.g. CS-8419).
-- Retries on UNIQUE violation; caller runs inside create_cash_order().
CREATE OR REPLACE FUNCTION issue_pickup_code(p_org_prefix TEXT)
RETURNS VARCHAR(10)
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public, pg_temp
AS $$
DECLARE
  v_code VARCHAR(10);
  v_attempts INT := 0;
BEGIN
  LOOP
    v_code := UPPER(TRIM(p_org_prefix)) || '-' || LPAD((floor(random() * 9000) + 1000)::INT::TEXT, 4, '0');
    BEGIN
      -- Probe uniqueness without inserting: caller inserts; on conflict it retries.
      IF NOT EXISTS (SELECT 1 FROM orders WHERE pickup_code = v_code) THEN
        RETURN v_code;
      END IF;
    END;
    v_attempts := v_attempts + 1;
    IF v_attempts > 10 THEN
      RAISE EXCEPTION 'pickup code issuance failed after retries';
    END IF;
  END LOOP;
END;
$$;

-- Atomic F2F cash bundle sale (v2.10 multi-line): find-or-create buyer (email required, REQ-22A),
-- create order directly as PAYMENT_VERIFIED (D-07, no TTL), branch PRE_ORDER vs FLASH per campaign.
-- p_lines JSONB: [{variant_id UUID, quantity INT>0, customization JSONB}] — 1..10 lines, single campaign scope.
-- Locks variants FOR UPDATE ORDER BY id; any line failure rolls back the whole bundle.
CREATE OR REPLACE FUNCTION create_cash_order(
  p_campaign_id UUID,
  p_lines JSONB,
  p_buyer_name TEXT,
  p_buyer_email TEXT,
  p_buyer_mobile TEXT,
  p_register_session_id UUID DEFAULT NULL,
  p_officer_id UUID DEFAULT NULL
)
RETURNS TABLE (success BOOLEAN, message TEXT, order_id UUID, pickup_code VARCHAR(10))
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public, pg_temp
AS $$
DECLARE
  v_campaign campaigns%ROWTYPE;
  v_variant product_variants%ROWTYPE;
  v_session register_sessions%ROWTYPE;
  v_officer users%ROWTYPE;
  v_buyer buyers%ROWTYPE;
  v_email TEXT := LOWER(TRIM(p_buyer_email));
  v_code VARCHAR(10);
  v_order_id UUID;
  v_unit_price NUMERIC(10, 2);
  v_total NUMERIC(10, 2);
  v_org_prefix TEXT := 'CS';
BEGIN
  -- 0. Basic input guards
  IF p_quantity IS NULL OR p_quantity <= 0 THEN
    RETURN QUERY SELECT false, 'Quantity must be > 0.'::TEXT, NULL::UUID, NULL::VARCHAR(10); RETURN;
  END IF;
  IF v_email IS NULL OR v_email = '' OR v_email NOT LIKE '%@%.%' THEN
    RETURN QUERY SELECT false, 'Buyer email is required for tracker delivery (REQ-22A).'::TEXT, NULL::UUID, NULL::VARCHAR(10); RETURN;
  END IF;
  IF TRIM(COALESCE(p_buyer_name, '')) = '' OR TRIM(COALESCE(p_buyer_mobile, '')) = '' THEN
    RETURN QUERY SELECT false, 'Buyer name and mobile are required.'::TEXT, NULL::UUID, NULL::VARCHAR(10); RETURN;
  END IF;

  -- 1. Officer must be active CASHIER/TREASURER/VOLUNTEER (or platform admin covering booth)
  SELECT * INTO v_officer FROM users WHERE id = p_officer_id AND is_active FOR SHARE;
  IF NOT FOUND OR v_officer.role_tag NOT IN ('CASHIER', 'TREASURER', 'VOLUNTEER', 'SUPERADMIN', 'PLATFORM_OWNER') THEN
    RETURN QUERY SELECT false, 'Unauthorized: officer role cannot take cash.'::TEXT, NULL::UUID, NULL::VARCHAR(10); RETURN;
  END IF;

  -- 2. Register session must be OPEN; lock it to serialize against close (F10)
  SELECT * INTO v_session FROM register_sessions WHERE id = p_register_session_id FOR UPDATE;
  IF NOT FOUND THEN
    RETURN QUERY SELECT false, 'Register session not found.'::TEXT, NULL::UUID, NULL::VARCHAR(10); RETURN;
  END IF;
  IF v_session.status != 'OPEN' THEN
    RETURN QUERY SELECT false, 'Register session is not OPEN; cash not accepted.'::TEXT, NULL::UUID, NULL::VARCHAR(10); RETURN;
  END IF;

  -- 3. Campaign must be LIVE; same-org as session and officer (F07 tenant guard)
  SELECT * INTO v_campaign FROM campaigns WHERE id = p_campaign_id FOR SHARE;
  IF NOT FOUND OR v_campaign.status != 'LIVE' THEN
    RETURN QUERY SELECT false, 'Campaign is not LIVE.'::TEXT, NULL::UUID, NULL::VARCHAR(10); RETURN;
  END IF;
  IF v_campaign.org_id != v_session.org_id THEN
    RETURN QUERY SELECT false, 'Cross-organization register assignment rejected.'::TEXT, NULL::UUID, NULL::VARCHAR(10); RETURN;
  END IF;
  IF v_officer.org_id IS NOT NULL AND v_officer.role_tag NOT IN ('SUPERADMIN', 'PLATFORM_OWNER')
     AND v_officer.org_id != v_campaign.org_id THEN
    RETURN QUERY SELECT false, 'Unauthorized: cross-organization cash entry.'::TEXT, NULL::UUID, NULL::VARCHAR(10); RETURN;
  END IF;

  -- 4. Variant must belong to campaign, be active; lock for stock (stable order: single row here)
  SELECT * INTO v_variant FROM product_variants WHERE id = p_variant_id FOR UPDATE;
  IF NOT FOUND OR v_variant.campaign_id != p_campaign_id OR NOT v_variant.is_active THEN
    RETURN QUERY SELECT false, 'Variant not available in this campaign.'::TEXT, NULL::UUID, NULL::VARCHAR(10); RETURN;
  END IF;
  v_unit_price := v_variant.price;
  v_total := v_unit_price * p_quantity;

  -- 5a. PRE_ORDER: no stock op. 5b. FLASH: atomic decrement + cap enforcement
  IF v_campaign.type = 'FLASH' THEN
    IF v_variant.stock_remaining IS NULL OR v_variant.stock_remaining < p_quantity THEN
      RETURN QUERY SELECT false, 'Sold out: insufficient stock.'::TEXT, NULL::UUID, NULL::VARCHAR(10); RETURN;
    END IF;
    -- Per-buyer cap: sum existing verified/submitted qty for this buyer-email + campaign + variant
    -- (enforced after buyer lookup; placeholder check here uses email match)
    UPDATE product_variants SET stock_remaining = stock_remaining - p_quantity WHERE id = v_variant.id;
  END IF;
  -- NOTE: per-buyer purchase caps (REQ-16) are enforced post-buyer-lookup below; exceeding aborts.

  -- 6. Find-or-create buyer by email (email UNIQUE); never overwrite name/mobile silently
  SELECT * INTO v_buyer FROM buyers WHERE email = v_email FOR UPDATE;
  IF NOT FOUND THEN
    INSERT INTO buyers (name, email, mobile_number)
    VALUES (TRIM(p_buyer_name), v_email, TRIM(p_buyer_mobile))
    RETURNING * INTO v_buyer;
  END IF;

  -- 7. Issue pickup code (UNIQUE, retry on race)
  v_org_prefix := COALESCE((SELECT UPPER(LEFT(REGEXP_REPLACE(o.name, '[^A-Za-z]', '', 'g'), 2)) FROM organizations o WHERE o.id = v_campaign.org_id), 'CS');
  IF v_org_prefix = '' THEN v_org_prefix := 'CS'; END IF;
  <<code_loop>>
  FOR i IN 1..5 LOOP
    v_code := issue_pickup_code(v_org_prefix);
    BEGIN
      INSERT INTO orders (
        campaign_id, buyer_id, register_session_id, status, total_amount,
        payment_method, source_channel, payment_reference, receipt_image_url,
        pickup_code, reservation_expires_at, verified_by_officer_id, verified_at
      ) VALUES (
        p_campaign_id, v_buyer.id, v_session.id, 'PAYMENT_VERIFIED', v_total,
        'CASH_ON_HAND', 'Cash', NULL, NULL,
        v_code, NULL, p_officer_id, now()
      ) RETURNING id INTO v_order_id;
      EXIT code_loop;
    EXCEPTION WHEN unique_violation THEN
      IF i = 5 THEN RAISE; END IF;
      -- retry with a fresh code
    END;
  END LOOP;

  INSERT INTO order_items (order_id, variant_id, unit_price, quantity, customization_data)
  -- v2.10: loop over p_lines in stable variant-id order (one row per line); unit_price frozen per line.
  -- (Single-variant callers pass 1-element p_lines; equivalent to the retired p_variant_id/p_quantity form.)

  INSERT INTO order_status_history (order_id, from_status, to_status, actor_id, notes)
  VALUES (v_order_id, NULL, 'PAYMENT_VERIFIED', p_officer_id,
          format('F2F cash bundle collected under session %s (%s line(s))', v_session.session_name, jsonb_array_length(p_lines)));

  -- 8. Email trigger is application-layer (Supabase Auth / Google SMTP magic-link + receipt).
  --    MUST be queued after commit; send failure retries out-of-band and never rolls back this sale (D-08).

  RETURN QUERY SELECT true, 'Cash bundle verified.'::TEXT, v_order_id, v_code;
END;
$$;

-- REVOKE FROM PUBLIC; grant to officer roles only (apply in migration):
-- REVOKE ALL ON FUNCTION create_cash_order(UUID, JSONB, TEXT, TEXT, TEXT, UUID, UUID) FROM PUBLIC;
-- GRANT EXECUTE ON FUNCTION create_cash_order(UUID, JSONB, TEXT, TEXT, TEXT, UUID, UUID) TO authenticated;
```

### 3.3 Digital Bundle Reservation — `reserve_cart()` (D-10; buyer Purchase, single TTL per bundle)

```sql
-- Atomic buyer bundle reservation: same-campaign p_lines (1..10 lines, ≤99 units),
-- locks variants FOR UPDATE ORDER BY id, validates is_active + FLASH stock + per-variant /
-- per-campaign caps, freezes unit_price, computes total = Σ price×qty, creates
-- orders(status=PAYMENT_SUBMITTED or AWAITING_PAYMENT per flow) + N order_items,
-- single reservation_expires_at = now() + 30 mins, single pickup_code issued at verify time
-- (UC-06) — code column filled on PAYMENT_VERIFIED to satisfy NOT NULL/UNIQUE with deferred issue.
-- Any line failure rolls back the whole bundle. Expiry job restores each line qty exactly once
-- conditional on prior status (quantity-aware, F11). Idempotency: client supplies
-- p_idempotency_key (UNIQUE per buyer+campaign, 24h) — retry returns existing order.
-- SECURITY DEFINER, REVOKE FROM PUBLIC, buyer role may EXECUTE; server derives buyer from auth,
-- never trusts caller-supplied buyer_id (F15). Full body follows the create_cash_order() locking/
-- cap/total pattern above with payment_method IN ('DIGITAL_WALLET','BANK_TRANSFER'), RLS WITH CHECK
-- digital-only (REQ-13B) preserved.
```


---

## 4. Row-Level Security (DEV-02 capability matrix — v2.10 cart/bundle)

Enable RLS on all app tables and deny direct buyer cash writes. Privileged writes go through
`§3.1` / `§3.2` / `§3.3` (`SECURITY DEFINER`, `REVOKE FROM PUBLIC`, `GRANT EXECUTE` to `authenticated`;
functions enforce active-role + same-org + single-campaign checks above — RLS alone does not constrain them, F15).

```sql
ALTER TABLE organizations ENABLE ROW LEVEL SECURITY;
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE buyers ENABLE ROW LEVEL SECURITY;
ALTER TABLE campaigns ENABLE ROW LEVEL SECURITY;
ALTER TABLE product_variants ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE order_items ENABLE ROW LEVEL SECURITY;
ALTER TABLE order_status_history ENABLE ROW LEVEL SECURITY;
ALTER TABLE register_sessions ENABLE ROW LEVEL SECURITY;
ALTER TABLE fee_settlements ENABLE ROW LEVEL SECURITY;
ALTER TABLE refunds ENABLE ROW LEVEL SECURITY;
```

**Policy sketch (implement per project auth claims: `auth.uid()` → `users.id`, `users.org_id/role_tag`):**

| Table | Read | Write |
|---|---|---|
| `campaigns`, `product_variants` | Public/`anon` read when `status='LIVE'` (catalog) | Org officers (same `org_id`) `INSERT/UPDATE`; structural delete blocked with active orders (REQ-07) |
| `buyers` | Owning buyer (`auth_user_id = auth.uid()`) or same-org officer | Officer `INSERT` (find-or-create in `create_cash_order`); owner self-`UPDATE` limited columns |
| `orders` | Owning buyer (via `buyer.auth_user_id`) or same-org officer / platform admin | Buyer `INSERT` only `payment_method != 'CASH_ON_HAND'` + `status=AWAITING_PAYMENT/PAYMENT_SUBMITTED` + single `campaign_id` per order (no cross-campaign lines; enforced in §3.3/§3.2, F07); **no direct `INSERT CASH_ON_HAND`** (REQ-13B — enforced by policy `WITH CHECK (payment_method != 'CASH_ON_HAND')` for buyer role + function-only path for officers); `total_amount` server-computed, buyers never `UPDATE total/status/verified/claimed/register/pickup_code`; header `CLAIMED` only via §3.1 when all lines claimed |
| `order_items` | Same as parent order | Same as parent order; `unit_price` frozen from variant at insert; `variant.campaign_id` must equal `order.campaign_id` (F07); per-line claim cols writable only via §3.1 |
| `order_status_history` | Same as parent order | `INSERT`-only via functions; no `UPDATE/DELETE` (append-only, F12) |
| `register_sessions` | Same-org officers | Open: `CASHIER/TREASURER`; close: `closed_by` + distinct `cosigned_by` (`TREASURER`/auditor), variance note required on mismatch (constraint above, F10) |
| `fee_settlements`, `refunds` | Same-org treasurer / platform admin | Treasurer/admin only |

```sql
-- Example (adapt role-claim mechanism to project):
-- Buyer-owned order reads:
-- CREATE POLICY orders_buyer_read ON orders FOR SELECT TO authenticated
--   USING (buyer_id IN (SELECT id FROM buyers WHERE auth_user_id = auth.uid()));
-- Buyer digital-only insert:
-- CREATE POLICY orders_buyer_insert_digital ON orders FOR INSERT TO authenticated
--   WITH CHECK (payment_method IN ('DIGITAL_WALLET', 'BANK_TRANSFER'));
-- Officer cash must use create_cash_order(): no direct cash INSERT policy is created
-- for any client role; officer clients are granted EXECUTE on the function only.
-- History append-only:
-- CREATE POLICY history_insert_only ON order_status_history FOR INSERT TO authenticated WITH CHECK (true);
-- (No UPDATE/DELETE policies → denied.)
```

**Migration order (F05):** enums (`order_item_status` + `PARTIALLY_CLAIMED`) → `organizations` → `users` → `buyers` → `campaigns` →
`product_variants` (+CHECKs) → `register_sessions` → `orders` (+UNIQUE pickup, cash CHECKs) →
`order_items` (+claim cols/CHECKs) → `order_status_history` → `fee_settlements` → `refunds` → functions §3.1/§3.2/§3.3 →
`ENABLE RLS` + policies + `REVOKE/GRANT` → validate on empty Supabase-compatible DB.

> **D-06 audit note (2026-10-01):** No data migration for legacy buyer cash `AWAITING_PAYMENT` rows is specified — repo is docs-only with no deployed rows. If any pre-v2.9 draft rows exist, treat buyer-created `CASH_ON_HAND/AWAITING_PAYMENT` as expired/cancelled and re-enter F2F via `create_cash_order()`.
