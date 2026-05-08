# Miinto Partner Portal — System Architecture

> **Version**: 2.2 — NTS Readiness Gates added (2026-05-05 strategic re-evaluation)
> **Architectural Shift**: Companion App is now the **central hub** for the partner lifecycle. The legacy "Partner Portal" splits into a thin marketing site and a module-based Companion App that hosts Contract Signing, Onboarding Wizard, Training, and Support.
> **Iteration 4 additions**: API contracts (§13), webhook spec (§14), idempotency (§15), rate limiting (§16), dashboards & SLOs (§17), disaster recovery (§18), compliance (§19).

---

## 1. High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                             PUBLIC SURFACE                                    │
│  ┌──────────────────┐                          ┌──────────────────────────┐ │
│  │  Marketing Site  │   ───── apply ────▶      │   Auth Gateway (OAuth2)  │ │
│  │  (Next.js / SSG) │                          │   JWT issuer + 2FA       │ │
│  └──────────────────┘                          └──────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                  ★ COMPANION APP — CENTRAL PARTNER HUB ★                     │
│                       (Web PWA + iOS/Android via React Native)               │
│                                                                              │
│   ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐             │
│   │  Module:        │  │  Module:        │  │  Module:        │             │
│   │  Onboarding     │  │  Contract       │  │  Training       │             │
│   │  Wizard         │  │  Signing        │  │  Academy        │             │
│   │  (6 steps)      │  │  (DocuSign)     │  │  (LMS)          │             │
│   └─────────────────┘  └─────────────────┘  └─────────────────┘             │
│                                                                              │
│   ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐             │
│   │  Module:        │  │  Module:        │  │  Module:        │             │
│   │  Support /      │  │  Operations     │  │  Settings       │             │
│   │  Help Desk      │  │  Dashboard      │  │  & API Keys     │             │
│   └─────────────────┘  └─────────────────┘  └─────────────────┘             │
│                                                                              │
│   Shared Container Services: Module Registry · Feature Flags · Analytics    │
└──────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                         BACKEND-FOR-FRONTEND (BFF)                            │
│            NestJS · GraphQL gateway · per-module REST adapters               │
└──────────────────────────────────────────────────────────────────────────────┘
                                       │
        ┌──────────────────────────────┼──────────────────────────────┐
        ▼                              ▼                              ▼
┌──────────────────┐          ┌──────────────────┐          ┌──────────────────┐
│ DOMAIN SERVICES  │          │ MIINTO CORE APIs │          │  EXTERNAL SaaS   │
│                  │          │                  │          │                  │
│ • Auth           │          │ • POAS v1 (legacy│          │ • DocuSign       │
│ • Onboarding     │          │   orders) ⚠️      │          │ • Clearbit       │
│ • KYC            │          │ • POAS v2.5 ✅   │          │ • VIES (VAT)     │
│ • Contract       │          │   (orders + RMA)│          │ • Onfido (KYC)   │
│ • Partner        │          │ • NTS (2025) 🔄 │          │ • SendGrid       │
│ • Training       │          │   transactional │          │ • Intercom       │
│ • Notification   │          │   migration     │          │ • Stripe         │
│ • Support        │          │                  │          │ • Slack          │
└──────────────────┘          └──────────────────┘          └──────────────────┘
                                       │
                                       ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  DATA LAYER:  PostgreSQL (primary) · Redis (cache/queue) · S3 (docs)        │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Companion App — The Central Hub

The Companion App replaces the previous "portal-as-monolith" model. It is a **module-based container** that ships as both a Web PWA and a native React Native app from a shared codebase. Every partner-facing surface — onboarding, contracts, training, support, day-to-day operations — runs as a module inside this single shell.

### 2.1 Why a single hub

| Driver | Detail |
|---|---|
| Single identity | One JWT session covers every partner-facing surface; no rebadging across portals. |
| Lifecycle continuity | A partner moves from prospect → onboarded → operating → escalating support without ever switching apps. |
| Mobile-first | Polish/SE/DK partners increasingly manage orders on phones; native app required for push and barcode flows. |
| Iteration speed | Modules ship independently behind feature flags; container team owns shell, module teams own modules. |

### 2.2 Container architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Companion App Shell                           │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Shell Services (always loaded)                           │  │
│  │  ├─ Auth & Session (JWT + refresh + biometric on mobile)  │  │
│  │  ├─ Module Registry (lazy-load + capability declaration)  │  │
│  │  ├─ Routing (deep links + universal links)                │  │
│  │  ├─ Telemetry (Datadog RUM + product analytics)           │  │
│  │  ├─ Notification Center (in-app + push)                   │  │
│  │  ├─ Feature Flags (LaunchDarkly)                          │  │
│  │  ├─ Offline cache (RxDB / IndexedDB)                      │  │
│  │  └─ Theming + i18n (8 locales)                            │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              │                                   │
│  ┌───────────────────────────▼───────────────────────────────┐  │
│  │  Module Bus  (typed events · capability requests)         │  │
│  └───────────────────────────────────────────────────────────┘  │
│        │           │           │           │           │        │
│  ┌─────▼────┐ ┌────▼────┐ ┌────▼────┐ ┌────▼────┐ ┌────▼────┐ │
│  │Onboarding│ │Contract │ │Training │ │Support  │ │Ops      │ │
│  │ Wizard   │ │Signing  │ │Academy  │ │Help Desk│ │Dashboard│ │
│  └──────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### 2.3 Module contract

Every module declares a manifest the shell consumes at registration time:

```typescript
interface ModuleManifest {
  id: 'onboarding' | 'contract' | 'training' | 'support' | 'ops' | 'settings';
  version: string;                  // semver, independently deployable
  routes: RouteDescriptor[];        // deep-link + nav entries
  requiredCapabilities: Capability[]; // 'camera', 'push', 'biometric', 'file-upload'
  requiredScopes: AuthScope[];      // RBAC gates from token
  featureFlag?: string;             // optional gate for staged rollout
  entryPoint: () => Promise<ModuleExports>; // dynamic import
}
```

The shell loads modules lazily. A first-day partner pays only for the Onboarding + Contract bundles; Training and Ops load on demand.

### 2.4 Module responsibilities

| Module | Owns | Backend Service |
|---|---|---|
| **Onboarding Wizard** | 6-step flow (company, contact, KYC, contract handoff, tech setup, go-live) with persistent draft state | Onboarding Service |
| **Contract Signing** | Embedded DocuSign experience, terms acknowledgment, countersigned PDF download, audit trail | Contract Service |
| **Training Academy** | Video modules, interactive labs, quizzes, certification, progress tracking, sandbox launches | Training Service |
| **Support / Help Desk** | Knowledge base search, ticket CRUD, Intercom live chat, community forum, escalation to AM | Support Service + Intercom |
| **Operations Dashboard** | Orders, products, returns, inventory, analytics — the day-to-day work surface | Partner Service + POAS v2.5 |
| **Settings** | Profile, team, API keys, webhooks, notification preferences, billing | Partner Service |

### 2.5 Mobile vs Web parity

| Capability | Web PWA | iOS/Android |
|---|---|---|
| Onboarding wizard | ✅ Full | ✅ Full |
| Contract signing | ✅ Embedded DocuSign | ✅ DocuSign SDK + biometric confirm |
| Training (video) | ✅ HLS player | ✅ Native player + offline download |
| Push notifications | ⚠️ Web Push (limited iOS) | ✅ APNs / FCM |
| Barcode/QR for stock | ❌ | ✅ Camera-based |
| Document upload | ✅ File picker | ✅ Camera capture + file picker |
| Offline | ⚠️ Read-only cache | ✅ Full offline drafts |

---

## 3. Frontend Tech Stack

