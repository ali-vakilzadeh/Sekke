# Sekke Draft Architecture (MVP + Evolution Path)

**Date**: July 2026 (updated July 18, 2026)
**Version**: 0.2 (Draft)
**Status**: Incorporates prior feedback (coin value, wallet receive policy, token lifecycle, anonymous support, DB scalability, Go preference) **plus** a new Inter-Issuer Compatibility layer.

**Changelog (v0.1 → v0.2)**: Added protocol-level inter-issuer compatibility. Key decisions locked in for this revision:
- **Topology**: Hybrid — mesh (direct bilateral calls) for chain validation between issuers; a central **Sekke Clearing Hub (SCH)** for registry, settlement, and the shared guarantee pool.
- **Cross-issuer orphan-chain risk**: Absorbed by a **shared inter-issuer guarantee pool**, split by formula among participating issuers, rather than by the origin or home issuer alone.
- **Settlement cadence**: Multiple net-settlement windows per day via the SCH (proposed default: 4 windows/day — see §4.4).

Some implementation-level parameters below (exact settlement clock times, pool cost-split formula, exposure resolution window) are proposed defaults flagged for confirmation, not final decisions — see §7.

---

## 1. Overall System Goals
Sekke enables **offline digital coins** pegged 1:1 to IRR that behave like physical cash while providing cryptographic auditability and issuer guarantees. The system prioritizes resilience in low-connectivity environments common in Iran.

**Core Principles**:
- Offline-first transfers (peer-to-peer).
- Periodic synchronization for reconciliation and security.
- Strong protection for honest users (no asset loss from sync delays or honest receives).
- **Interoperability across licensed issuers** — coins are fungible and spendable regardless of which licensed operator issued them (via SIIP).
- **Systemic risk-sharing** — cross-issuer operational risk (orphaned chains) is pooled rather than concentrated on a single issuer.
- Regulatory compliance from day one (CBI sandbox → full license).
- Scalability path to billions of transactions.

---

## 2. Key Updates from Feedback (v0.1, retained)

- **Coin Value**: 1 coin = **200,000 IRR** (≈ 0.1 USD at current rates).
- **Wallet Receive Policy (Overflow Handling)**: wallets can receive beyond the local limit; excess is flagged, given a 12-hour sync deadline, then swept to the user's online account; online account supports scheduled auto-cash-out to Shetab.
- **Token Lifecycle (Hybrid Chain of Spending)**: received tokens are immediately spendable, marked "pending confirmation," reconciled via provisional acceptance + 48–72h grace period; only affected downstream tokens freeze if unresolved.
- **Anonymous Wallets**: max 5 coins, ≥2 syncs/day, no cash-out.
- **Database**: PostgreSQL first, designed for horizontal scaling / future migration.
- **Backend Language**: Go.

---

## 3. Core Components

### 3.1 Mobile App (Flutter)
- **Platforms**: Android (primary), iOS (fallback).
- **Features (MVP)**:
  - Registration & KYC (manual for MVP, tiered: standard + anonymous).
  - Offline QR transfers (send/receive).
  - Local encrypted storage of coins.
  - Balance view (local + online account).
  - Sync client with 12-hour overflow handling.
  - Auto-sync reminders and scheduled background sync (when online).
  - Freeze notification & recovery flow.
- **Offline Capabilities**: Verify incoming coins (including foreign-issuer coins, via cached issuer public keys from the SCH registry), store received tokens, generate proofs.
- **Full Product**: Foreign-issuer discovery — app periodically refreshes a cached copy of the SCH issuer registry so it can verify coins from any licensed issuer offline, not just its home issuer (see §4.3, and open item in §7).

### 3.2 Issuer Backend (Go)
- **Framework**: Go with standard libraries + minimal frameworks (e.g., Gin/Fiber for API, GORM or sqlx for DB).
- **Modules**:
  - **Token Service**: Issuance of signed coins (simple ECDSA/Ed25519 in MVP).
  - **Ledger & Reconciliation**: Track ownership chains, resolve offline reports, detect double-spends.
  - **Sync Service**: Handle uploads, overflow logic, token refresh, freezing/unfreezing.
  - **User & Wallet Service**: KYC tiers, limits enforcement, anonymous support.
  - **Inter-Issuer Module (new)**:
    - *Mesh Validation Client*: resolves foreign-issuer chain segments by calling other issuers' `/validate_chain` directly (bilateral, no hub involvement) during a user's sync.
    - *Hub Settlement Client*: submits settlement batches and receives net positions from the SCH at each settlement window.
    - *Guarantee Pool Reporting*: reports confirmed cross-issuer orphan exposures to the SCH and receives pool payouts on behalf of affected users.
  - **Admin Panel**: Basic web UI (e.g., Templ or HTMX) for monitoring, manual interventions, and now visibility into cross-issuer exposure/pool status.
  - **Payment Integration**: Shetab cash-in/out (manual in MVP, API later).
  - **Fraud & Guarantee**: Logging, penalty application (future auto), plus reporting into the shared pool ledger for cross-issuer cases.

