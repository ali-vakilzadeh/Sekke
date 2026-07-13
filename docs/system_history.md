# System History - Daily Cycle (Sekke MVP)

This document describes a typical daily operational cycle of the overall Sekke system.

---

## Daily System Cycle

1. **Midnight / Early Morning (Automated)**  
   - Scheduled auto-cash-out processes run for online account balances (e.g., 04:00 AM).  
   - Funds transferred to users’ Shetab cards where configured.  
   - Nightly backups and basic integrity checks on the ledger.

2. **Morning (Issuer Operations)**  
   - Admin team reviews overnight syncs, pending KYC, and cash-in requests.  
   - New tokens issued for approved cash-ins.  
   - Monitoring dashboard checked for any reconciliation issues or freezes.

3. **Throughout the Day (Offline Activity)**  
   - Users perform peer-to-peer transfers via QR codes in offline environments (bazaars, taxis, markets).  
   - Apps locally validate and store transfer records.  
   - Receivers accept coins (with overflow flagging if needed).

4. **Daytime Syncs (Ongoing)**  
   - Users connect to internet (WiFi/mobile data) → apps automatically sync.  
   - Backend receives transaction batches.  
   - Reconciliation engine processes ownership chains.  
   - Overflow handling: excess moved to online accounts.  
   - Tokens refreshed and any freezes resolved.  
   - Updated status pushed back to apps.

5. **Evening / Peak Activity**  
   - Higher volume of syncs as users end daily transactions.  
   - Merchants especially sync to unlock overflow funds.

6. **End of Day Reconciliation**  
   - System aggregates daily activity.  
   - Flags any unresolved chains or suspicious patterns for admin review.  
   - Prepares for next day’s issuance and cash-outs.

7. **Monitoring & Maintenance**  
   - Logs reviewed.  
   - Server health, database performance, and sync success rates monitored.  
   - Minor updates or alerts handled by operations team.

---

**Notes**: The system runs mostly asynchronously. The daily sync cycle ensures security and liquidity while allowing full-day offline operation. Peak load occurs around evening sync windows.