| Concern | Choice |
|---|---|
| Web shell | Next.js 14 (App Router) compiled as PWA |
| Mobile shell | React Native 0.74 + Expo (managed workflow) |
| Shared UI | `@miinto/ui` (Tailwind tokens → React Native via NativeWind) |
| State | Zustand (per-module slice) + React Query (server state) |
| Forms | React Hook Form + Zod schemas (shared with BFF) |
| Auth | NextAuth (web) / `expo-auth-session` (native) — same OAuth2 issuer |
| Telemetry | Datadog RUM + Mixpanel events |
| Feature flags | LaunchDarkly |

---

## 4. Backend Architecture

### 4.1 Tech stack

- **Runtime**: Node.js 20 LTS
- **Framework**: NestJS (one app per service, shared libs via Nx)
- **Database**: PostgreSQL 16 (primary) + Redis 7 (cache, sessions, BullMQ)
- **ORM**: Prisma
- **API**: GraphQL gateway (BFF) → REST microservices
- **Async**: BullMQ on Redis; long-running jobs on AWS Step Functions

### 4.2 Domain services

| Service | Responsibility | Key Integrations |
|---|---|---|
| Auth | Login, JWT issue/refresh, 2FA (TOTP + WebAuthn), session, password reset | — |
| Onboarding | Wizard state machine, draft persistence, progress analytics | Notification |
| KYC | Company + owner verification, AML/PEP/sanctions, risk scoring | Clearbit, VIES, Onfido |
| Contract | Template rendering, DocuSign envelopes, versioning, archival, audit | DocuSign, S3 |
| Partner | Profile, API keys, webhooks, team management | POAS v2.5, NTS |
| Training | LMS content, progress, quizzes, certification | S3 (video), Mux |
| Support | Tickets, KB, escalation, SLA tracking | Intercom, Zendesk |
| Notification | Email, push, SMS, in-app | SendGrid, APNs/FCM, Twilio |

### 4.3 Onboarding state machine

```
[START] → [COMPANY_INFO] → [CONTACT_INFO] → [KYC]
            │                   │             │
            └─ saveable draft ──┴─ saveable ──┘
                                              │
                                              ▼
[KYC_APPROVED] → [CONTRACT] → [TECH_SETUP] → [TRAINING] → [GO_LIVE] → [COMPLETE]
                     │                            │
                     └─ DocuSign callback         └─ sandbox checklist gate
```

State transitions emit `partner.onboarding.*` events on the internal bus; downstream services react idempotently.

---

## 5. Miinto Core API Layer

### 5.1 POAS v2.5 (current)

POAS v2.5 is the active order-and-RMA API. The Partner Portal targets v2.5 exclusively for new development; v1 is in maintenance only.

**Base URL**: `https://api.miinto.com/poas/v2.5`
**Auth**: OAuth2 client credentials → bearer token (15 min) + HMAC signature header (see §7.4).

| Method | Endpoint | Purpose |
|---|---|---|
| `POST` | `/orders` | Create / receive order from Miinto core |
| `GET` | `/orders/:id` | Retrieve single order |
| `GET` | `/orders?status=&from=&to=` | Paginated order list |
| `POST` | `/orders/:id/accept` | Partner accepts order |
| `POST` | `/orders/:id/reject` | Partner rejects with reason code |
| `POST` | `/orders/:id/ship` | Mark shipped + tracking number |
| `POST` | `/orders/:id/cancel` | Partner-initiated cancellation |
| `GET`  | `/products` | Catalog read |
| `PUT`  | `/products/:sku/stock` | Stock update (real-time) |
| `PUT`  | `/products/:sku/price` | Price update |
| `POST` | `/returns` | Create RMA |
| `POST` | `/returns/:id/approve` | Approve return |
| `POST` | `/returns/:id/reject` | Reject return |
| `GET`  | `/webhooks/events` | Available subscribable events |
| `POST` | `/webhooks` | Register endpoint |
| `GET`  | `/health` | Liveness + version |

Rate limits: 100 req/s per partner (burst 200). Pagination: cursor-based, `limit` ≤ 200.

### 5.2 NTS — New Transactional System (Migration)

NTS is Miinto's strategic replacement for the legacy POAS transactional core. It introduces an event-sourced ledger, a unified order/payment/payout model, and ISO-20022-aligned settlement messaging.

**Cutover target**: 2025 (legacy POAS v1 fully decommissioned). As of 2026-05-05 the migration is mid-rollout — partners onboarded after Q1-2026 must be NTS-native; legacy partners are migrated cohort-by-cohort.

#### Migration impact on Partner Portal

| Area | Pre-NTS (POAS v1/v2.5) | Post-NTS |
|---|---|---|
| Order creation | REST `POST /orders` (sync) | Event `OrderProposed` consumed from Kafka topic `nts.orders.v1` |
| Order acceptance | REST `POST /accept` | Command emitted on `nts.commands.v1` |
| Payouts | Batch CSV nightly | Real-time `PayoutSettled` events |
| Idempotency | Header `Idempotency-Key` | Built-in via event ID |
| Retries | Client-driven | Broker-driven, at-least-once with dedupe |
| Auth | OAuth2 + HMAC | mTLS + signed JWT (RS256) |

#### Companion App responsibilities during migration

1. Detect partner cohort flag (`partner.nts_enabled`) at login and route Operations module to the right adapter.
2. Show "Migration Status" widget on dashboard (countdown + readiness checklist).
3. Surface schema differences in Settings → Integrations (webhook payload examples per cohort).
4. Force-trigger sandbox replay tests for partners 30 days before their cutover slot.

#### Partner-facing migration steps

```
T-60 days  → Notification + scheduled walkthrough call
T-30 days  → NTS sandbox credentials issued, replay tests required
T-14 days  → Dual-run period: events arrive in both POAS v2.5 and NTS
T-0        → Cutover; POAS v2.5 read-only for that partner
T+30 days  → Legacy webhooks disabled
```

#### ⚠️ NTS Readiness Gates (Companion App Phase Gates)

> **Strategic recommendation (2026-05-05 re-evaluation):** Decouple Companion App delivery from NTS migration timeline. New partners from Q2 2026 onboard on POAS v2.5 + Companion App. NTS migration continues in background for legacy partners. This prevents NTS delays from blocking Companion App shipping.

Each Companion App release requires the following NTS readiness gates:

| Gate | Requirement | Consequence if Failed |
|------|-------------|----------------------|
| **Phase 1 gate** | NTS dual-run divergence < 0.05% for 14 consecutive days | Delay Companion App Phase 1 until achieved |
| **Phase 2 gate** | NTS dual-run divergence < 0.05% for 30 consecutive days | Delay Phase 2 until achieved |
| **Phase 3 gate** | All legacy partners migrated to NTS | Phase 3 (mobile + scale) cannot ship until all legacy partners are on NTS |
| **Schema freeze** | NTS schema frozen 30 days before each Companion App release | No breaking NTS changes permitted post-freeze |

Circuit breaker: If `portal.nts.event_lag_seconds > 5 min`, automatically route partner to POAS v2.5 adapter (do not wait for on-call).

#### Failure modes specific to NTS migration

- **Dual-run divergence**: events delivered to both stacks must reconcile. Reconciliation job runs hourly; >0.1% divergence pages on-call.
- **Cohort regression**: a partner moved to NTS cannot be moved back; rollback strategy is feature-flag-gated read replay from event log.
- **Schema drift**: NTS payloads carry `schema_version`; consumers reject unknown major versions (fail closed).

---

## 6. Database Schema