### 3.3 Data Layer
- **Primary DB**: PostgreSQL.
  - Tables: users, wallets, tokens (serial, owner history, status), transactions (offline reports), online_balances.
  - **Token ownership history is now hop-structured**, not a flat hash list, so each hop records which issuer's customer held the token (needed for mesh validation routing and for pool cost-attribution). See SIIP §3.1 for the updated schema.
  - Use partitioning, indexes, and proper schema design for high volume.
- **Scalability Path**: read replicas + sharding by user/wallet or time; event sourcing / append-only logs; future migration to distributed SQL (e.g., CockroachDB) if needed for billions of txns.
- **Cache**: Redis for hot data (revocation lists, active sessions, rate limiting, and now a **cached SCH registry** of foreign issuer public keys/endpoints).

### 3.4 Cryptography (MVP → Full)
- **MVP**: Simple digital signatures for token ownership proofs.
- **Evolution**: Blinded signatures / Brands-like e-cash + zero-knowledge proofs.
- **HSM**: Software keys in MVP → Hardware in production.

### 3.5 Infrastructure
- **Hosting**: Iranian data centers (Afranet, Asiatech, MCI, etc.).
- **Deployment**: Docker + simple orchestration (Docker Compose → Kubernetes later).
- **Security**: HTTPS, mTLS for inter-issuer and hub traffic, rate limiting, data residency, local push notifications.
- **Monitoring**: Basic logging + Prometheus/Grafana in later stages.

### 3.6 Inter-Issuer Layer (new)
This layer is **not part of any single issuer's backend** — it's shared ecosystem infrastructure defined by SIIP.

- **Issuer Mesh**: Direct issuer-to-issuer HTTPS/mTLS connections used exclusively for chain validation and fraud reporting between the two issuers involved in a specific token's chain. No third party is in the loop for this traffic; this keeps validation latency low and avoids a central bottleneck for routine sync operations.
- **Sekke Clearing Hub (SCH)**: A neutral, CBI-run or consortium-run service, responsible for:
  - **Issuer Registry**: authoritative directory of public keys, endpoints, and issuer status (active/suspended). Issuers and mobile apps cache this locally and refresh periodically.
  - **Settlement Netting**: aggregates each issuer's submitted batch of cross-issuer positions at each settlement window and returns a single net position per issuer (see §4.4).
  - **Shared Guarantee Pool**: a pooled reserve, funded by participating issuers, that absorbs confirmed cross-issuer orphaned-chain exposure so that no single issuer bears disproportionate risk for a chain that happened to pass through several issuers' customers (see §4.6 and SIIP §4.6).
  - **CRL Aggregation**: collects fraud/double-spend reports from the mesh and publishes a consolidated revocation list.
  - **Audit Trail**: settlement and pool activity logged for CBI oversight.

The MVP runs with SIIP stubs only (single issuer, no live mesh/hub traffic — see §5). The Full Product activates the mesh and SCH once a second licensed issuer is onboarded.

---

## 4. Data Flows

