# Sekke — Pre-Seed Risk Analysis Report

**Date**: July 2026 

**Version**: 0.2 (Updated — incorporates concrete mitigation architecture for the top 5 risks and a physical-cash counterfeiting benchmark) 

**Scope**: Cross-cutting risk assessment covering financial/fraud, technical, operational, regulatory, security, privacy, market, and execution risk. 

**Audience**: Founders, early investors, technical leadership.

---

## 1. Executive Summary

Sekke's core value proposition — offline, cash-like digital coins with issuer-backed guarantees — is also its core risk source. Any system that lets value move peer-to-peer without instant settlement inherits the double-spend problem that centralized ledgers solve trivially. This version of the report updates the original assessment with a concrete, mostly-software mitigation architecture for the top 5 identified risks, designed under two constraints: (a) the app must run acceptably on low-end devices (Android 9+, no assumed hardware security module), and (b) the network can be assumed **always reachable, but slow** (server responses may take 5–240 seconds) rather than genuinely offline for extended periods.

That second assumption changes the risk picture more than any single control does. The original design's 48–72 hour grace period was a hedge against days-long disconnection. If disconnection is now assumed rare and the real constraint is *latency*, the fraud-detection window shrinks from days to roughly the length of one slow server round-trip — a reduction of about three orders of magnitude — before any additional control is even applied.

**Top 5 risks, now paired with concrete mitigations:**

1. **Fan-out double-spending** — mitigated via an in-flight value budget, short checkpoint cadence, per-token hop-count tagging with mandatory "hot coin" revalidation, and cryptographic equivocation proof. Together these convert an unknown behavioral parameter into an architecturally enforced ceiling.
2. **Single signing-key compromise** — mitigated via split-key issuance, threshold signing, an issuance-rate circuit breaker, and a continuous supply-reconciliation invariant, none of which require hardware beyond what the issuer's own servers provide.
3. **Slow/high-latency network handling** (reframed from "connectivity shutdown," per the updated assumption) — mitigated via an async, non-blocking client architecture with idempotent retries and a conservative timeout fallback, with the old grace-period design retained as a backstop only for genuine outages.
4. **Regulatory positioning vs. a future CBDC** — largely a business/policy question, not a software one; the one available technical hedge is designing the token format for future interoperability.
5. **Manual ops fraud/error** — mitigated via multisig-style dual-control approval, an append-only hash-chained audit log, and automated anomaly flags.

Section 3 now presents both a **baseline (pre-mitigation)** and an **enhanced (post-mitigation)** quantitative model side by side, so the effect of these controls is visible as a number, not just an assertion. Section 13 adds a benchmark against real-world physical-cash counterfeiting data, which is useful context — but only after the enhanced controls are in place; the unmitigated baseline compares unfavorably to physical cash and that should be stated plainly, not glossed over.

---

## 2. Methodology & Updated Operating Assumptions

Risks are assessed on Likelihood (Low/Medium/High) and Impact (Low/Medium/High/Critical), against MVP scope unless noted otherwise.

**Updated assumptions used throughout this version:**
- **Network**: always eventually reachable; server response times of 5–240 seconds should be expected and designed around, rather than treated as an error condition. Genuine multi-day disconnection is now treated as a rare fallback case, not the primary design point.
- **Device floor**: Android 9+, low-end ARM hardware, no guaranteed hardware-backed keystore or secure element. All client-side cryptography must run comfortably on this floor — this rules out pairing-based schemes (e.g., BLS aggregation, ZK-proofs) for the MVP, in favor of standard EdDSA/ECDSA operations, which run in low single-digit milliseconds even on old hardware.
- **Attacker fraction**: 0.1%–0.5% of the user base attempts tampering (as supplied for this analysis), used in Section 3.
- Where a parameter is a genuine engineering choice rather than a known fact (e.g., the hop-count threshold), it is presented as a tunable default with its trade-off stated explicitly, not as a fixed conclusion.

---

## 3. Financial & Double-Spend Risk (Quantitative)

### 3.1 What double-spend actually costs Sekke

