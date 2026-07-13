# Sekke MVP Requirements - Detailed

**Date**: July 2026  
**Version**: 0.1  
**Scope**: Minimum Viable Product (Target: 6 months, 500 users pilot)

---

## 1. Objectives
- Prove **offline peer-to-peer digital coin transfers** work reliably.
- Validate **reconciliation** of offline transactions on the backend.
- Test real-world UX in a controlled Iranian environment (e.g., Tajrish Bazaar).
- Gather data on adoption, fraud attempts, and technical performance.
- Stay within CBI fintech sandbox constraints.

---

## 2. Key Parameters
- **Coin Value**: 1 coin = **200,000 IRR** (≈ 0.1 USD).
- **Pilot Size**: Max 500 users.
- **Wallet Limit (Standard)**: 10 coins (local balance).
- **Anonymous Wallets**: Max 5 coins, minimum 2 syncs per day.
- **Sync Window**: 12 hours for overflow handling; tokens **freeze** (not destroyed) if delayed beyond deadline.

---

## 3. Functional Requirements

### 3.1 User Mobile App (Flutter)
**Core Flows**:
- **Registration & Onboarding**
  - Phone number + manual KYC (National ID via Shahkar).
  - Choice between Standard or Anonymous wallet.
- **Cash-in**: Manual (off-app bank transfer → admin approval credits coins).
- **Cash-out**: Online via Shetab-linked card (when synced).
- **Offline Send/Receive** (QR Code only):
  - Select coins → Generate QR with signed proof.
  - Scan QR → Verify signature and rules → Accept (even if near/over limit).
- **Overflow Handling (Receive)**:
  - Allow receiving beyond local limit.
  - Flag excess as "pending overflow".
  - 12-hour deadline to sync.
  - Upon sync: Excess auto-moves to **online account**.
- **Online Account**:
  - Holds overflow funds.
  - Supports scheduled **auto-cash-out** (configurable, default 04:00 AM daily).
- **Sync**:
  - Automatic when online.
  - Uploads offline transactions.
  - Receives refreshed coins + status updates.
- **Token Status**:
  - Frozen tokens: Visible but unspendable until synced and validated.
- **Other**:
  - Balance view (local coins + online account).
  - Transaction history.
  - Notifications for sync deadlines and freezes.

### 3.2 Issuer Backend (Go)
- **Token Issuance**: Generate signed coins (simple digital signatures).
- **Ledger**: Track token serials, ownership history, status (active/frozen).
- **Reconciliation Engine**:
  - Process batch uploads from apps.
  - Resolve multi-hop ownership chains.
  - Detect basic conflicts / double-spend attempts.
- **Sync Service**:
  - Handle overflow logic.
  - Refresh tokens.
  - Freeze/unfreeze logic.
- **User & Wallet Management**:
  - Tiered wallets (Standard vs Anonymous).
  - Limit enforcement.
- **Admin Panel**:
  - Manual KYC approval.
  - Cash-in approval.
  - Monitor pilot activity, balances, sync status.
  - Basic fraud review.
- **Payment Integration**: Manual Shetab cash-out in MVP.

### 3.3 Security & Compliance
- Simple software-based signing keys (tight controls).
- Data residency in Iran.
- Tiered KYC (basic for standard, none for anonymous).
- Issuer guarantee: Honest users protected.
- Logging and audit trails.

---

## 4. Non-Functional Requirements
- **Performance**: Sync < 5 seconds for typical daily volume.
- **Reliability**: Offline-first; graceful handling of poor connectivity.
- **Scalability**: PostgreSQL schema designed for future growth (partitioning, indexes). Easy migration path for billions of transactions.
- **Usability**: Simple UX for non-tech users (bazaar merchants, taxi drivers).
- **Security**: Cryptographic verification of transfers; no loss for honest participants.

---

## 5. Out of Scope (MVP)
- NFC / BLE.
- Advanced cryptography (blinding, ZKP, auto double-spend reveal).
- Security deposits & higher limits.
- Automated KYC / liveness.
- Real-time PSP integration.
- HSM.
- Merchant premium tools.

---

## 6. Success Criteria
- ≥ 80% of pilot users complete at least one offline transfer.
- Reconciliation engine correctly resolves chains in 99%+ of cases.
- No unrecoverable loss for honest users.
- Positive feedback from participants.
- Technical stability during daily syncs.
- Sandbox compliance demonstrated.

---

## 7. Assumptions & Dependencies
- Access to Shahkar for basic verification.
- Iranian hosting provider.
- Small pilot group for controlled testing.

This document will evolve. Next: Detailed technical specs (data models, API, crypto protocol).