```sql
-- ─────────────────────────────────────────────────────────
-- Core entities
-- ─────────────────────────────────────────────────────────

CREATE TABLE partners (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    legal_name VARCHAR(255) NOT NULL,
    trading_name VARCHAR(255),
    vat_number VARCHAR(50) UNIQUE,
    company_type VARCHAR(50),
    address JSONB NOT NULL,
    country CHAR(2) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'pending_onboarding',
    kyc_status VARCHAR(20) NOT NULL DEFAULT 'pending',
    contract_status VARCHAR(20) NOT NULL DEFAULT 'pending',
    nts_enabled BOOLEAN NOT NULL DEFAULT FALSE,
    nts_cutover_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_partners_status ON partners(status);
CREATE INDEX idx_partners_country ON partners(country);

CREATE TABLE partner_users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    partner_id UUID NOT NULL REFERENCES partners(id) ON DELETE CASCADE,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role VARCHAR(50) NOT NULL CHECK (role IN ('admin','manager','viewer')),
    is_2fa_enabled BOOLEAN NOT NULL DEFAULT FALSE,
    totp_secret_encrypted BYTEA,
    webauthn_credentials JSONB DEFAULT '[]',
    last_login_at TIMESTAMPTZ,
    locked_until TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ─────────────────────────────────────────────────────────
-- Onboarding
-- ─────────────────────────────────────────────────────────

CREATE TABLE onboarding_sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    partner_id UUID NOT NULL REFERENCES partners(id) ON DELETE CASCADE,
    current_step VARCHAR(50) NOT NULL,
    step_data JSONB NOT NULL DEFAULT '{}',
    completed_steps TEXT[] NOT NULL DEFAULT '{}',
    started_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_activity_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at TIMESTAMPTZ,
    abandoned_at TIMESTAMPTZ,
    UNIQUE (partner_id)
);
CREATE INDEX idx_onboarding_active ON onboarding_sessions(last_activity_at)
  WHERE completed_at IS NULL AND abandoned_at IS NULL;

-- ─────────────────────────────────────────────────────────
-- KYC
-- ─────────────────────────────────────────────────────────

CREATE TABLE kyc_records (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    partner_id UUID NOT NULL REFERENCES partners(id) ON DELETE CASCADE,
    record_type VARCHAR(30) NOT NULL CHECK (record_type IN ('company','owner','aml','sanctions')),
    status VARCHAR(20) NOT NULL DEFAULT 'pending'
        CHECK (status IN ('pending','approved','rejected','blocked','expired')),
    risk_score VARCHAR(10) CHECK (risk_score IN ('low','medium','high')),
    provider VARCHAR(50),                         -- clearbit, onfido, jumio, manual
    provider_reference VARCHAR(255),
    provider_response JSONB,
    documents JSONB NOT NULL DEFAULT '[]',        -- [{type, s3_key, sha256, uploaded_at}]
    reviewer_user_id UUID,                        -- internal compliance reviewer
    reviewed_at TIMESTAMPTZ,
    rejection_reason TEXT,
    expires_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_kyc_partner ON kyc_records(partner_id);
CREATE INDEX idx_kyc_pending ON kyc_records(status) WHERE status = 'pending';

-- ─────────────────────────────────────────────────────────
-- Contracts
-- ─────────────────────────────────────────────────────────

CREATE TABLE contracts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    partner_id UUID NOT NULL REFERENCES partners(id) ON DELETE CASCADE,
    contract_type VARCHAR(50) NOT NULL,           -- standard, premium, wholesale
    template_id VARCHAR(100) NOT NULL,
    template_version VARCHAR(20) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'draft'
        CHECK (status IN ('draft','sent','viewed','signed','countersigned','expired','voided')),
    docusign_envelope_id VARCHAR(255) UNIQUE,
    rendered_pdf_s3_key TEXT,
    signed_pdf_s3_key TEXT,
    signed_at TIMESTAMPTZ,
    countersigned_at TIMESTAMPTZ,
    expires_at TIMESTAMPTZ,
    audit_trail JSONB NOT NULL DEFAULT '[]',
    metadata JSONB NOT NULL DEFAULT '{}',         -- commission rate, term length, etc.
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_contracts_partner ON contracts(partner_id);
CREATE INDEX idx_contracts_envelope ON contracts(docusign_envelope_id);

-- ─────────────────────────────────────────────────────────
-- API access
-- ─────────────────────────────────────────────────────────

CREATE TABLE api_keys (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    partner_id UUID NOT NULL REFERENCES partners(id) ON DELETE CASCADE,
    key_prefix VARCHAR(16) NOT NULL,              -- mk_live_, mk_test_
    key_hash VARCHAR(255) UNIQUE NOT NULL,        -- argon2id hash
    name VARCHAR(100) NOT NULL,
    scopes TEXT[] NOT NULL DEFAULT '{}',
    environment VARCHAR(10) NOT NULL CHECK (environment IN ('sandbox','live')),
    last_used_at TIMESTAMPTZ,
    last_used_ip INET,
    expires_at TIMESTAMPTZ,
    revoked_at TIMESTAMPTZ,
    revoked_reason TEXT,
    created_by_user_id UUID REFERENCES partner_users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_api_keys_partner ON api_keys(partner_id) WHERE revoked_at IS NULL;

CREATE TABLE webhooks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    partner_id UUID NOT NULL REFERENCES partners(id) ON DELETE CASCADE,
    url TEXT NOT NULL,
    events TEXT[] NOT NULL,
    secret_hash VARCHAR(255) NOT NULL,            -- HMAC shared secret hash
    api_version VARCHAR(10) NOT NULL DEFAULT 'v2.5',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    consecutive_failures INT NOT NULL DEFAULT 0,
    disabled_at TIMESTAMPTZ,
    last_success_at TIMESTAMPTZ,
    last_failure_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE webhook_deliveries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    webhook_id UUID NOT NULL REFERENCES webhooks(id) ON DELETE CASCADE,
    event_type VARCHAR(100) NOT NULL,
    event_id UUID NOT NULL,
    payload JSONB NOT NULL,
    response_status INT,
    response_body_excerpt TEXT,
    attempt INT NOT NULL DEFAULT 1,
    next_retry_at TIMESTAMPTZ,
    delivered_at TIMESTAMPTZ,
    failed_permanently_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_webhook_deliveries_pending
  ON webhook_deliveries(next_retry_at)
  WHERE delivered_at IS NULL AND failed_permanently_at IS NULL;

-- ─────────────────────────────────────────────────────────
-- Training
-- ─────────────────────────────────────────────────────────

CREATE TABLE training_progress (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    partner_id UUID NOT NULL REFERENCES partners(id) ON DELETE CASCADE,
    user_id UUID REFERENCES partner_users(id),
    module_id VARCHAR(50) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'not_started'
        CHECK (status IN ('not_started','in_progress','completed','failed')),
    progress_pct SMALLINT NOT NULL DEFAULT 0 CHECK (progress_pct BETWEEN 0 AND 100),
    quiz_score INT,
    quiz_attempts INT NOT NULL DEFAULT 0,
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    certificate_s3_key TEXT,
    UNIQUE (partner_id, user_id, module_id)
);
CREATE INDEX idx_training_partner ON training_progress(partner_id);
```

---

## 7. Security Architecture

### 7.1 Authentication

- **Password**: argon2id (m=64MB, t=3, p=4). Minimum 12 chars; HIBP breach check on signup + reset.
- **2FA**: TOTP (RFC 6238) **mandatory for admin** role; optional WebAuthn passkeys; SMS only as backup.
- **Session**: short-lived JWT access tokens + rotating refresh tokens.
- **Mobile**: refresh token bound to device via attestation (DeviceCheck / Play Integrity).

```
Access token (JWT)
  alg:    RS256
  iss:    https://auth.miinto.com
  aud:    [companion-app, poas-api]
  exp:    15 minutes
  claims: sub, partner_id, scopes[], 2fa_verified, role, env

Refresh token
  format: opaque, server-side store
  ttl:    7 days, rotated on each use
  bound:  device_id (mobile) or http-only cookie (web)
```

### 7.2 Authorization

