# Token History - Sekke MVP

This document traces the full lifecycle of a single digital coin from issuance to cash-out and recycling.

---

## Token Lifecycle Sequence

1. **Issuance**  
   User completes cash-in. Backend generates a new signed token (with unique serial) and assigns it to the user’s wallet. Token becomes active.

2. **Distribution to User Wallet**  
   Token appears in the payer’s mobile app after sync.

3. **Offline Transfer (First Hop)**  
   Payer generates a signed proof of ownership and transfers via QR.  
   Receiver’s app verifies the proof and accepts the token immediately into the spendable balance (hybrid model).  
   Token ownership is updated locally on both devices. The coin is marked as "pending server confirmation".

4. **Multiple Offline Hops**  
   Token circulates phone-to-phone (bazaar, taxi, etc.) through successive QR transfers. Each device maintains local record of the transfer. Downstream receivers can spend the token further even while previous hops are pending confirmation.

5. **Periodic Sync**  
   Devices sync with the backend.  
   - Transaction proofs are uploaded.  
   - Backend reconciles the ownership chain (provisional acceptance for partial chains).  
   - Validates no double-spend.  
   - Applies grace period for delayed nodes (e.g. 48-72 hours).  
   - Refreshes the token and updates status.  
   - Tokens may be frozen only for affected downstream users if gaps remain unresolved.

6. **Overflow Handling (if applicable)**  
   If token causes a wallet to exceed limit upon receipt, it is flagged and moved to the user’s online account during sync.

7. **Cash-out / Liquidation**  
   User requests cash-out from online account.  
   Backend marks the token as redeemed.  
   Funds are transferred to the user’s Shetab bank card.

8. **Recycling**  
   Redeemed token is retired from active circulation.  
   Backend can issue a replacement fresh token for new cash-ins, closing the loop.

---

**Notes**: The token remains a bearer instrument during offline phase and becomes fully auditable after each sync. Honest holders never lose value.