Because tokens are bearer instruments during the offline phase, a malicious payer can hand out divergent, individually valid-looking signed transfer proofs of the same coin to multiple receivers before any of them can check with the backend. Per the stated design principle — no asset loss from sync delays or honest receives — **the issuer, not the defrauded receivers, absorbs the loss.** Total token supply on the ledger isn't inflated by the fraud itself, but the issuer must mint offsetting value to make every honest downstream holder whole. Each successful double-spend is therefore a direct liability against the guarantee fund/collateral reserve, and sizing that fund correctly is a financial planning problem as much as a technical one.

### 3.2 Baseline reference model (pre-mitigation)

This is the original model, kept here as a reference point to measure the value of the mitigations in 3.3–3.4 against. It assumes the original 48–72 hour grace period is the effective fraud-detection window, and treats fan-out (*k*, successful fraudulent transfers per attacker per cycle) as an unknown behavioral parameter, sensitivity-tested at Low/Medium/High = 3/8/20.

**Malicious actors, M = N × f:**

| Phase | Users (N) | M, f=0.1% | M, f=0.5% |
|---|---|---|---|
| Pilot | 500 | ~1 | ~2.5 |
| Early growth | 50,000 | 50 | 250 |
| National | 5,000,000 | 5,000 | 25,000 |

**Naive annualized exposure (USD-equivalent, ~120 grace-period cycles/year, no suppression of repeat offenders), national scale:**

| f | k=Low (3) | k=Medium (8) | k=High (20) |
|---|---|---|---|
| 0.1% | $900,000 | $2,400,000 | $6,000,000 |
| 0.5% | $4,500,000 | $12,000,000 | $30,000,000 |

With an assumed 90% suppression of repeat offenses after first detection, these figures drop by 10x — still $450,000–$3,000,000/year at national scale. This was the starting point; Section 3.4 shows what the additional controls below do to it.

### 3.3 Enhanced protocol design

Four complementary, mostly-client-side mechanisms replace the long grace period with a short, architecturally-bounded one.

#### 3.3.1 In-flight value budget & checkpoint cadence

Each wallet has a cap on total **unconfirmed** outstanding value at any moment (default: **5 coins**). A checkpoint — a background, non-blocking call to the server — is triggered whichever comes first: (a) 5 coins of cumulative unconfirmed activity, or (b) a fixed cooldown elapses (default: **30 seconds**) since the last checkpoint. Because the network is assumed always reachable, this call resolves within the stated 5–240 second range rather than being deferred for days. This is the mechanism behind your original two examples, formalized as two triggers instead of one, so neither a value-based nor a time-based attack path is left open on its own.

#### 3.3.2 Hop-count tagging & "hot coin" flagging

Each token carries its own transfer history: a hop counter plus one signature per hop, `(serial, hop_index, receiver_id_hash, hash(previous_signature))`, signed by the transferring holder. Crossing a threshold **H\*** (default: **5**, matching the existing chain-depth guidance) flags the token "hot": it cannot be spent again until an online validation confirms it. This bounds how far *any single* forked branch of a fraud can travel before being forced to check in — independent of, and complementary to, the value-based budget in 3.3.1, which bounds how *much* can be outstanding at once.

Use a full Ed25519 signature (64 bytes) per hop rather than a further-shortened tag — truncating a signature below its native length breaks verifiability, and at 64 bytes per hop the total payload (5 hops ≈ 320 bytes) is trivial for a QR code well within old-camera reliable scan limits. A shortened tag would also weaken collision resistance unacceptably (a 64-bit tag gives only ~2^32 effective resistance); there's no real pressure to make this trade-off given how small the full signature already is.

Combined, these two controls give a clean bound: **worst-case fraud radius per malicious actor ≤ (in-flight budget) × (hop cap)** — a designed architectural constant, not an estimated behavioral one.

#### 3.3.3 Cryptographic equivocation proof

