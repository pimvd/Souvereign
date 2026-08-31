# STACKIT

| | |
|---|---|
| **Status** | Draft v0.1 |
| **Date** | 31 August 2026 |
| **Owner** | Pim van Dijk |
| **Scope** | Placeholder — landing zone and platform architecture for a STACKIT estate |

---

## Author and contact

| | |
|---|---|
| **Name** | Pim van Dijk |
| **Location** | Rotterdam Area, Netherlands |
| **Available via** | [Team Rockstars IT](https://www.teamrockstars.nl/) |
| **Email** | [pim@vandijkcloud.nl](mailto:pim@vandijkcloud.nl) · [pim.vandijk@teamrockstars.nl](mailto:pim.vandijk@teamrockstars.nl) |
| **STACKIT certification** | Certified STACKIT Cloud Engineer Fundamentals (see below) |

Questions about this document, or interested in a landing zone like this for your own estate? Feel free to reach out via either address above.

### STACKIT certifications

| Certification | Issued | Certificate |
|---|---|---|
| Certified STACKIT Cloud Engineer Fundamentals | 29 May 2026, Neckarsulm | [PDF](../certifications/stackit-cloud-engineer-fundamentals.pdf) |

> **Note:** the STACKIT certificate carries no reference number or online verification URL — the PDF above is the certificate as issued. Scaleway certifications are listed separately in [the Scaleway document](../scaleway/landing-zone.md#scaleway-certifications).

---

## 1. Purpose and scope

_To be written._ This document is intended as the STACKIT counterpart to the [Scaleway landing zone design](../scaleway/landing-zone.md): the environment model, address plan, identity and segregation-of-duties model, traffic flows, capacity and availability assumptions, and the delivery (IaC) model for a STACKIT estate.

Nothing beyond the author and certification details above has been written yet; the sections below are the intended outline, not agreed design.

## 2. Intended outline

- Architecture overview — environment stamp and project hierarchy
- Network topology and address plan
- Identity, roles, and segregation of duties
- Ingress, egress, and inter-workload traffic flows
- Capacity planning and availability assumptions
- Observability and security controls
- Delivery model (IaC module layout, pipelines, state)
- Validation and rollback
- Cost model
- Exit strategy
- Architecture decision records