| Layer | Control |
|---|---|
| Module loading | `requiredScopes` checked against token before module init |
| API gateway | Scope check + partner_id scoping (cannot access another tenant) |
| Row-level | `partner_id` filter enforced via Postgres RLS in shared tables |
| Sensitive ops | Step-up auth — re-prompt 2FA for: API key creation, contract sign, payout method change |

### 7.3 Data protection

- **At rest**: AES-256 (RDS encryption) + field-level encryption for `totp_secret`, KYC document blobs (KMS envelope).
- **In transit**: TLS 1.3 only; HSTS preload; mTLS between BFF and Miinto core (NTS).
- **PII minimization**: only retain KYC documents 7 years post-contract end (legal); auto-purge on schedule.
- **GDPR**: subject access export + erasure flow, audit-logged.

### 7.4 HMAC signing for Miinto API calls

Every outbound request from the Companion App backend to POAS v2.5 / NTS carries an HMAC signature that the Miinto core validates.

```
Headers:
  Authorization:    Bearer <oauth-access-token>
  X-Miinto-Key-Id:  <key-id-rotated-90d>
  X-Miinto-Ts:      <unix-seconds>
  X-Miinto-Nonce:   <uuid-v4>
  X-Miinto-Sig:     base64(HMAC-SHA256(secret,
                          method + "\n" +
                          path + "\n" +
                          ts + "\n" +
                          nonce + "\n" +
                          sha256(body)))

Validation:
  • Reject if |now - ts| > 300s            (replay window)
  • Reject if nonce seen in last 10 min     (Redis dedupe)
  • Constant-time signature compare
```

Inbound webhooks **from** Miinto to the partner use the same scheme with the partner's `secret_hash` from the `webhooks` table.

### 7.5 Other controls

- Rate limiting: 100 req/s per partner per endpoint (token bucket on Redis).
- IP allowlisting: optional per API key; mandatory for keys with `payout:write` scope.
- Audit log: every mutation on `partners`, `contracts`, `api_keys`, `kyc_records`, `webhooks` writes immutable row to `audit_events` (append-only, separate DB user).
- WAF: AWS WAF managed rule sets + custom rules for known abuse patterns.

---

## 8. External Integrations

| System | Purpose | Notes |
|---|---|---|
| DocuSign | Contract signing (embedded + email) | Webhook → Contract Service updates `contracts.status` |
| Clearbit | Company KYC enrichment | Risk score input, not authoritative |
| VIES | EU VAT validation | Free; cached 24h |
| Onfido / Jumio | Owner ID + biometric verification | Used when ownership concentration > 25% |
| SendGrid | Transactional email | Templates versioned in repo |
| Intercom | Live chat + KB | Embedded in Support module via secure messenger |
| Stripe | Billing + payouts (legacy POAS); replaced by NTS settlement post-cutover | |
| Slack | Internal ops channels (`#new-partner-applications`, `#kyc-review-queue`, `#contract-signed`, `#alerts`) | |
| LaunchDarkly | Feature flags for module rollout + NTS cohorts | |
| Datadog | APM, logs, RUM, synthetic monitoring | |

---

## 9. Deployment

### 9.1 Environments

```
┌────────────────┐     ┌────────────────┐     ┌────────────────┐
│   Development  │     │     Staging    │     │   Production   │
│ Local Docker   │ ──▶ │ Auto-deploy    │ ──▶ │ Blue/Green     │
│ Mock services  │     │ Preview URLs   │     │ Multi-AZ       │
│ Feature flags  │     │ Full stack     │     │ Multi-region   │
└────────────────┘     └────────────────┘     └────────────────┘
```

### 9.2 Production infrastructure (AWS)

```
                    CloudFront (CDN, WAF)
                            │
                    Route 53 + ALB (multi-AZ)
                            │
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                   ▼
   ECS Fargate         ECS Fargate         ECS Fargate
   (BFF gateway)       (domain svcs)       (workers)
        │                   │                   │
        └───────────────────┼───────────────────┘
                            ▼
              ElastiCache Redis (Multi-AZ)
                            │
              RDS PostgreSQL (Multi-AZ + read replicas)
                            │
              S3 (SSE-KMS, versioned, lifecycle to Glacier)
                            │
              MSK (Kafka) — for NTS event consumption
```

---

## 10. SLA Metrics

| Stage | SLA Target | Max Allowed | Owner |
|---|---|---|---|
| Application Validation | 1 hour | 4 hours | Onboarding |
| KYC Automated Check | 5 min | 30 min | KYC |
| KYC Manual Review | 24 hours | 48 hours | Compliance Ops |
| Contract Generation | Instant | 1 min | Contract |
| DocuSign Signing | Partner-dependent | 14 days | Partner |
| API Credentials Issuance | Instant | 5 min | Partner |
| Go-Live Approval | 24 hours | 48 hours | Partner Success |
| Support First Response (P1) | 1 hour | 2 hours | Support |
| Support First Response (P2/P3) | 4 hours | 8 hours | Support |
| Webhook Delivery | < 5s p95 | 30s p99 | Platform |
| POAS v2.5 API availability | 99.9% | — | Platform |
| NTS event processing lag | < 30s p95 | < 5 min p99 | Platform |
| Migration cutover window | ≤ 2 hours | 4 hours | Platform |

---

## 11. Failure Handling & Retry Logic

### 11.1 Retry strategy (exponential backoff with jitter)

```
Outbound HTTP call (DocuSign, Clearbit, POAS, NTS publish)
  │
  ├─ Attempt 1 — immediate
  ├─ Attempt 2 — 1 min  ± 30%
  ├─ Attempt 3 — 5 min  ± 30%
  ├─ Attempt 4 — 15 min ± 30%
  └─ Attempt 5 — 1 hour ± 30%
       │
       └─ Still failing → dead-letter queue → ops alert (Slack + PagerDuty)
              │
              └─ Manual intervention queue with replay tooling
```

Webhooks **to** partners follow the same schedule, capped at 24h total. After max attempts:
- Mark `webhook_deliveries.failed_permanently_at`.
- Increment `webhooks.consecutive_failures`.
- At 100 consecutive failures within 24h, auto-disable the webhook and notify the partner.

### 11.2 Idempotency

| Path | Mechanism |
|---|---|
| Inbound POAS v2.5 calls | `Idempotency-Key` header, 24h Redis dedupe |
| NTS event consumers | event ID dedupe in `processed_events` table |
| Outbound calls | client-side idempotency keys per business operation |
| DocuSign envelope creation | one envelope per `contract.id`; collision returns existing |

### 11.3 Rollback scenarios

| Scenario | Recovery |
|---|---|
| Contract signing fails mid-envelope | Void DocuSign envelope, regenerate with same `contract.id`, append to audit trail |
| KYC verification times out | Extend timeout to 72h, notify partner with "we're still reviewing" email; escalate to compliance after 48h |
| API credential generation fails | Transactional rollback (no orphan key rows); user retries with idempotency key |
| Go-live blocked by sandbox checklist | Dashboard surfaces failed checks; ops can override with reason logged |
| NTS cutover divergence | Pause cutover, replay events from POAS v2.5 ledger, reconcile, resume |
| Webhook secret leaked | One-click rotate in Settings → invalidates old secret immediately, 5-min grace dual-accept window |

### 11.4 Error notification channels

- **Partner-visible**: in-app toast, banner on affected module, email for >P3 severity.
- **Internal**: Slack `#alerts` (warn), `#alerts-critical` (page), PagerDuty for P1/P2.
- **Audit**: every failure writes to `audit_events` with correlation ID linking partner action → service hop → external call.

---

## 12. Monitoring & Observability

### Metrics (Datadog)

- Module-level: load time, error rate, time-to-interactive, drop-off per onboarding step.
- Service-level: request latency p50/p95/p99, error rate, queue depth, DB connection pool saturation.
- Business: active partners, onboarding funnel conversion, contract signing time, training completion rate, NTS migration progress per cohort.

