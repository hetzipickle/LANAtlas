
                                ██╗      █████╗ ███╗   ██╗     █████╗ ████████╗██╗      █████╗ ███████╗
                                ██║     ██╔══██╗████╗  ██║    ██╔══██╗╚══██╔══╝██║     ██╔══██╗██╔════╝
                                ██║     ███████║██╔██╗ ██║    ███████║   ██║   ██║     ███████║███████╗
                                ██║     ██╔══██║██║╚██╗██║    ██╔══██║   ██║   ██║     ██╔══██║╚════██║
                                ███████╗██║  ██║██║ ╚████║    ██║  ██║   ██║   ███████╗██║  ██║███████║
                                ╚══════╝╚═╝  ╚═╝╚═╝  ╚═══╝    ╚═╝  ╚═╝   ╚═╝   ╚══════╝╚═╝  ╚═╝╚══════╝


> **Open to contributors.** We're always looking for new contributors and mentors across every role, whether you're technical or non-technical, just getting started or highly experienced. See [Contributing](#contributing) to get started.

# LAN Atlas

LAN Atlas is a lightweight, cloud-hosted network visibility SaaS designed for solo IT admins and small MSPs. On-prem agents scan local networks and securely send observations to a centralized cloud service, which provides dashboards, alerts, and exports that explain what devices exist, what changed, and what needs attention.

The MVP focuses on proving the end-to-end agent → cloud → dashboard workflow while remaining intentionally minimal and production-minded — every schema and architecture decision is made to be extended later without restructuring.

## Architecture Overview


[ On-Prem Agent ]
  ARP/ping sweep + port scan
  HMAC-SHA256 signed payloads (integrity — proves the bytes weren't altered)
  API key auth, hashed at rest (authentication — proves it's this agent)
  Buffering + retry on disconnect
        │
        │  HTTPS (TLS 1.3) — Authorization: Bearer <api_key>
        ▼
[ Cloud API ]
  Multi-tenant: Orgs → Sites → Agents → Devices
  Observation ingestion + heartbeat tracking
  Alert engine + export layer
        ▲
        │  HTTPS (TLS 1.3) — JWT session, HttpOnly + Secure cookie
        │  OAuth 2.0 / OIDC (Google / Microsoft / Okta) — dashboard users only
        │
[ Dashboard ]
  Per-site device inventory
  Change feed and open alerts
  CSV / JSON export


**Two authentication mechanisms exist because two different threat models exist.** An agent is a headless process running unattended on someone's LAN — no human is present to complete an OAuth redirect, so it authenticates with a long-lived API key, hashed at rest, with a rotation path. A dashboard user is a person at a browser — OAuth lets a third party (Google/Microsoft/Okta) own the credential lifecycle so LAN Atlas never has to. (ASVS V10.2 covers the OAuth client side of this; V10.3 covers how the resource server — our API — validates the token it receives.)

HMAC-SHA256 signing on observation payloads is a **separate** control from either of the above — it protects integrity in transit and gives the database a tamper/replay check independent of who sent the request.

> **Implementation status:** the architecture above reflects the finalized design in `LAN_Atlas_OWASP_Controls_v3.docx`. As of this revision, agent API key hashing, session revocation, and OAuth identity anchoring are implemented in schema and partially in code. JWT expiry/refresh rotation, HttpOnly/Secure cookie enforcement, and OAuth provider tenant restrictions are designed but not yet coded — tracked as "Planned" in the controls doc, not "Implemented." Treat this README as the target state, and the controls doc as the source of truth for what's actually shipped.

## Functional Requirements

**On-prem agent:**
- Scans configured subnets (ping sweep, ARP, limited port probing)
- Signs every observation payload with HMAC-SHA256 before transmission — server rejects on hash mismatch
- Authenticates using a per-agent API key; server stores only the SHA-256 hash, never the plaintext
- Buffers observations locally and retries with exponential backoff on connection loss

**Cloud service:**
- Multi-tenant data model: Organizations → Sites → Agents → Devices → Observations
- Dashboard users authenticate via OAuth 2.0 / OIDC (Google, Microsoft, Okta) — **LAN Atlas never receives, transmits, or stores a password**
- Issues its own short-lived JWT after OAuth login, plus a server-side session record (hashed token, expiry, revocation flag) so sessions can be killed independent of token expiry
- Per-site dashboards and alert feeds
- CSV / JSON export

**Alerting:**
- New device detected on a monitored subnet
- Known device absent for a configurable number of hours
- New open port observed on an existing device

## Non-Functional Requirements

| Requirement | Approach |
|---|---|
| Agent-to-cloud integrity | HMAC-SHA256 signed payloads; server-side hash comparison rejects tampered/replayed observations |
| Agent-to-cloud authentication | Per-agent API key, SHA-256 hashed at rest, versioned for rotation without reissuing all agents |
| Dashboard user authentication | OAuth 2.0 / OIDC (Google / Microsoft / Okta); short-lived JWT + server-revocable session; no local password storage |
| Multi-tenant isolation | Row-level tenancy enforced at the data layer; tenant ID sourced only from the verified session, never the request body |
| Agent resilience | Local observation buffer with exponential backoff retry |
| Cloud infrastructure | AWS deployment following Well-Architected Framework principles |
| Codebase quality | Clean, documented Python; reviewed against OWASP ASVS and OWASP Top 10 |

## Security Posture

LAN Atlas is built with security as a first-class requirement, tracked systematically rather than added ad hoc. Frameworks in active use:

- **OWASP Top 10 (2025)** — web dashboard surface
- **OWASP API Security Top 10 (2023)** — agent-to-cloud API surface
- **OWASP ASVS** — authentication, session, and validation control detail, including ASVS V10 (OAuth/OIDC) for the dashboard auth flow
- **NIST SP 800-53 / 800-61** — incident response and access control design
- **CIS Control 1** (Inventory and Control of Enterprise Assets) — the product's core use case
- **MITRE ATT&CK** — threat modeling reference for the agent-to-cloud trust boundary

Key controls:

- **No password storage, by design.** Dashboard authentication delegates entirely to third-party OAuth providers. There is no `password_hash` column and no password-reset flow to secure, because the credential never enters LAN Atlas's trust boundary — this removes a class of risk (credential stuffing, weak-password enforcement, breach-database reuse checks) rather than mitigating it.
- **Agent API keys are stored hashed only** (SHA-256, hex-encoded), never plaintext, with a version counter so a compromised key can be rotated without downtime.
- **Tenant isolation is sourced from the verified session, never the request body** — enforced consistently in API middleware.
- **404-not-403 pattern** on protected resources prevents leaking resource existence to unauthorized callers.

### Additional frameworks under consideration

These sit outside the two Top-10 lists above and are being folded into the v3.0 controls doc as separate tracked gaps rather than new categories:

| Control | Why it matters here |
|---|---|
| ASVS V13.3 — Secret Management | Where OAuth client secrets, JWT signing keys, and DB credentials live in production (target: AWS Secrets Manager / Parameter Store, not `.env`) |
| API Security Top 10 API9:2023 — Improper Inventory Management | Tracking which API versions/endpoints are live vs deprecated as the agent and dashboard APIs evolve |
| CIS Control 8 — Audit Log Management | The industry framing of an existing tracked gap: no structured logging or CloudWatch shipping yet |
| CIS Control 16 — Application Software Security | The industry framing of an existing tracked gap: dependency scanning, signed commits, artifact signing not yet implemented |
| CIS Control 4 — Secure Configuration | Debug mode, directory listing, and API docs endpoints disabled in production before first deploy |

See `LAN_Atlas_OWASP_Controls_v3.docx` for the full gap-by-gap status (Implemented / Planned / Partial / Not Started) across every control above.

## Contributing

We actively welcome open-source contributors at every level.

This project started as a Cloud Security Office Hours collaboration, and that spirit carries forward. If you're learning security engineering, building toward a portfolio, or want to contribute production-grade work to a real system — this is a good place to do it.

Here's how to get involved:
- Read `CONTRIBUTING.md` for branch conventions, commit style, and PR expectations
- Set up your local environment via [Contributor Setup](docs/contributor-setup.md) for local environment and `.env` instructions
- Browse open issues — anything tagged `good first issue` or `help wanted` is ready to be picked up
- Open a draft PR early if you want feedback before finishing

We follow a structured Git workflow (`feature/`, `fix/`, `security/`, `docs/`, `chore/` branch prefixes) and use GitHub Projects to track sprint work. You'll have context from day one.

No contribution is too small. Documentation, tests, schema feedback, and security reviews are as valued as feature code.


*Built openly through Cloud Security Office Hours. Come build with us.*