Because every hop is signed with a strictly increasing sequence number, two divergent signed messages for the same `(serial, sequence_number)` from the same signing key are mathematically irrefutable proof of double-spending — no investigation or judgment call required, and provable to a third party (a regulator, another issuer under SIIP) without ambiguity. This is the same principle used for equivocation-slashing in Byzantine-fault-tolerant systems, and it is cheap: just a counter and a hash comparison, well within Android 9 capability. Detection can happen at *any* point in the chain that receives a slow server response revealing the conflict, not only at the fork's origin — every honest downstream holder is a potential detector.

#### 3.3.4 Local revocation cache & rollback protection

Old devices can't hold a full revocation list, but they can hold a compact probabilistic filter (Bloom or Cuckoo, a few KB), refreshed at each checkpoint, giving a near-instant local first-line check before accepting a token. This also closes a residual gap in 3.3.2: a dishonest holder could try to re-present an *earlier*, lower-hop-count state of a token they've already spent further downstream (a rollback attack). Caching the highest hop-index seen per serial locally catches this cheaply. As agreed, both of these run as an internal client-side process inside the app — no user-facing complexity, no additional server round-trip beyond the normal checkpoint cadence.

#### 3.3.5 QR payload encoding

The QR code itself is optically readable by any generic scanner — that's inherent to the format and can't be prevented. What can be controlled is whether the *content* is meaningful to a casual reader. Encoding the payload as compact binary (protobuf/CBOR/msgpack) rather than human-readable JSON, optionally with a light app-embedded obfuscation layer, means a generic QR reader shows unintelligible bytes rather than parseable fields. This is worth doing for hygiene, but it should be understood clearly as a **defense-in-depth/obscurity layer against casual tampering by non-specialist actors, not a substitute for the cryptographic signature**, which remains the real integrity and authenticity guarantee. A determined reverse-engineer who extracts the app's embedded key would defeat the obfuscation; they would not defeat the signature.

### 3.4 Enhanced exposure model (post-mitigation)

With checkpoints resolving in seconds to a few minutes rather than days, a malicious payer's fan-out is now bounded not by a grace period but by how many QR handoffs are physically possible before the *first* checkpoint resolves — call this window **W ≈ 240 seconds** (the conservative worst case). Realistic fan-out within W, bounded by physical handoff time (roughly 10–20 seconds per QR exchange in a bazaar setting), gives revised scenarios: **k' = 3 (low) / 6 (medium) / 12 (high)**, tighter than the baseline model's k because the window itself is now three orders of magnitude shorter.

Critically, because equivocation proof is non-repudiable, detection at the first checkpoint typically means **permanent ban**, not merely a delay — so the relevant quantity is closer to a one-time, per-attacker lifetime exposure rather than a recurring annual one.

**Lifetime exposure per malicious actor ≈ k' × v** (v = 0.5 USD-equivalent, 5-coin average transfer):

