# Sekke Inter-Issuer Protocol (SIIP) - Draft Specification

**Version**: 0.2 (Draft)
**Date**: July 18, 2026
**Status**: Formalizes topology, cross-issuer orphan-chain handling, and settlement cadence for multi-operator support.
**Audience**: Licensed Issuers (EMIs), CBI regulators, developers.

**Changelog (v0.1 → v0.2)**:
- Topology decided: **hybrid mesh (validation) + hub (settlement)**, replacing the "Hub (Optional)" framing.
- New mandatory entity: **Sekke Clearing Hub (SCH)**.
- New mechanism: **Shared Inter-Issuer Guarantee Pool** for cross-issuer orphaned-chain exposure.
- Settlement cadence decided: multiple windows per day (proposed default: 4/day) instead of daily.
- Token schema updated: `ownership_chain` is now a structured, issuer-tagged hop list instead of a flat hash array.
- Some parameters (pool formula, pool sizing, exact window times, resolution-window length, mesh fee structure) remain open pending CBI/consortium agreement — flagged explicitly rather than silently decided.

## 1. Introduction

The Sekke Inter-Issuer Protocol (SIIP) enables multiple licensed operators (banks, fintechs, PSPs) to issue and manage Sekke digital coins while ensuring **interoperability**, **security**, and **regulatory compliance**.

Users can hold and transfer coins across different issuers without being locked into a single app or operator. Coins remain fungible within the national monetary base (1:1 IRR peg).

**Goals**:
- Seamless P2P transfers regardless of issuer.
- Secure cross-issuer reconciliation and settlement.
- Fraud containment (double-spend detection across operators).
- Minimal trust assumptions; strong cryptography.
- Compliance with CBI rules, data residency, AML.
- **Fair distribution of cross-issuer operational risk** (new — see §4.6).

**Non-Goals** (MVP SIIP):
- Real-time gross settlement (batch/netting preferred).
- Full decentralization (relies on licensed participants).

## 2. High-Level Architecture

- **Issuers**: Licensed entities operating their own Sekke backend (reference or fork).
- **Issuer Mesh**: Direct issuer-to-issuer connections (HTTPS/mTLS) used for chain validation and bilateral fraud reporting. This is the *validation* layer — no third party sits between two issuers resolving a specific token's chain.
- **Sekke Clearing Hub (SCH)** *(formerly "Hub (Optional)" — now mandatory once ≥2 issuers are live)*: A neutral entity (CBI-run or consortium-hosted) responsible for:
  - Issuer Registry (public keys, endpoints, status).
  - Multi-daily net settlement between issuers.
  - Administration of the Shared Inter-Issuer Guarantee Pool.
  - CRL aggregation from mesh-reported fraud.
  - Audit logging for CBI oversight.
- **Mobile Apps**: Any compliant Sekke client — binds to user's home issuer for sync, but verifies foreign coins offline using cached registry keys.

