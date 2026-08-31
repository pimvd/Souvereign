# Souvereign

Landing zone and platform architecture for European sovereign cloud providers. Each provider has its own folder; the designs are written to be read on their own.

| Provider | Document | Status |
|---|---|---|
| Scaleway | [Landing zone design](scaleway/landing-zone.md) | Draft v0.5 |
| Scaleway | [Implementation plan](scaleway/implementation-plan.md) | Companion build guide |
| STACKIT | [Landing zone design](stackit/landing-zone.md) | Draft v0.1 — outline only |

---

## Author and contact

| | |
|---|---|
| **Name** | Pim van Dijk |
| **Location** | Rotterdam Area, Netherlands |
| **Available via** | [Team Rockstars IT](https://www.teamrockstars.nl/) |
| **Email** | [pim@vandijkcloud.nl](mailto:pim@vandijkcloud.nl) · [pim.vandijk@teamrockstars.nl](mailto:pim.vandijk@teamrockstars.nl) |

Questions about these designs, or interested in a landing zone like this for your own estate? Feel free to reach out via either address above.

## Certifications

| Certification | Provider | Issued | Reference | Certificate |
|---|---|---|---|---|
| Scaleway Foundations | Scaleway | 15 April 2026 | `7646468504112674` | [PDF](certifications/scaleway-foundations.pdf) · [verify](https://scaleway.360learning.com/redirect/api/certification/7646468504112674/authed/html) |
| Scaleway Associate: Network | Scaleway | 1 August 2026 | `8580743074158647` | [PDF](certifications/scaleway-associate-network.pdf) · [verify](https://scaleway.360learning.com/redirect/api/certification/8580743074158647/authed/html) |
| Scaleway Associate: Security & Identity | Scaleway | 1 August 2026 | `5697254866632394` | [PDF](certifications/scaleway-associate-security-identity.pdf) · [verify](https://scaleway.360learning.com/redirect/api/certification/5697254866632394/authed/html) |
| Certified STACKIT Cloud Engineer Fundamentals | STACKIT | 29 May 2026, Neckarsulm | — | [PDF](certifications/stackit-cloud-engineer-fundamentals.pdf) |

> **Note:** the **verify** links go to Scaleway's learning platform (360Learning) and require you to be **signed in** — they will redirect to a login page otherwise. The **PDF** copies in this repository open without an account. Authenticity of a Scaleway certificate can also be confirmed with Scaleway directly at [contact.certifications@scaleway.com](mailto:contact.certifications@scaleway.com); each is valid for 24 months from its issue date. The STACKIT certificate carries no reference number or online verification URL — the PDF is the certificate as issued.

## Repository layout

```
README.md                       this file — index, contact, certifications
scaleway/
  landing-zone.md               hub & spoke design
  implementation-plan.md        phased build guide for the design
stackit/
  landing-zone.md               STACKIT counterpart (outline)
certifications/                 certificate PDFs, both providers
```