### Logging (CloudWatch + Datadog)

- Structured JSON, correlation IDs propagated end-to-end (web → BFF → service → external).
- Levels: ERROR, WARN, INFO, DEBUG. PII scrubbed at log-shipper layer.
- Retention: 30 days hot, 1 year cold (S3).

### Alerting

- PagerDuty integration with severity-based routing.
- SLA-based alerts (latency, error rate, queue lag).
- NTS-specific: dual-run divergence > 0.1%, event lag > 5 min, schema-version-rejection rate > 0.
- Synthetic monitoring on the critical onboarding path every 5 minutes from 4 regions.

---

## 13. API Contracts (Companion App ↔ BFF)

These are the partner-facing REST endpoints exposed by the BFF to the Companion App and to direct API consumers. All requests require `Authorization: Bearer <JWT>` unless explicitly noted; all bodies and responses are `application/json`.

**Base URL**: `https://api.miinto.com/portal/v1`

### 13.1 POST /auth/login

Public endpoint. Issues a JWT access token + refresh token.

Request:
```json
{
  "email": "owner@boutique-x.com",
  "password": "•••••••••••",
  "totp_code": "748291",
  "device_id": "dvc_01HV3K7M5N9Q...",
  "remember_device": true
}
```

Response `200`:
```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIs...",
  "refresh_token": "rt_01HV3K8M9Q...",
  "token_type": "Bearer",
  "expires_in": 900,
  "user": {
    "id": "usr_01HV3K8M...",
    "partner_id": "ptr_01HV2J1Z...",
    "role": "admin",
    "scopes": ["onboarding:read","onboarding:write","contract:read","kyc:read"],
    "twofa_verified": true
  }
}
```

Errors:
- `401 invalid_credentials` — email/password mismatch.
- `412 totp_required` — 2FA enabled but no `totp_code` supplied.
- `423 account_locked` — too many failed attempts; `locked_until` returned.
- `429 rate_limited` — IP-level abuse guard (10/min per IP).

### 13.2 POST /onboarding/start

Begins or resumes the onboarding state machine. Idempotent on `partner_id`.

Headers: `Idempotency-Key: <uuid>` (recommended).

Request:
```json
{
  "partner_id": "ptr_01HV2J1Z...",
  "channel": "self_service",
  "referral_code": "MIINTO-NL-2026"
}
```
`channel` ∈ `self_service | sales_assisted | partner_referral`.

Response `201`:
```json
{
  "session_id": "ons_01HV3K8M...",
  "current_step": "company_info",
  "completed_steps": [],
  "next_action": {
    "url": "/onboarding/ons_01HV3K8M.../company-info",
    "fields_required": ["legal_name","vat_number","address","country"]
  },
  "expires_at": "2026-05-19T10:00:00Z"
}
```

Response `200` if a session already exists — the existing session is returned untouched.

Errors:
- `409 onboarding_already_complete` — partner has `status='active'`.
- `403 partner_blocked` — KYC blocklist hit.

### 13.3 GET /kyc/status

Returns the current KYC posture for the authenticated partner.

Response `200`:
```json
{
  "partner_id": "ptr_01HV2J1Z...",
  "overall_status": "in_review",
  "risk_score": "medium",
  "records": [
    {
      "type": "company",
      "status": "approved",
      "provider": "clearbit",
      "approved_at": "2026-05-02T08:14:00Z",
      "expires_at": "2027-05-02T08:14:00Z"
    },
    {
      "type": "owner",
      "status": "in_review",
      "provider": "onfido",
      "submitted_at": "2026-05-04T15:22:00Z",
      "estimated_completion": "2026-05-05T15:22:00Z",
      "documents_required": []
    },
    {
      "type": "sanctions",
      "status": "approved",
      "approved_at": "2026-05-02T08:15:00Z"
    }
  ],
  "blocking_reasons": []
}
```
`overall_status` ∈ `pending | in_review | approved | rejected | blocked | expired`.

### 13.4 POST /contracts/generate

Generates a contract from a template and creates a DocuSign envelope.

Headers: `Idempotency-Key: <uuid>` **required**.

Request:
```json
{
  "partner_id": "ptr_01HV2J1Z...",
  "template_id": "standard_eu_2026_q1",
  "metadata": {
    "commission_rate": 0.18,
    "term_months": 24,
    "category": "luxury_apparel",
    "exclusivity": false
  }
}
```

Response `201`:
```json
{
  "contract_id": "ctr_01HV3K8M...",
  "status": "sent",
  "docusign_envelope_id": "env_a8f7c2e9-...",
  "signing_url": "https://docusign.com/signing/...",
  "expires_at": "2026-05-19T10:00:00Z",
  "rendered_pdf_url": "https://docs.miinto.com/contracts/ctr_01HV3K8M.../draft.pdf?sig=..."
}
```

Errors:
- `409 contract_already_active` — contract already exists in `signed`/`countersigned` state.
- `412 kyc_not_approved` — KYC must be `approved` first.
- `422 template_unavailable` — template retired.

### 13.5 POST /contracts/{id}/sign

Server-side mark-as-signed callback used by the DocuSign embedded-signing return handler. The cryptographic signature is captured by DocuSign; this endpoint records the event in our domain.

Request:
```json
{
  "docusign_event": "envelope_completed",
  "signed_at": "2026-05-05T12:34:56Z",
  "signer_ip": "203.0.113.42",
  "signer_user_agent": "Mozilla/5.0 ..."
}
```

Response `200`:
```json
{
  "contract_id": "ctr_01HV3K8M...",
  "status": "signed",
  "signed_pdf_url": "https://docs.miinto.com/contracts/ctr_01HV3K8M.../signed.pdf?sig=...",
  "next_step": "tech_setup",
  "audit_trail_entry_id": "aud_01HV3K8M..."
}
```

Errors:
- `409 already_signed` — idempotent replay returns the existing record.
- `410 envelope_voided` — contract was voided before signature.

### 13.6 GET /training/modules

Lists training modules available to the calling user with progress overlay.

Query: `?role=admin&category=onboarding&include_optional=true`

Response `200`:
```json
{
  "modules": [
    {
      "id": "intro_to_miinto",
      "title": "Introduction to Miinto",
      "category": "onboarding",
      "duration_min": 12,
      "required_for_role": ["admin","manager"],
      "status": "completed",
      "progress_pct": 100,
      "quiz_score": 92,
      "completed_at": "2026-04-29T09:01:00Z",
      "certificate_url": "https://docs.miinto.com/certs/...pdf"
    },
    {
      "id": "order_lifecycle",
      "title": "Order Lifecycle & SLA",
      "category": "operations",
      "duration_min": 25,
      "required_for_role": ["admin","manager"],
      "status": "in_progress",
      "progress_pct": 40,
      "quiz_score": null
    }
  ],
  "completion_required_pct": 80,
  "completion_pct": 56
}
```

### 13.7 POST /training/progress

Records a progress event from the player. Designed to be called frequently (heartbeat-style); coalesced server-side.

Request — video heartbeat:
```json
{
  "module_id": "order_lifecycle",
  "event": "video_progress",
  "progress_pct": 65,
  "playhead_seconds": 487,
  "session_id": "trn_01HV3K8M..."
}
```
`event` ∈ `video_started | video_progress | video_completed | quiz_submitted`.

Request — quiz submission:
```json
{
  "module_id": "order_lifecycle",
  "event": "quiz_submitted",
  "answers": [
    {"question_id": "q1", "selected": "b"},
    {"question_id": "q2", "selected": ["a","c"]}
  ],
  "session_id": "trn_01HV3K8M..."
}
```

Response `200` — heartbeat:
```json
{
  "module_id": "order_lifecycle",
  "status": "in_progress",
  "progress_pct": 65,
  "next_quiz_unlocked_at_pct": 100
}
```