**Why split this way**: routine chain validation happens far more often than settlement and is latency-sensitive (it blocks a user's sync response), so it stays direct/bilateral. Settlement and pool accounting are inherently multilateral and benefit from a single authoritative netting point, so they sit at the hub.

**Communication**:
- Mesh (issuer-to-issuer): HTTPS/TLS 1.3 + mTLS.
- Hub (issuer-to-SCH): HTTPS/TLS 1.3 + mTLS + JWT.
- Messages: JSON over REST or gRPC (for efficiency).

## 3. Core Data Structures

### 3.1 Token (Enhanced, v0.2)
```json
{
  "serial": "string (unique UUID or issuer-prefixed)",
  "issuer_id": "string (e.g., 'bank-mellat-001')",   // origin/minting issuer
  "value": 200000,
  "issuance_date": "ISO8601",
  "ownership_chain": [
    {
      "hop_index": 0,
      "holder_issuer_id": "bank-mellat-001",
      "wallet_ref_hash": "opaque hash, privacy-preserving",
      "timestamp": "ISO8601",
      "proof_hash": "hash of transfer proof"
    },
    {
      "hop_index": 1,
      "holder_issuer_id": "fintech-parsian-002",
      "wallet_ref_hash": "opaque hash",
      "timestamp": "ISO8601",
      "proof_hash": "hash of transfer proof"
    }
  ],
  "signature": "cryptographic blob (verifiable by issuer pubkey)",
  "status": "active | frozen | redeemed | pending_cross_issuer"
}
```

**Changes from v0.1**: `ownership_chain` was a flat array of hashes; it's now a structured, per-hop list tagging each holder's **home issuer**. This is required so that (a) a home issuer knows exactly which foreign issuers to query during mesh validation, and (b) the SCH can attribute cost fairly when a cross-issuer chain orphans (§4.6). Added `pending_cross_issuer` status to distinguish "provisionally accepted, awaiting a foreign issuer's confirmation" from ordinary same-issuer pending state — this distinction matters for reconciliation-engine logic and pool exposure reporting.

*This change should be mirrored in `data_model.md`'s Token entity — not yet done, flagging for a follow-up pass.*

### 3.2 Transaction Proof
- Signed proof of transfer (payer → receiver).
- Includes previous owner proof for chain validation.

## 4. Protocol Flows

### 4.1 Coin Issuance & Redemption
- Only home (origin) issuer creates new coins (backed by reserves).
- Redemption: user requests cash-out → home issuer settles (via SIIP if the chain involved other issuers).

### 4.2 Offline Transfer (Unchanged for Users)
- P2P via QR/NFC/BLE works across issuers (app verifies foreign issuer pubkey via cached SCH registry).

### 4.3 Sync & Cross-Issuer Chain Validation (Mesh — updated)
1. User syncs with **home issuer**.
2. Home issuer processes the chain:
   - Same-issuer hops: reconciled locally, as before.
   - Foreign-issuer hops (identified via `holder_issuer_id` on each hop): home issuer calls that issuer's `/validate_chain` **directly** (mesh, bilateral) — the SCH is not involved in this call.
3. The foreign issuer checks the segment against its own customers' sync records and responds with validation status + risk score + any known revocations for that segment.
4. Home issuer combines local + mesh results to finalize status (confirmed / provisionally accepted / frozen), same grace-period logic as the single-issuer case (48–72h).

### 4.4 Multi-Issuer Settlement (Hub Netting — updated, replaces "Batch Settlement (Daily/Periodic)")
1. Settlement runs in **multiple windows per day** rather than once daily.
   *Proposed default (needs sign-off)*: 4 windows/day, aligned to Sekke's existing operational rhythm — approx. 04:00 (after the existing auto-cash-out run), 10:00, 16:00, 22:00 Iran time.
2. At each window, every issuer submits to the SCH (`POST /submit_batch`) its confirmed cross-issuer positions accumulated since the last window (e.g., value of foreign-issued tokens its own users cashed out or absorbed, per counterpart issuer).
3. SCH computes **multilateral net positions** (not raw pairwise sums — reduces the number of actual transfers needed) and returns one net position per issuer (`GET /net_position`).
4. Issuers confirm (`POST /confirm_settlement`) and execute transfers via Shetab/CBI settlement rails.
5. SCH logs the window for CBI audit.

### 4.5 Revocation & Fraud Propagation (updated)
- Double-spend proofs are reported **bilaterally** to the counterpart issuer (`/report_fraud`, mesh) **and** to the SCH for CRL aggregation and pool-exposure tracking.
- SCH publishes a consolidated CRL; issuers/apps poll or receive push deltas.
- Each issuer remains responsible for penalizing its own users (collateral seizure) for confirmed fraud.

### 4.6 Cross-Issuer Orphaned Chain Resolution & Shared Guarantee Pool (new)
This section formalizes how cross-issuer orphan risk (flagged as a system-wide concern in `orphan_chain_risk_mitigation.md`) is handled once a chain spans more than one issuer's customers.

**Why pooled, not origin- or home-issuer-borne**: an orphaned intermediate hop can belong to *any* issuer in the chain, regardless of who minted the token or who's currently holding the frozen downstream tokens. Assigning the loss to the origin issuer or the home issuer alone would punish an issuer for another issuer's customer going dark. Pooling spreads that risk across all issuers who actually had a stake in the chain.

**Flow**:
1. Grace period (48–72h) lapses on a chain segment where the unresponsive intermediate holder belongs to a *different* issuer than the affected downstream holder.
2. The affected holder's **home issuer** freezes only the affected downstream tokens (existing principle, unchanged) and reports the exposure to the SCH: `POST /guarantee_pool/report_exposure` — token serial(s), value, the full list of participating issuers (read directly off the token's hop-tagged `ownership_chain`), and evidence of the unresolved segment.
3. SCH opens a resolution window. *Proposed default (needs sign-off): 14–30 days.* During this window:
   - If the orphaned hop's issuer reports the missing sync eventually arrived and the chain completes → case closes, no payout, normal unfreeze.
   - If unresolved when the window lapses → SCH triggers a **pool payout**.