| Scenario | Exposure per attacker |
|---|---|
| Low (k'=3) | $1.50 |
| Medium (k'=6) | $3.00 |
| High (k'=12) | $6.00 |

**System-wide, first-wave lifetime exposure (i.e., assuming the entire existing malicious population attempts fraud once, at national scale):**

| f | Low | Medium | High |
|---|---|---|---|
| 0.1% (M=5,000) | $7,500 | $15,000 | $30,000 |
| 0.5% (M=25,000) | $37,500 | $75,000 | $150,000 |

Since repeat attempts by caught actors are now suppressed near-completely (not just 90%), **ongoing steady-state exposure** is better modeled as driven by *new* malicious actors entering as the user base grows, rather than the whole installed base retrying every cycle. At an illustrative 20% annual user growth and national scale, this gives on the order of **$15,000–$60,000/year** in steady-state exposure — a figure a guarantee fund can be sized against comfortably.

### 3.5 Baseline vs. enhanced comparison

| Metric | Baseline (pre-mitigation) | Enhanced (post-mitigation) |
|---|---|---|
| Fraud detection window | 48–72 hours | ≤ ~240 seconds |
| Repeat-offense suppression | Assumed 90% | Near-total after first detection (non-repudiable proof) |
| National-scale annual exposure | $450,000–$30,000,000 | ~$15,000–$150,000 (first-wave/steady-state) |
| Order-of-magnitude improvement | — | ~20x–300x depending on scenario |

### 3.6 Guarantee-fund sizing implications

The enhanced model supports sizing the fund against tens of thousands of dollars annually at national scale rather than millions — a materially easier number to collateralize and, if ever required, to reinsure or disclose to regulators. Recommend re-deriving this figure from real pilot data once the mechanisms in 3.3 are instrumented (see Section 15), since k' remains an assumption until measured.

### 3.7 Residual risk: collusive rings

The bound above assumes at least one honest party in a forked chain, which forces a checkpoint regardless of the attacker's cooperation (since checkpoint triggers live on the *receiver's* side). A fully collusive ring — a group of Sybil-controlled wallets that never independently checkpoint — could in principle delay detection longer. This is a harder, more economic/behavioral problem than a purely cryptographic one, and is best handled by the reputation-scoring and anomaly-detection layer described in Section 5, rather than by the equivocation-proof mechanism alone. Flagging this honestly rather than claiming the enhanced model closes every path.

---

## 4. Technical & Cryptographic Risk

**Issuer key (server-side — no device constraint applies):**
- **Split-key issuance**: require two independently held keys to sign off on a new token — an issuance key and a separate authorization key that only signs after verifying an approved cash-in record. Compromising one key alone can't mint anything.
- **Threshold signing** (e.g., Shamir/threshold-ECDSA, t-of-n): no single machine ever holds the complete issuer key. Pure software, no hardware dependency.
- **Issuance-rate circuit breaker**: hard-cap tokens mintable per unit time regardless of requester; a stolen key still can't outrun legitimate cash-in volume without tripping an automatic freeze.
- **Continuous reconciliation invariant**: automatically verify `sum(active token value) == sum(approved cash-in value)` on every batch; drift is mathematically detectable proof of unauthorized issuance, not something waiting on a human audit.

**User wallet key (client-side — must run on Android 9+):**
- Ed25519 or secp256r1 signing, a few milliseconds even on old ARM hardware, supported in every mainstream mobile crypto library.
- Use Android Keystore hardware backing opportunistically where available, falling back to an app-level encrypted store (e.g., SQLCipher/Tink) on devices without it — never make hardware backing a hard requirement given the stated low-end-device target.

**Other technical risks carried forward from the original assessment, unchanged:** reconciliation-engine correctness as the single most safety-critical piece of code (a bug here either misses real fraud or incorrectly freezes honest users); the Postgres-to-distributed-database migration path remaining unproven at scale; the admin panel's security needing to match its privileges, not its UI simplicity.

---

## 5. Operational Risk

- **Multisig-style dual control**: no single admin action (cash-in credit, token issuance, wallet unfreeze) executes without a second independent approval — software-enforced segregation of duties instead of policy-only.
- **Append-only hash-chained audit log**: each admin action's log entry includes the hash of the previous entry, making tampering with history detectable rather than merely against policy.
- **Automated anomaly flags**: statistical rules (unusual cash-in size, approval velocity per admin, round-number patterns) hold a transaction for secondary review rather than relying on a human noticing.
- Carried forward unchanged: sync-deadline mismatch for high-volume users (taxi/merchant routines), key-person dependency in a small early team, and the general observation that every manual control that works at 500 users needs a redesigned, less human-dependent version before "early growth" scale.

---

## 6. Regulatory & Compliance Risk

Unchanged from the original assessment in substance — this domain is primarily a policy/business risk, not a software one. The one available technical hedge: **design the token/protocol format for interoperability with a future CBDC standard** (the SIIP layer is already positioned for this), which doesn't remove the competitive risk but reduces the cost of pivoting toward complementary infrastructure if that becomes the path of least resistance. AML/CFT exposure of anonymous wallets, data-residency-vs-supply-chain tension, and hardware/HSM export-restriction risk all carry forward as previously assessed.

---

## 7. Security Risk (Beyond Double-Spend)

Carried forward largely unchanged: insider threat around token-issuance/freeze privileges, device-level malware on a majority-Android, side-loading-prevalent user base, and third-party push-notification dependency. One addition from this update: the QR payload encoding practice in 3.3.5 (compact binary rather than plaintext JSON) applies here too, as a general hygiene measure against casual payload inspection or tampering, independent of its double-spend-specific role.

---

## 8. Privacy Risk

Unchanged from the original assessment: tension between the anonymous-wallet promise and AML needs, de-anonymization risk via transaction-pattern analysis even without a formal identity field, and the concentration of sensitive KYC and transaction data at a single issuer backend as a high-value breach target.

---

## 9. Market & Adoption Risk

Unchanged: IRR volatility undermining a fixed coin denomination, behavior-change adoption risk among cash-native target personas, trust-building risk in a market with historical banking instability, and narrowing differentiation versus existing Shetab-based apps wherever connectivity is reasonably reliable.

---

## 10. Inter-Issuer Protocol (SIIP) Risk

Unchanged: rogue/under-collateralized issuer risk, batch-netting settlement risk, cross-issuer fraud-propagation latency, and governance/dispute-resolution ambiguity. Worth noting explicitly: the equivocation-proof mechanism in 3.3.3 generalizes cleanly across issuer boundaries — a divergent signed claim on the same serial is just as provable whether the two conflicting branches sit under one issuer or two — which is a useful property to carry into the SIIP specification once multi-issuer support is built.

---

## 11. Infrastructure & Connectivity Risk

**Materially reduced under the updated network assumption.** The original top concern here — that connectivity shutdowns extend offline float exactly when fraud detection depends on sync — is largely addressed by treating the network as always-reachable-but-slow rather than genuinely absent for days. That said, this is an assumption, not a guarantee: recommend retaining the original 48–72 hour grace-period logic as a **fallback mode** for the rare case of genuine multi-day outages, so the system degrades gracefully rather than failing outright if the "always reachable" assumption is ever wrong in practice. Shetab-rail dependency for cash-in/out and hosting concentration in a small set of Iranian data centers carry forward unchanged.

---

## 12. Team & Execution Risk

Unchanged: scope ambition versus pre-seed resourcing, regulatory-relationship dependency, and the manual-process ceiling that needs a redesign path before scale, as already covered in Sections 5 and 6.

---

## 13. Benchmark: Physical Cash Counterfeiting Comparison

### 13.1 What the data shows

**Eurozone**: 554,000 counterfeit euro banknotes were withdrawn from circulation in 2024, at a rate of 18 counterfeits per million genuine notes in circulation; this fell to about 444,000 notes (14 per million) in 2025 — one of the lowest rates since the euro's introduction. The €20 and €50 notes account for over 75% of all counterfeits, consistent with counterfeiters targeting the best effort-to-payoff denominations.

**United States**: estimates vary by methodology. A Federal Reserve study puts the stock of counterfeits in circulation at roughly $15–30 million, about 1 in every 80,000 genuine notes; a separate secondary source cites a rougher 1-in-10,000 figure, with cumulative global seized-and-passed value exceeding $250 million over several years (not an annual figure). These two estimates disagree by 2–4x, which is itself a useful reminder of how much precision to expect from any fraud model, including this one.

**Germany, mid-2025**: the Bundesbank withdrew about 36,600 counterfeit euro banknotes worth just under €2.1 million in the first half of 2025, with counterfeiters increasingly targeting the more commonly used €50 and €100 denominations.

**Iran**: no systematic official annual statistic was found. Public reporting is limited to occasional seizure reports (e.g., a routine seizure of 11 counterfeit 500,000-rial notes during a traffic stop) rather than an aggregate published rate. One structural vulnerability is worth noting honestly: to cope with high inflation without printing new higher-denomination banknotes, Iran introduced quasi-currency "bank checks" in 500,000 and 1 million rial denominations carrying weaker anti-counterfeiting features than ordinary banknotes — suggesting the real background rate is plausibly higher than the Eurozone baseline, though this can't be quantified precisely from available sources.

### 13.2 Translating Sekke's assumptions into the same units

Converting the attacker-fraction assumption into "fraudulent transactions per million," assuming ~500 legitimate transactions/user/year:

**Rate = (f × k × cycles/year ÷ 500) × 1,000,000**

| Model | Rate (ppm) | vs. Eurozone (14–18 ppm) |
|---|---|---|
| Baseline, f=0.1%, k=3 | ~720 ppm | ~40–50x higher |
| Baseline, f=0.5%, k=20 | ~24,000 ppm | ~1,300–1,700x higher |
| Enhanced, f=0.5%, k'=12, one-time-per-attacker basis | order of magnitude lower than baseline; converges toward comparable range once expressed as *realized loss* rather than *attempts* |

### 13.3 Naive vs. mitigated comparison — the honest version

Stated plainly: **an unmitigated Sekke would have a fraud *attempt* rate vastly higher than physical cash**, by two to three orders of magnitude. That comparison should not be softened. What changes the picture is that a passed counterfeit banknote is, by definition, a completed and essentially untraceable loss, whereas a caught Sekke fraud attempt — under the enhanced model in Section 3 — is both bounded in size and cryptographically non-repudiable, enabling permanent banning after a single incident. That combination plausibly brings *realized loss* rates down into a more comparable range to physical cash, but this remains a hypothesis to validate against real pilot data, not a settled conclusion.

### 13.4 Where this comparison does and doesn't justify easing controls

- Physical-cash systems concentrate anti-counterfeiting effort on the denominations actually targeted (€20/€50) and apply little friction to small-value cash. Sekke can mirror this: relax checkpoint/cooldown friction for single-coin, low-value spends, and concentrate the hop-cap/in-flight-budget controls on cumulative wallet exposure and larger transfers, where real loss concentrates.
- This comparison should **not** be used to justify relaxing issuer-key custody (Section 4) or manual-ops segregation of duties (Section 5) — a compromised issuer key has no physical-cash analogue at all, and the comparison doesn't transfer there.

### 13.5 Using this comparison externally vs. internally

For internal risk management and regulator-facing diligence, the full picture in 13.1–13.3 should be presented as-is, including the unfavorable baseline numbers — that is what a fund or a regulator will expect and respect.

For external, pitch-oriented framing, it is fair and defensible to lead with the strongest true points from this analysis without restating the full unmitigated baseline every time: for example, that unlike a passed counterfeit banknote, every detected Sekke fraud attempt comes with cryptographic, non-repudiable proof of who committed it — a property physical cash has never had — and that the enhanced design bounds worst-case exposure per bad actor to a small, fixed ceiling rather than an open-ended one. These are accurate, favorable facts, not spin; the distinction from the internal version is one of emphasis and completeness, not accuracy.

---

## 14. Consolidated Risk Register (Updated)

| # | Risk | Likelihood | Impact | Priority (pre-mitigation) | Residual priority (post-mitigation) |
|---|---|---|---|---|---|
| 1 | Fan-out double-spend | High (at scale) | High | Critical | **Medium** (bounded by budget + hop cap + equivocation proof) |
| 2 | Issuer signing-key compromise | Low | Critical | Critical | **Medium** (split-key + threshold signing + circuit breaker) |
| 3 | Slow/high-latency network handling | Medium | Medium (was High under old "shutdown" framing) | High | **Low–Medium** (async client + fallback grace mode) |
| 4 | Regulatory competition with CBDC plans | Medium | High | High | High (largely unmitigated by software) |
| 5 | Manual ops fraud/error | Medium | Medium–High | High | **Medium** (dual control + hash-chained log + anomaly flags) |
| 6 | Reconciliation-engine bug freezing honest users | Low–Medium | High | High | High (unchanged — needs testing investment, not a listed mitigation here) |
| 7 | Anonymous wallet AML/Sybil structuring | Medium | Medium | Medium | Medium |
| 8 | Device malware stealing local wallet keys | Medium | Medium | Medium | Medium |
| 9 | QR relay/replay attacks | Medium | Medium | Medium | **Low–Medium** (hop-cap + rollback cache + equivocation proof) |
| 10 | De-anonymization via pattern analysis | Medium | Medium | Medium | Medium |
| 11 | IRR volatility undermining fixed denomination | High | Medium | Medium | Medium |
| 12 | Sync-deadline mismatch for high-volume users | High | Medium | Medium | Medium |
| 13 | Third-party push provider dependency | Low–Medium | Low–Medium | Low–Medium | Low–Medium |
| 14 | Database migration incident at scale | Low (MVP)/Medium (scale) | Medium–High | Medium | Medium |
| 15 | SIIP rogue/under-collateralized issuer | Low (pre-multi-issuer) | High | Medium (rising) | Medium (rising) |
| 16 | Cross-issuer fraud-propagation latency | Low (pre-multi-issuer) | Medium | Low (rising) | **Low** (equivocation proof generalizes across issuers) |
| 17 | Hardware/software supply-chain export restrictions | Medium | Medium | Medium | Medium |
| 18 | Key-person dependency in admin/fraud ops | Medium | Medium | Medium | Medium |
| 19 | App-store distribution disruption | Low–Medium | Medium | Low–Medium | Low–Medium |
| 20 | Scope overreach for pre-seed team capacity | Medium | Medium | Medium | Medium |

---

## 15. Recommendations & Priority Roadmap

**Before/during pilot:**
1. Implement the in-flight value budget (5 coins) and dual checkpoint trigger (30 sec / 5 coins) — Section 3.3.1.
2. Implement hop-count tagging with H\*=5 and mandatory hot-coin revalidation — Section 3.3.2.
3. Implement equivocation-proof detection logic on the server, and ensure it produces a storable, presentable non-repudiable proof artifact — Section 3.3.3.
4. Implement the local Bloom/Cuckoo revocation cache and rollback protection inside the app — Section 3.3.4.
5. Switch QR payload encoding to compact binary — Section 3.3.5.
6. Instrument telemetry now (checkpoint trigger frequency, hot-coin flag rate, equivocation-proof incidents) so the enhanced model's k' parameter can be measured directly rather than assumed.
7. Formalize dual-control approval and the hash-chained audit log for admin actions — Section 5.
8. Treat issuer key custody (split-key + threshold signing) as a top-tier investment now, not deferred to post-MVP.

**Before scaling beyond pilot:**
9. Re-size the guarantee fund using real k' data from the pilot rather than the illustrative figures in 3.4.
10. Build the reputation-scoring/anomaly-detection layer needed to address the residual collusive-ring risk in 3.7.
11. Retain the original 48–72h grace-period logic as a coded fallback path, tested explicitly for the case of genuine multi-day outages.

**Before multi-issuer (SIIP) rollout:**
12. Carry the equivocation-proof format into the SIIP specification so cross-issuer fraud claims are provable in the same non-repudiable way.
13. Specify concrete collateral audit/enforcement mechanics before a second issuer joins.

---

## 16. Appendix: Formulas & Parameter Reference

- Malicious actors: **M = N × f**, f ∈ [0.001, 0.005].
- Baseline exposure per grace-period cycle: **E = M × k × v**, k ∈ {3, 8, 20} (unvalidated behavioral parameter), v ≈ 0.5 USD-equivalent (5-coin average).
- Baseline naive annual exposure: **E × 120** (≈3-day cycles, no suppression).
- Enhanced fraud window: **W ≈ 240 sec** (worst-case server response), replacing the 48–72h grace period.
- Enhanced fan-out bound: **k' ∈ {3, 6, 12}**, physically bounded by handoff time within W.
- Enhanced fraud radius bound: **worst case ≤ (in-flight budget) × (hop cap H\*)** — a designed constant, not an estimate.
- Enhanced lifetime exposure per actor: **k' × v** (near one-time, given non-repudiable detection and permanent banning).
- Steady-state annual exposure (enhanced): approximated by **(new users/year) × f × k' × v**, driven by user growth rather than the full installed base.

All figures in this report are planning-grade sensitivity ranges, not forecasts. The highest-value next step remains capturing real pilot data on k' and on detection/ban effectiveness, so every downstream figure here can be replaced with a measured value.
