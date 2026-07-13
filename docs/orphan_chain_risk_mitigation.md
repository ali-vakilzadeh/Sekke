# Orphaned Chain Mitigation Strategies - Sekke

**Date**: July 2026  
**Version**: 0.1

This document outlines operational and technical strategies to manage the risk of orphaned chains in the hybrid reconciliation model.

---

## 1. Risk Overview
In the hybrid approach, tokens can circulate offline across multiple hops before all parties sync. If some users (especially in the middle of a chain) delay syncing for too long, it creates **orphaned / unresolved chain segments**, increasing potential exposure for the issuer guarantee fund.

---

## 2. Core Mitigation Strategies

### Technical Controls
1. **Grace Period with Escalation**
   - Default: 48–72 hours provisional acceptance.
   - After grace period: Automatically freeze affected downstream tokens.
   - Notify all involved parties.

2. **Chain Depth & Velocity Limits**
   - Limit the number of offline hops before forcing a sync (e.g., max 5–7 hops).
   - Implement token "age" tracking — older tokens have lower priority or trigger warnings.

3. **Partial Chain Settlement**
   - Backend can provisionally settle known-good prefixes/suffixes of a chain.
   - Only the unresolved segment remains pending.

4. **Revocation Lists**
   - Push frequent (daily) revocation/pending lists to apps.
   - Receivers can reject high-risk (long-pending) tokens.

### Operational & Incentive Measures
1. **User Incentives for Syncing**
   - Small rewards/fee discounts for users with high sync compliance.
   - Strong push notifications and in-app banners for users with pending chains.
   - Temporary limit reductions for frequent offenders.

2. **Monitoring Dashboard**
   - Real-time view of longest pending chains, high-risk users, and potential exposure.
   - Alerts when total orphaned value exceeds defined thresholds.

3. **Rate Limiting & Circuit Breakers**
   - Limit high-velocity spending (e.g., many transfers in short time) until syncs occur.
   - Geo-fencing or merchant-specific rules in high-risk areas.

4. **Issuer Intervention**
   - Manual force-resolution or token replacement for exceptional cases.
   - Collaboration with regulators/authorities for persistently offline users.

5. **Educational Campaigns**
   - Onboarding tutorials emphasizing the importance of regular syncing.
   - Targeted messages for merchants and high-volume users.

---

## 3. Risk Thresholds (MVP)
- Monitor total value in orphaned chains daily.
- Trigger review if > 5% of circulating tokens are in unresolved state.
- Hard cap on guarantee fund exposure per chain.

---

## 4. Future Enhancements (Post-MVP)
- Advanced cryptography for better offline proofs (reducing reliance on syncs).
- Reputation scoring for wallets based on sync history.
- Integration with mobile operator data for better connectivity predictions.