Response `200` — quiz submission:
```json
{
  "module_id": "order_lifecycle",
  "status": "completed",
  "quiz_score": 88,
  "passed": true,
  "certificate_url": "https://docs.miinto.com/certs/...pdf",
  "next_module": "returns_management"
}
```

---

## 14. Webhook Specification

Webhooks fire from Miinto core (POAS v2.5 / NTS) to partner-registered endpoints whenever a domain event occurs. All payloads share a common envelope and HMAC signature scheme.

### 14.1 Common envelope

Headers on every webhook delivery:

| Header | Purpose |
|---|---|
| `Content-Type: application/json` | |
| `X-Miinto-Event` | Event name, e.g. `order.created`. |
| `X-Miinto-Event-Id` | UUID; **stable across retries** — use to dedupe. |
| `X-Miinto-Delivery-Id` | UUID; new per delivery attempt. |
| `X-Miinto-Ts` | Unix seconds at signing. |
| `X-Miinto-Signature` | `t=<ts>,v1=<hex(HMAC-SHA256(secret, ts + "." + body))>`. |
| `X-Miinto-Api-Version` | `v2.5` or `nts.v1`. |
| `X-Miinto-Attempt` | 1..5 (retry count). |

All payloads share the same outer shape:

```json
{
  "id": "evt_01HV3K8M...",
  "type": "order.created",
  "api_version": "v2.5",
  "created_at": "2026-05-05T12:34:56.789Z",
  "partner_id": "ptr_01HV2J1Z...",
  "data": { /* event-specific, see below */ }
}
```

### 14.2 Order events

`order.created` — Miinto received an order destined for the partner.

```json
{
  "id": "evt_01HV3K8M...",
  "type": "order.created",
  "api_version": "v2.5",
  "created_at": "2026-05-05T12:34:56Z",
  "partner_id": "ptr_01HV2J1Z...",
  "data": {
    "order_id": "ord_01HV3K8M...",
    "external_reference": "MIINTO-2026-882041",
    "currency": "EUR",
    "total_amount": 489.00,
    "commission_amount": 88.02,
    "payout_amount": 400.98,
    "customer": {
      "country": "NL",
      "language": "nl",
      "shipping_address_id": "addr_01HV3K8M..."
    },
    "items": [
      {
        "line_id": "oli_1",
        "sku": "BX-LV-1234",
        "qty": 1,
        "unit_price": 489.00,
        "title": "Leather handbag — black"
      }
    ],
    "deadline_to_accept": "2026-05-05T14:34:56Z",
    "deadline_to_ship": "2026-05-07T18:00:00Z"
  }
}
```

`order.accepted` — partner confirmed acceptance (echo for audit).

```json
{
  "type": "order.accepted",
  "data": {
    "order_id": "ord_01HV3K8M...",
    "accepted_at": "2026-05-05T12:40:11Z",
    "accepted_by_user_id": "usr_01HV3K8M...",
    "expected_ship_date": "2026-05-06"
  }
}
```

`order.shipped`:

```json
{
  "type": "order.shipped",
  "data": {
    "order_id": "ord_01HV3K8M...",
    "shipped_at": "2026-05-06T09:15:00Z",
    "carrier": "DHL_EXPRESS",
    "tracking_number": "JD0099887766",
    "tracking_url": "https://dhl.com/track?...",
    "label_s3_key": "labels/2026/05/ord_01HV3K8M....pdf"
  }
}
```

### 14.3 Return events

`return.created` — customer raised an RMA.

```json
{
  "type": "return.created",
  "data": {
    "return_id": "rma_01HV3K8M...",
    "order_id": "ord_01HV3K8M...",
    "items": [
      { "line_id": "oli_1", "qty": 1, "reason_code": "size_mismatch", "customer_note": "Smaller than expected" }
    ],
    "deadline_to_decide": "2026-05-12T23:59:59Z",
    "label_required": true
  }
}
```

`return.approved`:

```json
{
  "type": "return.approved",
  "data": {
    "return_id": "rma_01HV3K8M...",
    "approved_at": "2026-05-08T10:02:11Z",
    "refund_amount": 489.00,
    "restocking_fee": 0,
    "carrier_label_url": "https://docs.miinto.com/labels/rma_01HV3K8M....pdf"
  }
}
```

### 14.4 HMAC verification

Constant-time comparison; reject if `|now − ts| > 5 min` or signature mismatch. The signed string is `<ts>.<raw_body>` — note the literal dot.

Node.js (Express):
```javascript
const crypto = require('node:crypto');

function verifyMiintoSignature(req, secret) {
  const header = req.headers['x-miinto-signature']; // "t=1714905296,v1=ab12..."
  const ts     = req.headers['x-miinto-ts'];
  const raw    = req.rawBody;                       // Buffer; capture in body parser

  if (!header || !ts) return false;
  if (Math.abs(Math.floor(Date.now() / 1000) - Number(ts)) > 300) return false;

  const v1 = Object.fromEntries(
    header.split(',').map(p => p.trim().split('='))
  ).v1;

  const expected = crypto
    .createHmac('sha256', secret)
    .update(`${ts}.${raw.toString('utf8')}`)
    .digest('hex');

  const a = Buffer.from(v1, 'hex');
  const b = Buffer.from(expected, 'hex');
  return a.length === b.length && crypto.timingSafeEqual(a, b);
}
```

Python (Flask):
```python
import hmac, hashlib, time

def verify_miinto_signature(headers, raw_body: bytes, secret: bytes) -> bool:
    sig_header = headers.get("X-Miinto-Signature", "")
    ts = headers.get("X-Miinto-Ts", "")
    if not sig_header or not ts:
        return False
    if abs(int(time.time()) - int(ts)) > 300:
        return False
    parts = dict(p.strip().split("=", 1) for p in sig_header.split(","))
    v1 = parts.get("v1", "")
    msg = f"{ts}.".encode() + raw_body
    expected = hmac.new(secret, msg, hashlib.sha256).hexdigest()
    return hmac.compare_digest(v1, expected)
```

### 14.5 Delivery semantics

- **At-least-once.** Consumers must dedupe on `X-Miinto-Event-Id`.
- **Ordering**: not guaranteed across event types. Within a single `order_id`, events are ordered by `created_at`.
- **Retries**: 5 attempts, exponential backoff capped at 24 h (see §11.1).
- **Auto-disable**: 100 consecutive failures in 24 h → endpoint disabled, partner emailed.
- **Replay**: ops can replay any event by `event_id` for up to 30 days from the Operations console.

---

## 15. Idempotency Keys

The portal uses idempotency keys to guarantee that retries — whether triggered by the network, the user, or our own retry logic — do not produce duplicate side effects.

### 15.1 Where required

| Method + Path | Required? | Rationale |
|---|---|---|
| `POST /auth/login` | No | Re-issuing a token is safe. |
| `POST /onboarding/start` | Recommended | Naturally idempotent on `partner_id`; key avoids race. |
| `POST /contracts/generate` | **Required** | Creates a billable DocuSign envelope. |
| `POST /contracts/{id}/sign` | **Required** | Records a signature; replay must be a no-op. |
| `POST /training/progress` | No | Coalesced server-side. |
| `POST /orders/{id}/accept` (POAS v2.5) | **Required** | State-changing, paired with deadline timer. |
| `POST /orders/{id}/ship` (POAS v2.5) | **Required** | Triggers customer notification. |
| `POST /returns/{id}/approve` (POAS v2.5) | **Required** | Triggers refund. |
| `PUT /products/{sku}/stock` | No | Last-write-wins by design. |
| `POST /webhooks` | Recommended | Avoids duplicate registrations. |
| `POST /api-keys` | **Required** | Issues a credential. |

### 15.2 Key format