4. **Pool payout**: SCH pays the affected holder's home issuer from the shared pool, sized to make the affected end user whole (preserving the existing "no loss for honest users" guarantee). SCH then debits the pool contributions of the chain's participating issuers.
5. **Cost-split formula**. *Proposed default (needs CBI/consortium sign-off, not decided unilaterally here)*: pro-rata by each participating issuer's share of the token's value that passed through their own customers within that specific broken chain; falls back to an equal split across participating issuers if attribution is ambiguous (e.g., a hop's proof is itself part of the dispute). This ties cost to actual participation rather than overall market share, which seems fairer, but is ultimately a policy call.
6. **Pool funding**: each issuer posts collateral to the SCH-administered pool, separate from (and in addition to) its own single-issuer guarantee fund, which continues to cover ordinary single-issuer orphan chains exactly as before. *Sizing/top-up mechanics — proposed proportional to each issuer's share of cross-issuer transaction volume — open for confirmation.*
7. All exposure reports, resolutions, and payouts are logged to the CBI audit trail.

## 5. API Endpoints

### 5.1 Mesh Endpoints (issuer-to-issuer, bilateral)
**Base**: `https://issuer.example.com/siip/v1/`
- `POST /validate_chain` — Submit an ownership-chain segment; returns validation status + risk score.
- `POST /report_fraud` — Submit double-spend proof (bilateral copy; also sent to SCH — see §4.5).
- `POST /query_token_status` — For specific serials.

### 5.2 Hub Endpoints (issuer-to-SCH)
**Base**: `https://hub.sekke.ir/v1/`
- `GET /registry` — Fetch public issuer directory (cached locally by issuers and apps).
- `POST /submit_batch` — Submit settlement-window batch of cross-issuer positions.
- `GET /net_position` — Retrieve computed net position for the current window.
- `POST /confirm_settlement` — Confirm execution of a settled net position.
- `GET /crl` — Current aggregated revocation list.
- `POST /guarantee_pool/report_exposure` — Report a confirmed cross-issuer orphan exposure.
- `GET /guarantee_pool/status` — View pool balance / an issuer's contribution and exposure history.

**Authentication**: mTLS + JWT (issuer-scoped) for both mesh and hub traffic.

## 6. Security & Trust Model

- **Cryptography**: ECDSA/Ed25519 (MVP) → BLS/ZKP.
- **Issuer Onboarding**: CBI approval + public key registration with the SCH.
- **Economic Security — two layers**:
  1. Each issuer's own collateral/reserves, proportional to its issuance (covers single-issuer orphan chains, unchanged from v0.1).
  2. The **Shared Guarantee Pool** at the SCH (new), covering cross-issuer orphan exposure per §4.6.
- **Threat Mitigation**:
  - Rogue issuer: revocation by registry + CBI intervention.
  - Chain attacks: grace periods + partial validation (unchanged).
  - Cross-issuer free-riding (an issuer's customers disproportionately causing orphaned hops at other issuers' expense): mitigated by tying the pool cost-split to actual participation (§4.6.5) rather than a flat/equal split, plus SCH monitoring dashboards analogous to the single-issuer case.
  - Privacy: minimize data shared (only necessary hashes/proofs, never raw PII, across mesh or hub).

## 7. Compliance & Operations

- All mesh and hub messages logged for audit (CBI access).
- Data residency: all inter-issuer and hub traffic stays within Iran.
- Tiered KYC respected across operators.
- Dispute resolution: escalate to CBI — including disputes over pool cost-split attribution.

## 8. Implementation Roadmap

**Phase 1 (MVP Extension)**: Single issuer with SIIP stubs — token schema (hop-structured `ownership_chain`) and endpoint scaffolding in place, but no live mesh traffic, no live SCH.
**Phase 2**: Multi-issuer support in reference backend + protocol tests — mesh validation goes live between at least two issuers; SCH registry and settlement go live; guarantee pool established with initial collateral.
**Phase 3**: Production with banks, shared hub at full scale, pool formula and sizing finalized by CBI/consortium based on Phase 2 data.

## 9. Open Questions

**Resolved this revision**: mesh vs. hub topology (hybrid); who bears cross-issuer orphan risk (shared pool); settlement frequency (multiple/day).

**Still open**:
- Exact settlement window clock times (proposed: 04:00/10:00/16:00/22:00 Iran time).
- Exact guarantee-pool cost-split formula and collateral sizing/top-up mechanics.
- Exact resolution-window length before a pool payout triggers (proposed: 14–30 days).
- Fee structure for mesh `/validate_chain` queries (free/reciprocal vs. metered) — carried over from v0.1.
- Mobile client discovery/caching strategy for foreign issuer keys (bundled snapshot vs. on-demand fetch) — carried over from v0.1.
- Whether `/report_fraud` to the SCH should be real-time or batched with settlement windows.
