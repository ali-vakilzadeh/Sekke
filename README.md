# Sekke

**Sekke** is an **offline-first digital coin system** pegged 1:1 to the Iranian Rial (IRR), designed to replace or supplement physical cash in Iran's high-cash, disruption-prone environment (e.g., bazaars, taxis, P2P). 

- **Core Innovation**: Digital bearer coins stored in a mobile app that transfer **completely offline** (NFC on Android via HCE, QR codes, BLE fallback for iOS). No real-time network, bank approval, or SMS needed for transfers.
- **Security Model**: Cryptographic double-spend detection (e.g., Brands-like e-cash or BLS signatures) where copying is possible but fraud is self-revealing upon sync (identity extraction from proof pairs). Issuer guarantee fund protects honest users; no reliance on phone secure elements.
- **Risk Controls**: Strict wallet limits (e.g., 10-15 coins base, higher with deposits), anonymous low-cap wallets, mandatory periodic syncs (daily in MVP, weekly in prod) with token expiry, local revocation lists.
- **Backend**: Iranian-hosted (data residency), handles issuance, reconciliation of offline chains, KYC/AML (tiered Shahkar/biometric), Shetab integration for cash-in/out, fraud dashboards. Uses HSM in production.
- **Tech Stack Outline**: Flutter mobile app; backend in Go or Spring Boot (PostgreSQL + Redis); Iranian infra (no US clouds).
- **Business/Regulatory Fit**: Free core usage for virality. Revenue from float, fees, merchant tools. Aligned with CBI sandbox → E-money license path, AML, local data rules. Targets millions of small cash transactions for inclusion and resilience under sanctions.
- **Roadmap**: MVP (QR, 500 users, simple tokens) → Full (multi-transfer, blinding/ZKP, scale) → National infrastructure.

The pitch narrative provides the compelling story and market/regulatory rationale; the architecture gives a practical, staged technical blueprint. This is a solid foundation for a regulated fintech product solving real Iranian pain points.