### 4.1 Offline Transfer (unchanged)
1. Payer selects coin(s) → generates signed proof.
2. QR exchange → Receiver verifies signature & local rules (including verifying a foreign issuer's signature via cached registry keys, if applicable).
3. Receiver stores coin (even if near/over limit).
4. Payer marks as spent locally.

### 4.2 Synchronization (Critical Path, updated)
1. App connects → uploads batch of offline transactions to its **home issuer**.
2. Home issuer:
   - Reconciles chain segments it can resolve locally (same-issuer hops).
   - For hops belonging to **other issuers**, triggers Cross-Issuer Chain Validation (§4.3) via the mesh.
   - Detects issues (double-spend, conflicts) using combined local + mesh results.
   - Handles overflow: moves excess to online account.
   - Refreshes tokens, unfreezes if valid.
   - Updates revocation lists (local + SCH-aggregated CRL).
3. Push updates back to app.
4. Online account → optional scheduled auto-cash-out.

### 4.3 Cross-Issuer Offline Transfer & Chain Validation (new)
1. A token can circulate through customers of several issuers before anyone syncs — the token itself doesn't care whose app is holding it, only its origin issuer's signature matters for offline verification.
2. Each hop's ownership-chain entry now records which issuer's customer held the token at that hop (see SIIP §3.1).
3. When a user syncs, their home issuer inspects the chain: any hop tagged with a different issuer triggers a direct, bilateral `/validate_chain` call to that issuer (looked up via the cached SCH registry) — **not** routed through the hub.
4. The foreign issuer checks the segment against its own customer's sync history and returns a validation status + risk score.
5. The home issuer combines all responses (its own + foreign) to finalize the token's status: confirmed, still provisionally accepted (within grace period), or frozen.

### 4.4 Multi-Issuer Settlement (Hub Netting, new)
1. **Proposed default**: settlement windows run 4x/day, aligned to existing operational rhythm — approx. 04:00 (right after auto-cash-out), 10:00, 16:00, 22:00 Iran time — *flagged for confirmation, not final*.
2. At each window, every issuer submits a batch to the SCH (`/submit_batch`) summarizing confirmed cross-issuer positions accumulated since the last window (e.g., value of foreign-issued tokens redeemed/absorbed by this issuer's users, and vice versa).
3. SCH computes minimal net positions across all issuers (multilateral netting, not just pairwise sums) and returns one net position per issuer (`/net_position`).
4. Issuers confirm (`/confirm_settlement`) and execute transfers via Shetab/CBI settlement rails.
5. SCH logs the window to the CBI audit trail.

### 4.5 Cross-Issuer Orphaned Chain → Shared Guarantee Pool (new)
See SIIP §4.6 for full detail. Summary:
1. Grace period (48–72h) lapses on a chain segment with an unresponsive intermediate holder who belongs to a *different* issuer than the affected downstream holder.
2. The affected holder's home issuer freezes only the affected downstream tokens (unchanged principle) and reports the exposure to the SCH.
3. If the segment resolves later (delayed sync), the case closes with no pool payout — normal unfreeze.
4. If unresolved past a resolution window (**proposed default: 14–30 days, open for confirmation**), the SCH pays the affected holder's home issuer from the pool (so the end user is made whole, per the existing "no loss to honest users" guarantee), then debits participating issuers per the agreed cost-split formula (**proposed default: pro-rata by each issuer's share of the token's value that passed through their customers in that specific chain, falling back to equal split if attribution is ambiguous — open for CBI/consortium sign-off**).

---

## 5. MVP vs Full Product

**MVP (6 months)**:
- QR only. Simple signatures. Manual KYC & cash-in.
- Daily sync target (12h overflow window).
- 500 users, single server, single issuer.
- PostgreSQL single instance.
- **SIIP protocol stubs only** — token schema and endpoints scaffolded, but no live second issuer, no live mesh traffic, no SCH in production use.

**Full Product**:
- NFC + BLE. Advanced e-cash crypto. Automated KYC, anonymous wallets with rules.
- Tiered limits + deposits. HSM, distributed backend. Real-time Shetab integration.
- Scalable DB for national volume.
- **Multi-issuer live**: Issuer Mesh active for chain validation; Sekke Clearing Hub live for registry, multi-daily settlement, and the shared guarantee pool.

---

## 6. Non-Functional Requirements
- **Performance**: Sync should handle thousands of daily transactions per user efficiently; mesh validation calls add latency per foreign hop, so home issuers should batch/parallelize foreign-issuer queries during sync rather than serialize them.
- **Reliability**: Offline-first; graceful degradation; SCH settlement failures must not block user-facing sync/unfreeze flows (settlement is a backend-to-backend concern, decoupled from the user's sync response).
- **Security**: Issuer guarantee + economic incentives. Hybrid reconciliation manages orphaned chains via grace periods and selective freezing. Cross-issuer orphan exposure is additionally backstopped by the shared guarantee pool, on top of (not instead of) each issuer's own single-issuer guarantee fund.
- **Compliance**: Data in Iran, tiered AML, sandbox first; SCH settlement and pool activity logged for CBI audit.
- **Scalability**: Designed with billions of txns in mind from the start; SCH settlement volume must scale independently of any single issuer's transaction volume.

**Operational Risk Note**: The hybrid approach carries a potential risk of accumulated orphaned chains increasing financial exposure. Within a single issuer this is managed via monitoring, sync incentives, rate limits, and timely intervention (see `orphan_chain_risk_mitigation.md`). Across issuers, the same risk is additionally mitigated by pooling — but pooling only redistributes cost, it doesn't reduce the underlying orphan rate, so the existing single-issuer mitigations remain necessary regardless of how many issuers participate.

---

## 7. Open Questions / Next Steps

**Resolved this revision**: mesh+hub hybrid topology; shared guarantee pool for cross-issuer orphan risk; multi-daily settlement cadence.

**Still open (need your or CBI/consortium sign-off)**:
- Exact settlement window clock times (proposed: 04:00/10:00/16:00/22:00 Iran time).
- Exact pool cost-split formula and pool collateral sizing/top-up mechanics.
- Exact resolution-window length before a cross-issuer orphan triggers a pool payout (proposed: 14–30 days).
- Mobile client strategy for discovering/caching foreign issuer keys offline (bundled registry snapshot vs. on-demand fetch when online).
- Fee structure for mesh `/validate_chain` queries between issuers (free/reciprocal vs. metered).
- `data_model.md` will need a matching update to the token/ownership-chain schema (hop-structured, issuer-tagged) — not yet done, can do on request.
- Detailed coin data model & QR payload format (carry-over from v0.1).
- Reconciliation algorithm pseudocode (carry-over).
- Mobile UI/UX wireframes; offline simulation testing strategy (carry-over).