- Header: `Idempotency-Key`.
- Format: client-generated UUID v4 or v7, 16–64 chars `[A-Za-z0-9_-]+`.
- Scope: per `(partner_id, route, key)` tuple.
- TTL: 24 hours in Redis; after TTL the key may be reused.
- Body fingerprint: server stores `sha256(body)` alongside the key. **A second request with the same key but a different body fingerprint returns `409 idempotency_conflict`** — protects against the client mutating the payload mid-retry.

### 15.3 Server semantics

```
Receive request with Idempotency-Key K
  ├─ Redis GET idem:{partner_id}:{route}:{K}
  │
  ├─ Miss:
  │    ├─ SET idem:... = "in_progress" with NX (5 min TTL)
  │    ├─ Process request
  │    ├─ SET idem:... = {status_code, body, body_fingerprint} (24 h TTL)
  │    └─ Return response
  │
  ├─ Hit: in_progress → return 425 Too Early (client retries after delay)
  │
  └─ Hit: completed
       ├─ Body fingerprint matches → return cached response (status, body)
       └─ Body fingerprint differs → return 409 idempotency_conflict
```

### 15.4 Client guidance

- Generate the key **before** the first send; reuse on retries triggered by network failure or 5xx.
- Do **not** reuse keys across distinct business operations.
- Store the key alongside the local representation of the operation until you receive a definitive 2xx/4xx (not 5xx).

---

## 16. Rate Limiting

Rate limits protect the platform and provide predictable QoS per partner tier. Limits apply at the BFF edge using a token-bucket implementation in Redis (`limiter:{partner_id}:{tier_key}`).

### 16.1 Tiers

`partners.tier` is set at contract signing based on the commercial agreement.

| Tier | Default bucket | Burst | Concurrent | Webhook out (per partner) |
|---|---|---|---|---|
| `starter` | 30 req/s | 60 | 10 | 5 in-flight |
| `pro` | 100 req/s | 200 | 50 | 25 in-flight |
| `enterprise` | 500 req/s | 1000 | 200 | 100 in-flight |
| `internal` | unlimited | — | 1000 | n/a |

### 16.2 Per-endpoint overrides

Some endpoints are more expensive and have stricter limits independent of tier. The most restrictive applicable bucket wins.

| Endpoint | Starter | Pro | Enterprise |
|---|---|---|---|
| `POST /contracts/generate` | 5/min | 30/min | 120/min |
| `POST /auth/login` | 10/min/IP | 10/min/IP | 10/min/IP |
| `PUT /products/{sku}/stock` | 30/s | 200/s | 1000/s |
| `GET /orders` (list) | 10/s | 30/s | 100/s |
| Reports / analytics | 5/min | 20/min | 60/min |

### 16.3 Headers

Every API response carries:

```
X-RateLimit-Limit:     100        # bucket capacity
X-RateLimit-Remaining: 87
X-RateLimit-Reset:     1714905600 # unix seconds at refill
X-RateLimit-Tier:      pro
```

On exceedance the server returns:

```
HTTP/1.1 429 Too Many Requests
Retry-After: 3
{
  "error": "rate_limited",
  "tier": "pro",
  "limit": 100,
  "retry_after_ms": 3000,
  "request_id": "req_01HV3K8M..."
}
```

### 16.4 Burst & smoothing

Token buckets refill continuously; burst capacity equals `2 × bucket_size`. Repeated bursts that empty the bucket emit a `partner.tier_pressure` event to the AM team for upgrade-fit review.

### 16.5 Tier upgrades

- Upgrades take effect immediately (config flag) and are prorated for the billing period.
- Downgrades take effect at the next billing cycle to avoid mid-period traffic shocks.

---

## 17. Monitoring, Observability & Dashboards

§12 listed metric categories; this section enumerates the actual Datadog assets and SLOs the platform team operates against.

### 17.1 Datadog dashboards

| Dashboard | Audience | Key panels |
|---|---|---|
| `Partner Portal — Golden Signals` | On-call | RPS, latency p50/p95/p99, error rate, saturation per service |
| `Onboarding Funnel` | Product | Step entries, completions, drop-offs, time-in-step, abandonment by country |
| `KYC Pipeline` | Compliance | Records by status, manual-review queue depth, provider error rate, time-to-decision |
| `Contract Lifecycle` | Partner Success | Envelopes sent/viewed/signed, time-to-sign cohorts, DocuSign 5xx rate |
| `Webhook Reliability` | Platform | Delivery success %, retry depth, p95 delivery time, auto-disabled endpoints |
| `NTS Migration` | Platform + eng leadership | Cohort cutover progress, dual-run divergence %, event lag, schema-rejection rate |
| `Partner API Usage` | Account managers | Per-partner request volume, tier pressure, rate-limit hits |
| `Companion App RUM` | Mobile / Web team | TTI, crash-free sessions, deep-link entry, module load time |

### 17.2 Key metrics & SLOs

| Metric | Tag dimensions | SLO | Burn-rate alert |
|---|---|---|---|
| `portal.bff.request.latency_ms` p95 | `route`, `partner_tier` | < 300 ms | 2× burn over 1 h |
| `portal.bff.error_rate` | `route`, `status_class` | < 0.5% | 5× burn over 5 m |
| `portal.onboarding.step_drop_off` | `step`, `country` | < 15% per step | weekly trend alert |
| `portal.kyc.time_to_decision` p95 | `record_type`, `provider` | < 24 h auto / < 48 h manual | breach → ops page |
| `portal.contract.docusign_failure_rate` | `template_id` | < 1% | 1 h burn |
| `portal.webhook.delivery_p95_ms` | `event_type`, `partner_tier` | < 5 s | sustained 30 m |
| `portal.webhook.delivery_success_rate` | `event_type` | > 99.5% (1 h rolling) | 99.0% hard floor |
| `portal.nts.event_lag_seconds` | `topic`, `cohort` | p95 < 30 s | p99 > 5 m |
| `portal.nts.dualrun_divergence_pct` | `cohort` | < 0.1% | > 0.1% pages |
| `portal.api.rate_limit_429_pct` | `partner_id`, `tier` | < 1% per partner | trend → AM review |
| `portal.companion.crash_free_sessions` | `platform`, `app_version` | > 99.5% | release-gating |

### 17.3 Logs, traces, RUM

- **Logs**: structured JSON to CloudWatch → forwarded to Datadog. Mandatory fields: `request_id`, `partner_id`, `user_id`, `correlation_id`, `module`, `span_id`. PII is stripped at the shipper layer (`password*`, `*_token`, email partially masked outside `auth.*` services).
- **Traces**: OpenTelemetry → Datadog APM. Trace propagation across BFF → service → POAS/NTS via `traceparent`. Sampling: 100% errors, 10% baseline, 100% during incidents.
- **RUM**: Datadog Browser & Mobile SDKs in the Companion App. Session replay with PII masking for `[data-private]` selectors.

### 17.4 Alert routing

| Severity | Condition example | Channel |
|---|---|---|
| P1 | error rate > 5% for 5 min, or NTS dual-run divergence > 1% | PagerDuty primary + Slack `#alerts-critical` |
| P2 | p95 latency > 1 s for 10 min on a critical path | PagerDuty secondary + `#alerts` |
| P3 | webhook auto-disable for a partner | `#alerts` + AM Slack DM |
| P4 | tier pressure detected | `#partner-success` weekly digest |

### 17.5 Synthetic tests

Datadog Synthetics every 5 minutes from `eu-west-1`, `eu-central-1`, `eu-north-1`, `us-east-1`:

1. Marketing → apply form submit (HTTP).
2. Companion App login → onboarding wizard step 1 load (browser test).
3. POAS v2.5 `GET /health` (multistep).
4. NTS Kafka consumer lag probe (custom check).

---

## 18. Disaster Recovery

### 18.1 RTO / RPO targets

