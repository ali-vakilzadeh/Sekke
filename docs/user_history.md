# User History - Sekke MVP

This document outlines the typical journey of two user types: **Customer (Payer)** and **Seller/Merchant (Receiver)**.

---

## Customer (Payer) Journey

1. **Registration**  
   User downloads the app, registers with phone number, completes manual KYC (National ID). Wallet is activated with initial balance limit.

2. **Cash-in**  
   User transfers money to issuer’s bank account (off-app). Admin approves → coins are issued and appear in the user’s wallet.

3. **Daily Use - Making a Payment**  
   - Opens app (offline).  
   - Selects amount (number of coins).  
   - Generates QR code containing signed proof of ownership.  
   - Shows QR to seller.  
   - After successful scan by seller, coin is marked as spent locally.

4. **Receiving Coins**  
   - Scans seller’s QR or receives via other means.  
   - App verifies the coin.  
   - Coin is added to local wallet.

5. **Sync (When Online)**  
   - App automatically or manually syncs.  
   - Uploads transaction history.  
   - Receives updated coin status, refreshed coins, and any overflow handling.  
   - Frozen coins (if any) are unfrozen after validation.

6. **Cash-out**  
   - User requests cash-out from online account balance.  
   - Funds transferred to linked Shetab bank card (online).

7. **End of Day**  
   - Ensures sync before major offline periods to avoid freezes.

---

## Seller/Merchant (Receiver) Journey

1. **Registration**  
   Same as customer, often with business verification emphasis.

2. **Cash-in (Initial Float)**  
   Loads wallet with coins for change or operations.

3. **Receiving Payment**  
   - Opens app in receive mode.  
   - Generates QR code or scans customer’s QR.  
   - App verifies incoming coin(s).  
   - Accepts even if approaching/exceeding local limit (flags excess for overflow).

4. **Overflow Management**  
   - If local limit exceeded: Excess marked pending.  
   - App notifies user of 12-hour sync deadline.

5. **Making Payments / Giving Change**  
   - Uses coins from wallet to pay suppliers or give change (same as customer flow).

6. **Sync (Critical for Merchants)**  
   - Syncs within 12 hours (ideally multiple times daily).  
   - Excess coins automatically moved to online account.  
   - Receives refreshed coins.

7. **Auto Cash-out**  
   - Online account balance is scheduled for automatic transfer to bank card (e.g., 04:00 AM).

8. **End of Day**  
   - Reviews daily transactions and ensures sync to unlock full funds.

---

**Notes**: Both users experience cash-like simplicity for offline moments, with periodic online steps for full functionality and security.
