# Database Schema Definition (Supabase / PostgreSQL)
## Project Ebisu — Campus Merch Platform

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

-- Order lifecycle
CREATE TYPE order_status AS ENUM (
  'AWAITING_PAYMENT',
  'PAYMENT_SUBMITTED',
  'PAYMENT_VERIFIED',
  'IN_PRODUCTION',
  'READY_FOR_PICKUP',
  'CLAIMED',
  'REJECTED',
  'CANCELLED',
  'REFUNDED'
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
  per_item_fee NUMERIC(10, 2) NOT NULL DEFAULT 5.00,
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

### 2.3 Orders & Transaction History

```sql
-- Orders
CREATE TABLE orders (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  campaign_id UUID NOT NULL REFERENCES campaigns(id) ON DELETE RESTRICT,
  buyer_id UUID NOT NULL REFERENCES buyers(id) ON DELETE RESTRICT,
  register_session_id UUID REFERENCES register_sessions(id) ON DELETE SET NULL,
  status order_status NOT NULL DEFAULT 'AWAITING_PAYMENT',
  total_amount NUMERIC(10, 2) NOT NULL,
  payment_method payment_method NOT NULL,
  source_channel TEXT, -- e.g. GCash, Maya, GoTyme, BDO, Cash
  payment_reference TEXT UNIQUE, -- Global deduplication constraint (REQ-14)
  receipt_image_url TEXT,
  rejection_reason TEXT,
  pickup_code VARCHAR(10) NOT NULL, -- e.g. "CS-8419"
  reservation_expires_at TIMESTAMPTZ,
  verified_by_officer_id UUID REFERENCES users(id),
  verified_at TIMESTAMPTZ,
  claimed_by_officer_id UUID REFERENCES users(id),
  claimed_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Order Items & Custom Personalization
CREATE TABLE order_items (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  variant_id UUID NOT NULL REFERENCES product_variants(id) ON DELETE RESTRICT,
  unit_price NUMERIC(10, 2) NOT NULL,
  quantity INT NOT NULL DEFAULT 1,
  customization_data JSONB NOT NULL DEFAULT '{}'::jsonb -- e.g. {"back_name": "DELA CRUZ", "back_number": "24"}
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
  expected_cash NUMERIC(10, 2) NOT NULL DEFAULT 0.00,
  counted_cash NUMERIC(10, 2),
  variance_note TEXT,
  opened_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  closed_at TIMESTAMPTZ
);

-- Platform Fee Settlements
CREATE TABLE fee_settlements (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  campaign_id UUID NOT NULL REFERENCES campaigns(id) ON DELETE CASCADE,
  total_units_sold INT NOT NULL,
  fee_per_unit NUMERIC(10, 2) NOT NULL,
  total_fee_due NUMERIC(10, 2) NOT NULL,
  payment_proof_url TEXT,
  cleared_by_admin_id UUID REFERENCES users(id),
  cleared_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
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

---

## 3. Atomic Database Operations

### 3.1 Booth Dispatch Confirmation (Double-Claim Prevention)
```sql
CREATE OR REPLACE FUNCTION confirm_order_pickup(
  p_pickup_code TEXT,
  p_officer_id UUID
)
RETURNS TABLE (success BOOLEAN, message TEXT, order_data JSONB)
LANGUAGE plpgsql
SECURITY DEFINER
AS $$
DECLARE
  v_order orders%ROWTYPE;
BEGIN
  -- Lock row for update
  SELECT * INTO v_order
  FROM orders
  WHERE pickup_code = UPPER(TRIM(p_pickup_code))
  FOR UPDATE;

  IF NOT FOUND THEN
    RETURN QUERY SELECT false, 'Order not found for pick-up code.'::TEXT, NULL::JSONB;
    RETURN;
  END IF;

  IF v_order.status = 'CLAIMED' THEN
    RETURN QUERY SELECT false, format('Order already claimed on %s.', v_order.claimed_at)::TEXT, to_jsonb(v_order);
    RETURN;
  END IF;

  IF v_order.status != 'READY_FOR_PICKUP' AND v_order.status != 'PAYMENT_VERIFIED' THEN
    RETURN QUERY SELECT false, format('Order status is %s; cannot release.', v_order.status)::TEXT, to_jsonb(v_order);
    RETURN;
  END IF;

  UPDATE orders
  SET
    status = 'CLAIMED',
    claimed_at = now(),
    claimed_by_officer_id = p_officer_id,
    updated_at = now()
  WHERE id = v_order.id;

  INSERT INTO order_status_history (order_id, from_status, to_status, actor_id, notes)
  VALUES (v_order.id, v_order.status, 'CLAIMED', p_officer_id, 'Merchandise released at booth terminal');

  RETURN QUERY SELECT true, 'Merchandise successfully released.'::TEXT, to_jsonb(v_order);
END;
$$;
```
