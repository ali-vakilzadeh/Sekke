# Sekke Draft Architecture (MVP + Evolution Path)

**Date**: July 2026  
**Version**: 0.2 (Draft)  
**Status**: Incorporating user feedback on coin value, wallet receive policy, token lifecycle, anonymous support, DB scalability, and Go preference. Updated per the pre-seed risk analysis (v0.2) to reflect a revised network assumption (always reachable, but latency of 5–240 sec) and a concrete anti-fraud/key-custody mitigation architecture agreed during risk review.

---

## 1. Overall System Goals
Sekke enables **offline digital coins** pegged 1:1 to IRR that behave like physical cash while providing cryptographic auditability and issuer guarantees. The system prioritizes resilience in low-connectivity environments common in Iran.

**Core Principles**:
- Offline-first transfers (peer-to-peer).
- Continuous background synchronization for reconciliation and security — **network assumption updated**: the client network is treated as *always eventually reachable, but slow* (server responses of 5–240 seconds should be expected and designed around), rather than genuinely offline for extended periods. Multi-day disconnection is retained only as a rare fallback case (see §4).
- Strong protection for honest users (no asset loss from sync delays or honest receives).
- Regulatory compliance from day one (CBI sandbox → full license).
- Scalability path to billions of transactions.
- **Device floor**: Android 9+, low-end ARM hardware, no guaranteed hardware-backed keystore or secure element. All client-side cryptography and anti-fraud logic must run comfortably on this floor — this rules out pairing-based schemes (e.g., BLS aggregation, ZK-proofs) for the MVP in favor of standard EdDSA/ECDSA operations.

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
- **Token Lifecycle (Hybrid Chain of Spending — revised anti-fraud design)**:
  - Received tokens are immediately spendable (cash-like UX).
  - Marked as "pending confirmation" until a checkpoint resolves (see below) — no longer until a multi-day sync.
  - **In-flight value budget & checkpoint cadence**: each wallet caps total unconfirmed outstanding value at **5 coins**. A background, non-blocking checkpoint call to the server fires on whichever comes first: 5 coins of cumulative unconfirmed activity, or a **30-second** cooldown since the last checkpoint. Given the always-reachable-but-slow network assumption, this resolves within 5–240 seconds rather than being deferred for days.
  - **Hop-count tagging & "hot coin" flagging**: every token carries a hop counter plus one signature per hop — `(serial, hop_index, receiver_id_hash, hash(prev_signature))` — signed by the transferring holder with a full Ed25519 signature (64 bytes; not truncated, to preserve verifiability and collision resistance). Crossing a threshold **H\* = 5** hops flags the token "hot": it cannot be spent again until an online validation confirms it. Combined with the in-flight budget, this gives a designed (not estimated) bound: **worst-case fraud radius per malicious actor ≤ (in-flight budget) × (hop cap)**.
  - **Cryptographic equivocation proof**: because every hop is signed with a strictly increasing sequence number, two divergent signed messages for the same `(serial, sequence_number)` from the same key are mathematically irrefutable, non-repudiable proof of double-spending — detectable at any point in the chain, not only at the fork's origin, and provable to a third party (regulator, another issuer under SIIP) without ambiguity. Detection at the first checkpoint typically triggers **permanent ban** of the offending wallet, not merely a delay.
  - **Local revocation cache & rollback protection**: each device maintains a compact Bloom/Cuckoo filter (a few KB, refreshed at each checkpoint) as a near-instant local first-line check before accepting a token, and caches the highest hop-index seen per serial to catch rollback attempts (a dishonest holder re-presenting an earlier, lower-hop-count state of an already-spent token). Both run as an internal client-side process inside the app, with no added server round-trip beyond the normal checkpoint cadence.
  - **QR payload encoding**: transfer payloads are encoded as compact binary (protobuf/CBOR/msgpack) rather than human-readable JSON, optionally with a light app-embedded obfuscation layer. This is a defense-in-depth/hygiene measure against casual tampering by non-specialist actors using generic QR readers — it is not a substitute for the cryptographic signature, which remains the actual integrity/authenticity guarantee.
  - **Fallback mode**: the original 48–72h grace-period logic (provisional chain acceptance for delayed nodes) is retained in code as a fallback path for the rare case of genuine multi-day connectivity outages, so the system degrades gracefully rather than failing outright if the "always reachable" network assumption is ever wrong in practice.
  - Tokens may still be frozen only for affected downstream users if a chain segment genuinely cannot be resolved.
  - Strong protection for honest users via issuer guarantee, sized against the enhanced (post-mitigation) exposure model rather than the original unmitigated one — see the risk analysis report, §3.
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
  - Offline QR transfers (send/receive), payload encoded as compact binary (protobuf/CBOR), not plaintext JSON.
  - Local encrypted storage of coins: Ed25519/secp256r1 client keys, using Android Keystore hardware backing opportunistically where available, with a fallback to an app-level encrypted store (SQLCipher/Tink) on devices without it — hardware backing is never a hard requirement, given the Android 9+/low-end-device floor.
  - Balance view (local + online account), including a "hot coin" indicator for tokens awaiting hop-cap revalidation.
  - Checkpoint client: dual-trigger background sync (5 coins unconfirmed / 30 sec cooldown), replacing the old "sync when online" model with continuous, non-blocking background checkpoints, plus separate 12-hour overflow handling for wallet-limit excess (unchanged, see §2).
  - Local Bloom/Cuckoo revocation cache and hop-index cache for rollback protection, refreshed at each checkpoint.
  - Auto-sync reminders and scheduled background sync (now continuous rather than periodic, given the always-reachable-but-slow network assumption).
  - Freeze / hot-coin notification & recovery flow.
