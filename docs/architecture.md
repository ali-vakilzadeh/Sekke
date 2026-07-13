# Sekke Draft Architecture (MVP + Evolution Path)

**Date**: July 2026  
**Version**: 0.1 (Draft)  
**Status**: Incorporating user feedback on coin value, wallet receive policy, token lifecycle, anonymous support, DB scalability, and Go preference.

---

## 1. Overall System Goals
Sekke enables **offline digital coins** pegged 1:1 to IRR that behave like physical cash while providing cryptographic auditability and issuer guarantees. The system prioritizes resilience in low-connectivity environments common in Iran.

**Core Principles**:
- Offline-first transfers (peer-to-peer).
- Periodic synchronization for reconciliation and security.
- Strong protection for honest users (no asset loss from sync delays or honest receives).
- Regulatory compliance from day one (CBI sandbox → full license).
- Scalability path to billions of transactions.

---

## 2. Key Updates from Feedback

- **Coin Value**: 1 coin = **200,000 IRR** (≈ 0.1 USD at current rates). This supports practical small-to-medium transactions (e.g., taxi fares, groceries).
- **Wallet Receive Policy (Overflow Handling)**:
  - Wallets can **receive** payments even if it would exceed the local balance limit.
  - Excess coins are flagged for "overflow".
  - App gives a **12-hour deadline** to sync.
  - Upon sync, excess automatically transfers to the user's **online account** (linked bank-like balance).
  - **Scheduled auto-cash-out**: Online account can be configured to auto-transfer to Shetab-linked bank card (e.g., daily at 04:00 AM).
  - This is critical for high-volume receivers like taxi drivers.
- **Token Lifecycle**:
  - Tokens **freeze** (cannot be spent) if not synced past the deadline, but **do not die/expire**.
  - Protects user assets during connectivity issues or exceptional cases.
  - Once synced and validated, tokens are refreshed and unfrozen.
- **Anonymous Wallets**:
  - Allowed with **max 5 coins** limit.
  - Must sync **at least twice per day**.
  - No cash-out capability (spend/receive only).
- **Database**:
  - Start with **PostgreSQL**.
  - Design for horizontal scaling + future migration (e.g., to CockroachDB, TimescaleDB, or sharded setup) to handle billions of transactions.
- **Backend Language**: **Go** (preferred for performance, simplicity, and ease of deployment in Iranian infrastructure).

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
- **Offline Capabilities**: Verify incoming coins, store received tokens, generate proofs.

### 3.2 Issuer Backend (Go)
- **Framework**: Go with standard libraries + minimal frameworks (e.g., Gin/Fiber for API, GORM or sqlx for DB).
- **Modules**:
  - **Token Service**: Issuance of signed coins (simple ECDSA/Ed25519 in MVP).
  - **Ledger & Reconciliation**: Track ownership chains, resolve offline reports, detect double-spends.
  - **Sync Service**: Handle uploads, overflow logic, token refresh, freezing/unfreezing.
  - **User & Wallet Service**: KYC tiers, limits enforcement, anonymous support.
  - **Admin Panel**: Basic web UI (e.g., with Templ or HTMX) for monitoring, manual interventions.
  - **Payment Integration**: Shetab cash-in/out (manual in MVP, API later).
  - **Fraud & Guarantee**: Logging, penalty application (future auto).

### 3.3 Data Layer
- **Primary DB**: PostgreSQL.
  - Tables: users, wallets, tokens (serial, owner history, status), transactions (offline reports), online_balances.
  - Use partitioning, indexes, and proper schema design for high volume.
- **Scalability Path**:
  - Read replicas + sharding by user/wallet or time.
  - Event sourcing / append-only logs for transaction history.
  - Future: Migrate to distributed SQL (e.g., CockroachDB) if needed for billions of txns.
- **Cache**: Redis for hot data (revocation lists, active sessions, rate limiting).

### 3.4 Cryptography (MVP → Full)
- **MVP**: Simple digital signatures for token ownership proofs.
- **Evolution**: Move to blinded signatures / Brands-like e-cash + zero-knowledge proofs for privacy and automatic double-spend revelation.
- **HSM**: Software keys in MVP → Hardware in production.

### 3.5 Infrastructure
- **Hosting**: Iranian data centers (Afranet, Asiatech, MCI, etc.).
- **Deployment**: Docker + simple orchestration (Docker Compose → Kubernetes later).
- **Security**: HTTPS, rate limiting, data residency, local push notifications.
- **Monitoring**: Basic logging + Prometheus/Grafana in later stages.

---

## 4. Data Flows

### Offline Transfer
1. Payer selects coin(s) → generates signed proof.
2. QR exchange → Receiver verifies signature & local rules.
3. Receiver stores coin (even if near/over limit).
4. Payer marks as spent locally.

### Synchronization (Critical Path)
1. App connects → uploads batch of offline transactions.
2. Backend:
   - Reconciles chain of ownership.
   - Detects issues (double-spend, conflicts).
   - Handles overflow: moves excess to online account.
   - Refreshes tokens, unfreezes if valid.
   - Updates revocation lists.
3. Push updates back to app.
4. Online account → optional scheduled auto-cash-out.

---

## 5. MVP vs Full Product

**MVP (6 months)**:
- QR only.
- Simple signatures.
- Manual KYC & cash-in.
- Daily sync target (12h overflow window).
- 500 users, single server.
- PostgreSQL single instance.

**Full Product**:
- NFC + BLE.
- Advanced e-cash crypto.
- Automated KYC, anonymous wallets with rules.
- Tiered limits + deposits.
- HSM, distributed backend.
- Real-time Shetab integration.
- Scalable DB for national volume.

---

## 6. Non-Functional Requirements
- **Performance**: Sync should handle thousands of daily transactions per user efficiently.
- **Reliability**: Offline-first; graceful degradation.
- **Security**: Issuer guarantee + economic incentives.
- **Compliance**: Data in Iran, tiered AML, sandbox first.
- **Scalability**: Designed with billions of txns in mind from the start.

---

## 7. Open Questions / Next Steps
- Detailed coin data model & QR payload format.
- Reconciliation algorithm pseudocode.
- Exact overflow & auto-cash-out business rules.
- Mobile UI/UX wireframes.
- Testing strategy (offline simulation).