| Service tier | RTO | RPO |
|---|---|---|
| Tier 1 — Auth, BFF, Companion App, POAS v2.5 read | 1 hour | 5 minutes |
| Tier 2 — Onboarding, Contract, KYC, Training | 4 hours | 15 minutes |
| Tier 3 — Reporting, analytics, internal back-office | 24 hours | 1 hour |

### 18.2 Backup strategy

| Asset | Mechanism | Frequency | Retention |
|---|---|---|---|
| RDS PostgreSQL | Automated snapshots + PITR | Snapshot daily, WAL continuous | 35 days snapshots, 7-day PITR |
| RDS PostgreSQL — long-term | Cross-region monthly snapshot copy to `eu-north-1` | Monthly | 7 years (compliance) |
| ElastiCache Redis | Daily RDB snapshot to S3; cache is rebuildable | Daily | 7 days |
| S3 (documents, contracts, KYC) | Versioning + cross-region replication to `eu-north-1`; Object Lock on signed contracts | Continuous | Versions 90 days; signed contracts 10 years (Object Lock) |
| MSK Kafka topics (NTS) | Tiered storage; cross-region mirror via MirrorMaker 2 | Continuous | 30 days hot, 1 year cold |
| Application config | IaC in Git (Terraform); secrets in AWS Secrets Manager (replicated) | On change | Indefinite |

Backups are **tested monthly**: a restore drill in the DR account must produce a working PostgreSQL replica and S3 mirror that can serve a smoke-test session.

### 18.3 Failover procedures

**Primary region**: `eu-west-1`. **DR region**: `eu-north-1` (warm standby).

| Scenario | Detection | Action | Target |
|---|---|---|---|
| Single AZ failure | ALB health checks | Automatic — ECS reschedules across remaining AZs; RDS Multi-AZ failover | < 2 min |
| RDS primary failure | RDS event + Datadog | Automatic Multi-AZ failover | < 60 s |
| Region-wide failure | CloudWatch composite alarm + PagerDuty | Manual: invoke `dr-promote` runbook → Route 53 weighted record flips → promote read replica in `eu-north-1` → start ECS services from latest task definitions | RTO 1 h |
| Data corruption | Audit log + reconciliation jobs | PITR restore to a new instance, app traffic paused, replay from corruption checkpoint | RTO 4 h |
| Compromised credential | Auth service + WAF | Forced session invalidation, key rotation, partner notification | < 30 min |
| MSK outage | Lag alert | Switch consumers to mirrored topic in DR; producers pause; replay backlog on resume | RTO 1 h |

Failover is **practiced quarterly** in a game-day exercise. The runbook lives at `runbooks/dr/region-failover.md`.

### 18.4 Communications during incidents

- Status page (`status.miinto-partners.com`) updated within 15 min of P1.
- Affected partners receive in-app banner + email; tier-1/enterprise partners are paged by their AM.
- Post-incident review within 5 business days; report shared with affected enterprise partners.

---

## 19. Compliance

### 19.1 GDPR — data handling

| Concern | Approach |
|---|---|
| Lawful basis | Contractual necessity (operating partnership) and legitimate interest (fraud, KYC, security). Marketing communications via separate consent record. |
| Data residency | Partner PII stored in EU regions only (`eu-west-1` primary, `eu-north-1` DR). Transfers outside EEA covered by DPA + SCCs (DocuSign, SendGrid — listed in subprocessor register). |
| Subject access (Art. 15) | Self-service export from Settings → Privacy. Generates a ZIP of profile, contracts, KYC records, training history and audit-log excerpts within 7 days. |
| Erasure (Art. 17) | Self-service erasure request → 30-day cooling-off (legal hold check) → cascading anonymization. Records subject to legal retention (signed contracts, AML logs) are anonymized at the user level but retained for the legal period. |
| Rectification (Art. 16) | All profile fields editable in Settings; KYC corrections trigger re-verification. |
| Portability (Art. 20) | The Art. 15 export is in machine-readable JSON. |
| DPO contact | `dpo@miinto.com` — surfaced in app footer and privacy policy. |
| Breach notification | < 72 h to authority; partner notification per Art. 34 if high risk. Runbook: `runbooks/security/breach-response.md`. |

### 19.2 Data retention

| Data class | Retention | Trigger | Mechanism |
|---|---|---|---|
| Active partner profile | Lifetime of contract + 7 years | Contract end | Anonymization job |
| Signed contracts | 10 years from contract end | Contract end + 10 y | S3 Object Lock + scheduled deletion |
| KYC documents | 7 years from contract end | Contract end + 7 y | KMS-encrypted S3 with lifecycle policy |
| AML / sanctions screening records | 7 years from screening | Screening date + 7 y | Append-only `audit_events` partition |
| Onboarding session drafts (abandoned) | 90 days | `abandoned_at + 90 d` | Daily purge job |
| Authentication logs | 1 year hot, 7 years cold | Event timestamp | CloudWatch → S3 Glacier |
| Webhook delivery logs | 30 days hot, 90 days cold | Event timestamp | Postgres partition + S3 archive |
| Marketing analytics (anonymized) | 25 months | Event timestamp | Mixpanel retention setting |
| Support tickets | 5 years | Ticket close | Intercom retention setting |
| Training progress | Lifetime of contract + 3 years | Contract end | Anonymization (replace `user_id` with hash) |

A nightly retention job (`workers/retention/sweep.ts`) walks each table, applies the policy, writes a summary to `audit_events`, and pages on policy violations (records past TTL not yet purged).

### 19.3 Audit-log requirements

`audit_events` is the immutable record of every privileged or compliance-relevant action.

```sql
CREATE TABLE audit_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    actor_type VARCHAR(20) NOT NULL,        -- partner_user, internal_user, system, partner_api
    actor_id UUID,
    actor_ip INET,
    actor_user_agent TEXT,
    partner_id UUID,
    event_category VARCHAR(50) NOT NULL,    -- auth, kyc, contract, api_key, webhook, payout, settings
    event_action VARCHAR(100) NOT NULL,     -- e.g. contract.signed, api_key.created
    target_type VARCHAR(50),                -- contract, api_key, ...
    target_id UUID,
    before_state JSONB,
    after_state JSONB,
    correlation_id UUID,
    request_id UUID,
    severity VARCHAR(10) NOT NULL DEFAULT 'info', -- info, warn, security
    prev_hash BYTEA,                        -- hash-chain integrity link
    metadata JSONB NOT NULL DEFAULT '{}'
) PARTITION BY RANGE (occurred_at);
```

Constraints:

- **Append-only**: write-only DB user; no `UPDATE`/`DELETE` grant. Logical tampering is detected by an hourly hash-chain integrity check (`prev_hash` column) — any break pages compliance.
- **Coverage**: required for every mutation on `partners`, `partner_users`, `contracts`, `api_keys`, `kyc_records`, `webhooks`, plus logins by compliance reviewers.
- **Retention**: 7 years hot in Postgres partitions, then S3 Glacier with Object Lock for the legal retention period.
- **Access**: read access scoped to `compliance` and `security` IAM roles; partner self-service export covers only their own `partner_id`.
- **Search**: indexed by `(partner_id, occurred_at)` and `(event_category, event_action)`. Long-running searches go through Athena over the Glacier export.

### 19.4 Compliance-adjacent controls

- **PCI**: payment data is **not** stored in the portal; redirect/iframe to Stripe (current) or NTS settlement (post-cutover). PCI scope kept to SAQ-A.
- **SOC 2 Type II**: controls inventoried in `compliance/soc2/controls.md`; auditor evidence dropped quarterly to a restricted S3 bucket.
- **ISO 27001**: ISMS scope covers the portal; risk register reviewed semi-annually.
- **Subprocessor register**: maintained at `compliance/subprocessors.md` and surfaced on a public page; partners receive 30-day notice of additions.
