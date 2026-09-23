[← back to the index](../README.md)

# Field 7 — Security for Backend Engineers

17 nodes. Senior candidates are expected to design securely by default, not to be penetration
testers.

Each node has a **preface**, then **details** with **theory** / **example** / **advanced**, and
interview questions.

## Study files

| Part | Nodes | Covers |
|---|---|---|
| [1 — Threat modelling, identity & access](1-identity-and-access.md) | `S01`–`S05` | Threat modelling and STRIDE, authentication and password storage, sessions vs JWT, OAuth2/OIDC, authorisation models |
| [2 — The vulnerability catalogue](2-vulnerabilities.md) | `S06`–`S09` | OWASP Top 10, injection, XSS/CSRF/SSRF/deserialisation, broken access control in practice |
| [3 — Secrets, crypto, privacy & operations](3-crypto-privacy-and-operations.md) | `S10`–`S17` | Secrets and rotation, applied cryptography, privacy and GDPR, abuse prevention, supply chain, infrastructure identity, detection and incident response, secure SDLC |
| [4 — Real-life production problems](4-real-life.md) | Pre-knowledge + 200 questions | Production war stories across `S01`–`S17`: JWT rotation and revocation, refresh-token reuse, SSO edge cases, IDOR at scale, SSRF and IMDSv2, desync and cache deception, credential stuffing, secret leaks and dual-key rotation, KMS limits and crypto-shredding, supply chain, workload identity, incident forensics |

## Node map

| ID | Node | Level | Requires | Unlocks |
|---|---|---|---|---|
| `S01` | Threat modelling | Intermediate | — | S05, S06, S16 |
| `S02` | Authentication | Intermediate | S01 | S03, S04, S13, F11 |
| `S03` | Sessions vs tokens | Advanced | S02 | S04, A18, S13 |
| `S04` | OAuth2 and OIDC | Advanced | S03 | A18, S05 |
| `S05` | Authorisation models | Advanced | S01, S04, M29 | S09, SD13 |
| `S06` | OWASP Top 10 | Intermediate | S01 | S07, S08, S09, F26 |
| `S07` | Injection | Advanced | S06 | DB37, S14 |
| `S08` | XSS, CSRF, SSRF | Advanced | S06, A19 | F26 |
| `S09` | Broken access control | Advanced | S05, S06, F06 | S16 |
| `S10` | Transport, secrets, keys | Advanced | S01, A11 | S11, S15, O19 |
| `S11` | Applied cryptography | Advanced | S10 | S12 |
| `S12` | Privacy, PII, compliance | Intermediate | S11, DB37 | S16 |
| `S13` | Abuse and anti-automation | Intermediate | S02, S03, A08, M06 | SD11 |
| `S14` | Supply chain | Intermediate | S07 | S15, O08 |
| `S15` | Infrastructure and workload identity | Advanced | S10, S14, M26 | O11, SD13 |
| `S16` | Detection, audit, incident response | Intermediate | S01, S09, S12 | O16, O17 |
| `S17` | Security in the SDLC | Intermediate | S06, S14 | O08 |

## Topological order (study waves)

```
Wave 0  S01
Wave 1  S02  S06
Wave 2  S03  S07  S08  S10  S14
Wave 3  S04  S11  S15  S17
Wave 4  S05  S12  S13
Wave 5  S09
Wave 6  S16
```

Cross-field parents: `A08` rate limiting, `A11` TLS, `A18`/`A19` API auth and CORS, `DB37` database
security, `F06`/`F11`/`F26` framework, `M06` gateway, `M26` mesh, `M29` multi-tenancy.

**Near-certain questions:** JWT versus session and revocation (`S03`); how you prevent IDOR
structurally (`S05`, `S09`); password storage (`S02`); SSRF on a URL-fetching feature (`S08`); and
"what do you look for in a security code review" (`S17`).