- **Offline Capabilities**: Verify incoming coins against the local revocation cache and hop-count rules, store received tokens, generate hop-signed proofs.

### 3.2 Issuer Backend (Go)
- **Framework**: Go with standard libraries + minimal frameworks (e.g., Gin/Fiber for API, GORM or sqlx for DB).
- **Modules**:
  - **Token Service**: Issuance of signed coins (ECDSA/Ed25519 in MVP), under a **split-key issuance** scheme — a separate issuance key and authorization key, the latter signing only after verifying an approved cash-in record — plus **threshold signing** (e.g., Shamir/threshold-ECDSA, t-of-n) so no single machine ever holds the complete issuer key, and an **issuance-rate circuit breaker** hard-capping tokens mintable per unit time regardless of requester.
  - **Ledger & Reconciliation**: Track ownership chains (including per-hop signatures and hop index), resolve offline reports, detect double-spends via cryptographic equivocation-proof checks, and run a **continuous reconciliation invariant** (`sum(active token value) == sum(approved cash-in value)` on every batch) so unauthorized issuance is mathematically detectable rather than dependent on manual audit.
  - **Checkpoint Service** (renamed from Sync Service): Handle the dual-trigger checkpoint calls (5 coins / 30 sec), overflow logic, hop-cap/hot-coin revalidation, token refresh, freezing/unfreezing, and permanent-ban actions following a proven equivocation.
  - **User & Wallet Service**: KYC tiers, limits enforcement, anonymous support.
  - **Admin Panel**: Basic web UI (e.g., with Templ or HTMX) for monitoring and manual interventions, with **multisig-style dual control** (no single admin action executes without a second independent approval), an **append-only hash-chained audit log** (each entry includes the hash of the previous one), and **automated anomaly flags** (unusual cash-in size, approval velocity per admin, round-number patterns) holding transactions for secondary review.
  - **Payment Integration**: Shetab cash-in/out (manual in MVP, API later).
  - **Fraud & Guarantee**: Logging, non-repudiable equivocation-proof storage, penalty application (future auto).

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
- **MVP, issuer-side**: split-key issuance (separate issuance + authorization keys) and threshold signing (t-of-n) so no single machine ever holds the full issuer key — pure software, no hardware dependency, deployable before any HSM is procured.
- **MVP, client-side**: Ed25519/secp256r1 signing, a few milliseconds even on old ARM hardware; every token hop is individually signed (not truncated) to support the hop-count/equivocation-proof scheme in §2. Pairing-based schemes (BLS aggregation, ZK-proofs) are explicitly out of scope for MVP given the Android 9+/no-guaranteed-secure-element device floor.
- **Evolution**: Move to blinded signatures / Brands-like e-cash + zero-knowledge proofs for privacy and automatic double-spend revelation, once device and connectivity assumptions allow the additional computational cost.
- **HSM**: Software keys (split-key + threshold signing) in MVP → Hardware in production. Hardware procurement should be planned for with extra lead time given export-restriction exposure on HSM hardware.

