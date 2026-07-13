# Sekke Inter-Issuer Protocol (SIIP) - Draft Specification

**Version**: 0.1 (Draft)  
**Date**: July 13, 2026  
**Status**: Initial proposal for multi-operator support in Sekke ecosystem.  
**Audience**: Licensed Issuers (EMIs), CBI regulators, developers.

## 1. Introduction

The Sekke Inter-Issuer Protocol (SIIP) enables multiple licensed operators (banks, fintechs, PSPs) to issue and manage Sekke digital coins while ensuring **interoperability**, **security**, and **regulatory compliance**. 

Users can hold and transfer coins across different issuers without being locked into a single app or operator. Coins remain fungible within the national monetary base (1:1 IRR peg).

**Goals**:
- Seamless P2P transfers regardless of issuer.
- Secure cross-issuer reconciliation and settlement.
- Fraud containment (double-spend detection across operators).
- Minimal trust assumptions; strong cryptography.
- Compliance with CBI rules, data residency, AML.

**Non-Goals** (MVP SIIP):
- Real-time gross settlement (batch/netting preferred).
- Full decentralization (relies on licensed participants).

## 2. High-Level Architecture

- **Issuers**: Licensed entities operating their own Sekke backend (reference or fork).
- **Issuer Registry**: CBI-maintained (or consortium-hosted) directory of public keys, endpoints, and status.
- **Hub (Optional)**: Neutral clearing hub (CBI or shared infrastructure) for batch settlement and CRL propagation.
- **Mobile Apps**: Any compliant Sekke client (reference Flutter or forks) — binds to user's home issuer for sync but verifies foreign coins.

**Communication**:
- Backend-to-backend: HTTPS/TLS 1.3 + mTLS (mutual authentication).
- Messages: JSON over REST or gRPC (for efficiency).

## 3. Core Data Structures

### 3.1 Token (Enhanced)
```json
{
  "serial": "string (unique UUID or issuer-prefixed)",
  "issuer_id": "string (e.g., 'bank-mellat-001')",
  "value": 200000,  // IRR subunits
  "issuance_date": "ISO8601",
  "ownership_chain": ["hash1", "hash2", ...],  // Merkle-like or linked proofs
  "signature": "cryptographic blob (verifiable by issuer pubkey)",
  "status": "active | frozen | redeemed"
}
```

- Coins include issuer ID for routing.
- Full blinding/ZKP in production for privacy.

### 3.2 Transaction Proof
- Signed proof of transfer (payer → receiver).
- Includes previous owner proof for chain validation.

## 4. Protocol Flows

### 4.1 Coin Issuance & Redemption
- Only home issuer creates new coins (backed by reserves).
- Redemption: User requests cash-out → home issuer settles (possibly via SIIP if chain involves others).

### 4.2 Offline Transfer (Unchanged for Users)
- P2P via QR/NFC/BLE works across issuers (app verifies foreign issuer pubkey via registry).

### 4.3 Sync & Cross-Issuer Validation
1. User syncs with **home issuer**.
2. Home issuer:
   - Processes local chain.
   - For foreign tokens: Queries originating issuer(s) via SIIP (`/validate_chain`).
3. Response includes validation status, any revocations.

### 4.4 Batch Settlement (Daily/Periodic)
- Issuers exchange net positions (e.g., "Issuer A owes Issuer B 150M IRR").
- Settle via Shetab or CBI accounts.
- SIIP endpoints: `/submit_batch`, `/confirm_settlement`.

### 4.5 Revocation & Fraud Propagation
- Double-spend proofs shared via `/report_fraud`.
- Shared CRL (Certificate Revocation List) or delta updates pushed via hub or polling.
- Issuer responsible for penalizing their users (collateral seizure).

## 5. API Endpoints (REST Example)

**Base**: `https://issuer.example.com/siip/v1/`

- `GET /registry` — Fetch public issuer directory (cached).
- `POST /validate_chain` — Submit ownership chain snippet; returns validation + risk score.
- `POST /report_fraud` — Submit double-spend proof (reveals identity to authorities).
- `POST /submit_batch` — Net settlement positions.
- `POST /query_token_status` — For specific serials.
- `GET /crl` — Current revocation list.

**Authentication**: mTLS + JWT (issuer-scoped).

## 6. Security & Trust Model

- **Cryptography**: ECDSA/Ed25519 (MVP) → BLS/ZKP.
- **Issuer Onboarding**: CBI approval + public key registration.
- **Economic Security**: Each issuer maintains collateral/reserves proportional to issuance.
- **Threat Mitigation**:
  - Rogue issuer: Revocation by registry + CBI intervention.
  - Chain attacks: Grace periods + partial validation.
  - Privacy: Minimize data shared (only necessary proofs).

## 7. Compliance & Operations

- All messages logged for audit (CBI access).
- Data residency: All inter-issuer traffic within Iran.
- Tiered KYC respected across operators.
- Dispute resolution: Escalate to CBI.

## 8. Implementation Roadmap

**Phase 1 (MVP Extension)**: Single issuer with SIIP stubs.
**Phase 2**: Multi-issuer support in reference backend + protocol tests.
**Phase 3**: Production with banks, shared hub.

## 9. Open Questions
- Exact settlement frequency and netting rules.
- Hub vs. fully meshed P2P between issuers.
- Fee structure for cross-issuer queries.
- Mobile client discovery of foreign issuers.

---

**Next Steps**: Review by stakeholders, formalize crypto details, add sequence diagrams, implement reference endpoints.

*Feedback welcome — this enables a competitive yet unified Sekke ecosystem.*
