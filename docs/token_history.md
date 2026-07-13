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
   Receiver’s app verifies the proof and accepts the token.  
   Token ownership is updated locally on both devices.

4. **Multiple Offline Hops**  
   Token circulates phone-to-phone (bazaar, taxi, etc.) through successive QR transfers. Each device maintains local record of the transfer.

5. **Periodic Sync**  
   All devices involved sync with the backend.  
   - Transaction proofs are uploaded.  
   - Backend reconciles the full ownership chain.  
   - Validates no double-spend.  
   - Refreshes the token (new signature) and returns it to circulation.  
   - Any freeze (due to missed sync) is lifted upon successful validation.

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