### 3.5 Infrastructure
- **Hosting**: Iranian data centers (Afranet, Asiatech, MCI, etc.).
- **Deployment**: Docker + simple orchestration (Docker Compose → Kubernetes later).
- **Security**: HTTPS, rate limiting, data residency, local push notifications.
- **Monitoring**: Basic logging + Prometheus/Grafana in later stages.

---

## 4. Data Flows

### Offline Transfer
1. Payer selects coin(s) → generates a hop-signed proof (`serial, hop_index, receiver_id_hash, hash(prev_signature)`), encoded as compact binary.
2. QR exchange → Receiver checks the local Bloom/Cuckoo revocation cache and hop-index cache, then verifies signature & local rules.
3. Receiver stores coin (even if near/over limit); if hop count crosses **H\*=5**, the coin is flagged "hot" pending online validation before it can be spent onward.
4. Payer marks as spent locally.
5. Receiver's checkpoint budget accrues; a background checkpoint fires per the dual trigger (5 coins unconfirmed / 30 sec) independent of the payer's cooperation.

### Checkpoint & Reconciliation (Critical Path — revised)
1. App's background checkpoint client connects (always assumed reachable; response expected within 5–240 sec) → uploads the pending batch of transaction proofs.
2. Backend:
   - Reconciles chain of ownership, including hop signatures.
   - Runs the cryptographic equivocation-proof check: any divergent signed claim on the same `(serial, sequence_number)` is treated as non-repudiable proof of fraud, triggering immediate freeze of the affected downstream branch and permanent ban of the offending wallet.
   - Handles overflow: moves excess to online account (unchanged 12h logic, independent of the fraud-detection checkpoint).
   - Refreshes tokens, unfreezes if valid, clears hot-coin flags on validated tokens.
   - Updates revocation lists / Bloom filter snapshot pushed to clients.
3. Push updates back to app.
4. Online account → optional scheduled auto-cash-out.
5. **Fallback**: if a genuine multi-day outage prevents any checkpoint from resolving, the app falls back to the original 48–72h provisional-acceptance logic rather than blocking spending entirely.

---

## 5. MVP vs Full Product

**MVP (6 months)**:
- QR only, binary-encoded payload.
- Hop-signed Ed25519 signatures with equivocation-proof detection (not just "simple signatures" — see §2).
- Manual KYC & cash-in.
- Continuous background checkpointing (5 coins / 30 sec trigger) for fraud detection; separate 12h overflow window for wallet-limit excess.
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
- **Security**: Issuer guarantee + economic incentives. Fraud exposure is bounded by design — in-flight value budget × hop cap — and detected via non-repudiable cryptographic equivocation proof, rather than relying primarily on grace-period timeouts. Grace periods and selective freezing are retained only as a fallback for genuine connectivity outages.
- **Compliance**: Data in Iran, tiered AML, sandbox first.
- **Scalability**: Designed with billions of txns in mind from the start.

**Operational Risk Note**: A residual risk remains from **collusive rings** — Sybil-controlled wallets that never independently checkpoint — which could delay detection longer than the honest-counterparty case the equivocation-proof bound assumes. This is a harder, more economic/behavioral problem than a cryptographic one, and should be managed via reputation-scoring, anomaly-detection monitoring, user incentives for timely checkpointing, and timely intervention on flagged chains, rather than assumed away by the cryptographic controls alone.

---

## 7. Open Questions / Next Steps
- ~~Detailed coin data model & QR payload format~~ — resolved in principle (hop-signed binary payload, §2); needs a finalized wire-format spec.
- Reconciliation algorithm pseudocode, including the equivocation-proof check and hop-cap enforcement.
- Exact overflow & auto-cash-out business rules (unchanged from the original 12h design).
- Mobile UI/UX wireframes, including the "hot coin" wait/notification state.
- Testing strategy (offline simulation), extended to cover the 5–240 sec latency range and fan-out/rollback attack simulations.
- Instrumentation plan for measuring the real-world fan-out parameter (k′) and checkpoint-trigger frequency during the pilot, so the illustrative figures in the risk analysis report can be replaced with measured values.
- Reputation-scoring/anomaly-detection design for the collusive-ring residual risk (currently unaddressed by the cryptographic controls alone).
- SIIP integration of the equivocation-proof format for cross-issuer fraud claims (forward-looking, post-MVP).
