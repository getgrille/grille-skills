---
name: grille-certs
description: "Use when reading Windows certificate store contents via Grille. WHEN: 'is my TLS cert expiring', 'what certs are in the personal store', 'check certificate expiry', 'find certs expiring soon', 'which root CAs are trusted', 'is there a rogue CA installed', 'cert_store', 'certificate thumbprint', 'certificate SAN', 'what is the issuer of this cert', 'cert expiry', 'certificate store'. DO NOT USE WHEN: 'create or renew a certificate' (not supported — read-only); 'private keys' (never accessible — Windows CNG OS guarantee, not a Grille filter); 'firewall or network diagnostics' (use grille-networking or grille-firewall); 'code signing cert' (use Azure Code Signing tooling directly)."
---

## Overview

This skill covers Grille's Certs module — 1 tool for reading Windows certificate store metadata. Read-only. No elevation required. Private keys are never accessible (Windows CNG architectural guarantee — not a Grille filter).

Requires `"certs"` in the active role's modules list.

**Security note (SD-050):** This tool reads certificate metadata only. SANs may contain internal hostnames — see SD-050 for the full threat model and accepted risk rationale.

## Tools

- `cert_store` — Read certificates from a named Windows certificate store. Returns subject, SANs (DNS), issuer, expiry date, days-until-expiry, thumbprint (SHA-1), and key usage per cert. Sorted soonest-expiry first. Expiry labels: CRITICAL (≤7 days), WARNING (≤30 days).

  Store names:
  - `MY` (default) — Personal store. Developer certs, client auth certs, device identity certs (Apple MDM, Entra/AAD, self-signed).
  - `ROOT` — Trusted Root CAs. Use to check for rogue CA installations (MITM risk).
  - `CA` — Intermediate CAs.
  - `TRUST` — Enterprise trust.
  - `ADDRESSBOOK` — Contact certificates.

  Filters:
  - `filter_subject` — Substring match on subject, SANs, or issuer (case-insensitive).
  - `filter_expiry_days` — Only certs expiring within N days. Use 30 for "expiring this month", 0 for already-expired.
  - `filter_thumbprint` — Exact SHA-1 thumbprint match (hex, spaces ignored).
  - `include_expired` — Include already-expired certs (default false).

## Patterns

**Check all personal certs:**
```
grille:cert_store
```

**Find certs expiring within 90 days:**
```
grille:cert_store filter_expiry_days=90
```

**Check trusted root CAs for Microsoft:**
```
grille:cert_store store="ROOT" filter_subject="microsoft"
```

**Check for any rogue/unexpected root CAs:**
```
grille:cert_store store="ROOT"
```

**Look up a cert by thumbprint:**
```
grille:cert_store filter_thumbprint="3B1EFD3A66EA28B16697394703A72CA340A05BD5"
```

**Find all expired certs:**
```
grille:cert_store include_expired=true filter_expiry_days=0
```

**Check intermediate CAs:**
```
grille:cert_store store="CA"
```

## Output format

```
MY  cert store 'MY' — 3 certificates

  Subject:     CN=localhost
  SANs:        localhost
  Issuer:      CN=localhost
  Expiry:      expires 2027-05-06 (342 days)
  Key usage:   Digital Signature, Key Encipherment
  Thumbprint:  16F0DE327E9F5EDD59B667C55003AE09450CF545

  Subject:     0028864F-4A2C-4C23-997D-E73E4A0838D3
  SANs:        0028864F-4A2C-4C23-997D-E73E4A0838D3
  Issuer:      US, Apple Inc., Apple iPhone Device CA
  Expiry:      CRITICAL — expires in 57 days
  Key usage:   Digital Signature, Key Encipherment
  Thumbprint:  3643ED201D792FA183F2D37ABA27B4B95FFFAD77
```

Expiry urgency labels:
- `CRITICAL — expires in N days` — ≤7 days
- `WARNING — expires in N days` — ≤30 days
- `expires YYYY-MM-DD (N days)` — healthy
- `EXPIRED (N days ago)` — only shown with `include_expired=true`

## Key usage values

`Digital Signature`, `Non-Repudiation`, `Key Encipherment`, `Data Encipherment`, `Key Agreement`, `Certificate Sign`, `CRL Sign`. Root and intermediate CAs typically show `Certificate Sign, CRL Sign`.

## ⛔ Common mistakes

- **Asking for private keys** — never available, period. Windows CNG stores private keys in hardware-backed key containers. No enumeration API exposes them. This is not a Grille restriction.

- **Wrong store for the task** — developer TLS certs are in MY, root CA trust decisions are in ROOT, intermediate chain certs are in CA. Using the wrong store returns empty or irrelevant results.

- **`filter_expiry_days=0` shows empty** — that filter means "expiring within 0 days from now" i.e. already expired today. Combine with `include_expired=true` to see all expired certs.

- **Rogue CA check** — if you suspect a MITM CA was installed, check ROOT store, not MY. Rogue CAs installed by enterprise proxies or malware appear as unexpected entries in ROOT with `Certificate Sign` key usage and a non-standard issuer.
