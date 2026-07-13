# Sekke - Business Data Model & Data Protection (Privacy)

**Date**: July 2026  
**Version**: 0.1  
**Compliance Focus**: Iran Personal Data Protection Act, CBI regulations, data residency, and GDPR-like principles (minimization, consent, security, rights).

---

## 1. Business Data Model Overview

### Core Entities

**User**
- Attributes: User ID (internal), Phone Number, National ID (Shahkar verified), Full Name, Biometrics hash (optional), Registration Date, KYC Level (Standard / Anonymous).
- Stored by: Backend (Issuer) + Local App (minimal).
- Purpose: Authentication, KYC/AML compliance.

**Wallet**
- Attributes: Wallet ID, User ID (link), Wallet Type (Standard/Anonymous), Current Balance (local + online), Limit, Security Deposit (if any), Status (Active/Frozen).
- Stored by: Backend (authoritative) + Mobile App (local view).
- Purpose: Balance management, limit enforcement.

**Token (Coin)**
- Attributes: Token Serial (unique), Value (200,000 IRR), Current Owner (Wallet ID), Status (Active/Frozen/Pending/ Redeemed), Issuance Date, Last Sync Timestamp, Ownership History (list of transfers - hashes).
- Stored by: Backend (ledger) + Mobile App (local possession).
- Purpose: Represent value, enable offline transfer, audit trail.

**Transaction (Offline)**
- Attributes: Tx ID, Token Serial, From Wallet, To Wallet, Timestamp, Cryptographic Proof (signature), Sync Status.
- Stored by: Mobile App (local batch) + Backend (after sync).
- Purpose: Record transfers for reconciliation.

**Online Account**
- Attributes: User ID, Balance (IRR), Linked Bank Card (masked), Auto Cash-out Settings.
- Stored by: Backend only.
- Purpose: Overflow handling and cash-out.

**Audit Log**
- Attributes: Event Type, User/Wallet ID, Timestamp, Action, Outcome.
- Stored by: Backend.
- Purpose: Compliance, fraud investigation.

---

## 2. Data Storage Responsibility

| Data Type              | Mobile App (Local)          | Issuer Backend (Central)       | Why Stored There |
|------------------------|-----------------------------|--------------------------------|------------------|
| User Profile / KYC     | Minimal (phone hash)       | Full details                   | Regulatory compliance |
| Wallet Balance         | Full local view             | Authoritative + online portion | Offline operation + reconciliation |
| Tokens (Coins)         | Current possession          | Global ledger + history        | Bearer instrument + audit |
| Offline Transactions   | Batch (until sync)          | Permanent after sync           | Enable offline + reconciliation |
| Online Account         | View only                   | Full                           | Financial settlement |
| Revocation Lists       | Cached copy                 | Master                         | Security checks |

**Data Minimization**: Anonymous wallets store almost no PII. Standard wallets only collect necessary KYC.

---

## 3. Storage Modes & Security
- **Mobile App**: Encrypted local storage (Flutter secure storage / SQLCipher). Tokens stored with cryptographic material.
- **Backend**: PostgreSQL with encryption at rest (pgcrypto or column-level). All servers in Iranian data centers.
- **Retention**: 
  - Active data: As long as account is open.
  - Transaction logs: 7 years (regulatory requirement).
  - Deleted user data: Anonymized where possible.

---

## 4. Connection, Transfer & Protocols
- **Offline Mode**: No connection. Local verification using public keys/signatures.
- **Sync (Online)**: HTTPS (TLS 1.3) to Iranian-hosted endpoints. Mutual TLS where possible.
- **Data Transfer**: 
  - Only delta changes during sync (batched).
  - No raw sensitive data transferred offline.
- **Push Notifications**: Local Iranian provider (e.g., Pushe) — minimal data.
- **Authentication**: JWT + device binding. Biometrics for app access.

---

## 5. Security Protocols & Data Integrity
- **Cryptography**: ECDSA/Ed25519 signatures for tokens. Hash chains for ownership history.
- **Integrity**:
  - Digital signatures on all tokens/transactions.
  - Backend reconciliation verifies chain consistency.
  - Checksums + Merkle-tree style proofs for batches.
- **Access Control**: Role-based (RBAC) on backend. Least privilege.
- **Protection Against Threats**:
  - Double-spend: Cryptographic + economic (guarantee fund).
  - Data Breach: Encryption + residency.
  - Tampering: Signature validation on every transfer/sync.
- **Backup & Recovery**: Encrypted backups with versioning.

---

## 6. Privacy & User Rights (GDPR-like + Iranian Law)
- **Consent**: Explicit during registration. Granular for data processing.
- **Data Subject Rights**:
  - Access, correction, deletion (where not conflicting with AML).
  - Right to be forgotten (limited by financial regulations).
- **Tiered KYC**: Anonymous wallets minimize data collection.
- **Data Residency**: All PII and transaction data stays inside Iran.
- **Breach Notification**: Internal policy to notify users and regulators as required.
- **Data Protection Officer**: To be appointed for full license stage.

---

## 7. Open Items
- Detailed database schema (ER diagram).
- Exact retention policy per data type.
- Privacy policy text for the app.

This model ensures **functional offline operation** while maintaining strong compliance and security.

**Next**: We can create the detailed DB schema or privacy policy text. Let me know your feedback.
