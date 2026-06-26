```
 ██╗      █████╗ ███╗   ██╗     █████╗ ████████╗██╗      █████╗ ███████╗
 ██║     ██╔══██╗████╗  ██║    ██╔══██╗╚══██╔══╝██║     ██╔══██╗██╔════╝
 ██║     ███████║██╔██╗ ██║    ███████║   ██║   ██║     ███████║███████╗
 ██║     ██╔══██║██║╚██╗██║    ██╔══██║   ██║   ██║     ██╔══██║╚════██║
 ███████╗██║  ██║██║ ╚████║    ██║  ██║   ██║   ███████╗██║  ██║███████║
 ╚══════╝╚═╝  ╚═╝╚═╝  ╚═══╝    ╚═╝  ╚═╝   ╚═╝   ╚══════╝╚═╝  ╚═╝╚══════╝
```

> **Open to contributors.** Whether you're a student, a career-changer, or a seasoned engineer — if you care about network security and clean code, there's a place for you here. See [Contributing](#contributing) to get started.

---

## What Is LAN Atlas?

LAN Atlas is a lightweight, cloud-hosted network visibility SaaS built for solo IT admins and small MSPs who need to know what's on their network — without enterprise-scale complexity or cost.

A small agent runs on-premises, scans local subnets, and securely forwards observations to a centralized cloud service. From there, operators get simple dashboards, actionable alerts, and clean exports that answer three questions:

- **What devices exist?**
- **What changed?**
- **What needs attention?**

The MVP is intentionally minimal and production-minded, focused on proving a complete `agent → cloud → dashboard` workflow end-to-end before adding features.

---

## Architecture Overview

```
[ On-Prem Agent ]
  ARP/ping sweep + port scan
  Signed HMAC-SHA256 payloads
  Buffering + retry on disconnect
        │
        │  HTTPS (mTLS or API key auth)
        ▼
[ Cloud API ]
  Multi-tenant: Orgs → Sites → Agents → Devices
  Observation ingestion + heartbeat tracking
  Alert engine + export layer
        │
        ▼
[ Dashboard ]
  Per-site device inventory
  Change feed and open alerts
  CSV / JSON export
```

---

## Functional Requirements

**On-Prem Agent**
- Subnet scanning via ping sweep, ARP, and limited port probing
- Signed observation payloads and periodic heartbeats sent to the cloud API
- Resilient behavior: local buffering and automatic retry on connection loss

**Cloud Service**
- Multi-tenant data model: Organizations → Sites → Agents → Devices → Observations
- Per-site dashboards and alert feeds
- Data export in CSV and JSON formats

**Alerting**
- New device detected on a monitored subnet
- Known device absent for a configurable number of hours
- New open port observed on an existing device

---

## Non-Functional Requirements

| Requirement | Approach |
|---|---|
| Agent-to-cloud security | HMAC-SHA256 signed payloads; API key auth scoped per agent |
| Multi-tenant isolation | Row-level tenancy enforced at the data layer |
| Agent resilience | Local observation buffer with exponential backoff retry |
| Cloud infrastructure | AWS deployment following Well-Architected Framework principles |
| Codebase quality | Clean, documented Python; reviewed against OWASP ASVS and Top 10 |

---

## Security Posture

LAN Atlas is built with security as a first-class requirement, not an afterthought. Key controls include:

- **OWASP API Security Top 10 (2023)** alignment across all agent-to-cloud endpoints
- **OWASP ASVS** compliance targets for authentication, session, and data validation layers
- **NIST SP 800-53 / 800-61** informed incident response and access control design
- **CIS Control 1** (Inventory and Control of Enterprise Assets) as the product's core use case
- Agent tokens stored as hashed values — never plaintext — consistent with credential management best practices

---

## Contributing

**We actively welcome open-source contributors at every level.**

This project started as a Cloud Security Office Hours collaboration, and that spirit carries forward. If you're learning security engineering, building toward a portfolio, or want to contribute production-grade work to a real system — this is a good place to do it.

Here's how to get involved:

1. Read [`CONTRIBUTING.md`](CONTRIBUTING.md) for branch conventions, commit style, and PR expectations
2. Set up your local environment via [`docs/contributor-setup.md`](docs/contributor-setup.md)
3. Browse open issues — anything tagged `good first issue` or `help wanted` is ready to be picked up
4. Open a draft PR early if you want feedback before finishing

We follow a structured Git workflow (`feature/`, `fix/`, `security/`, `docs/`, `chore/` branch prefixes) and use GitHub Projects to track sprint work. You'll have context from day one.

No contribution is too small. Documentation, tests, schema feedback, and security reviews are as valued as feature code.

---

## Getting Started

```bash
git clone https://github.com/CloudSecurityOfficeHours/LANAtlas.git
cd LANAtlas
```

See [Contributor Setup](docs/contributor-setup.md) for local environment configuration and `.env` instructions.

---

## License

[MIT](LICENSE)

---

*Built openly through Cloud Security Office Hours. Come build with us.*
