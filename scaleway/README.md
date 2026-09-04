# Scaleway Landing Zone — Hub & Spoke Design

| | |
|---|---|
| **Status** | Draft v0.10 |
| **Date** | 4 September 2026 |
| **Owner** | Pim van Dijk |
| **Scope** | Network, identity, delivery, and operations architecture for a multi-environment, multi-workload Scaleway estate |
| **Changes in v0.2** | Architecture diagrams; capacity planning; availability assumptions; observability; security controls; Terraform module layout; validation; ADR appendix; formula-based cost model |
| **Changes in v0.3** | **Fix: spoke CIDR derivation overlapped hub /20 (now `n + 4`, spokes from 10.e.16.0/22)**; hub internal PN layout (§6.3); corrected NVA capacity model (planning formula + validation invariant, 4 dimensions); validation split into pre-/post-apply with rollback classes and Phase 1 LB acceptance gate |
| **Changes in v0.4** | New **Exit strategy** chapter (§16): reversible-lock-in principle (ADR-010), exit register, data-export requirement for stateful services, off-provider rebuild rehearsal and a time-to-exit SLO. Provider-neutral; managed LB pool retained unchanged. Subsequent sections renumbered §17–§20 |
| **Changes in v0.5** | **Delivery model reshaped**: global layer renamed `platform`; **repo + Terraform state per workload** (spoke), networking now lives in the spoke repo under a platform-controlled identity, **two-sided peering as the duty boundary** (§6.6, §14, ADR-011/012). **Environments are instance-numbered stamps** `<class><NN>` and the **250 target is now a global spoke-instance budget**, not per-environment; small stamps → **one /16 per stamp** addressing (§6.1, §6.2, ADR-013). Capacity reworked to single-shard-per-stamp (§8). **Cost model rebuilt with a dated euro snapshot** and the inverted "fixed base × stamp count" driver (§18). Multi-domain DNS made explicit (§11). Single manual bootstrap seam (§14.4, ADR-014). Managed LB retained |
| **Changes in v0.6** | **Addressing and estate model reshaped again**: the estate is now **four named stamps** (`prd01`, `acc01`, `tst01`, `dev01`) plus **three reserve blocks**, each stamp receiving **one /12** with the hub as the first **/21** and spokes as /22 from block 2 up (§6.1, §6.2, §6.3, ADR-015, supersedes ADR-013 addressing). Instance numbering is **retained** so a second instance is an allocation, not a redesign. **Scaleway quotas confirmed against published documentation** and the speculative "~50 spokes" threshold retired: there is **no per-VPC peering cap**; the binding limit is the **Organization-wide 255 Private Network quota**, which caps the estate at ~19 spokes/stamp across four stamps (§6.6, §8, §17, §20). The 250 global spoke-instance budget is replaced by that PN budget as the CI invariant (§14.2, §15). Cost projection rebased on 4 and 7 stamps (§18) |
| **Changes in v0.7** | **Cost model rebuilt on published rates (Scaleway Network Pricing, Sept 2026)** — the peering connector rate is confirmed at **€0.02/h** (risk #10 and open item 1 closed), **4× the €0.005 previously assumed**. Consequence: **peering now dominates platform cost** (64–77% of the estate run-rate) and the driver inverts back from stamp count to **spoke count**; the shared-non-prod-hub lever no longer touches the largest line (§18). Rate sheet, costed stamp and estate projection all restated. **Bandwidth mismatch surfaced** between the §8 capacity model and the default shard sizes — VPC-GW-S is 100 Mbps against ~950 Mbps of planned egress, LB-S is 200 Mbps — so prod defaults move to **VPC-GW-M / LB-GP-M** and a new risk (#16) tracks validation. DNS zones costed for the first time (§11, §18) |
| **Changes in v0.8** | **NVA sizing corrected against published Instance bandwidth (Scaleway Compute pricing, Sept 2026)**. The §8 worked example was impossible: it assumed 2,000 Mbps inspected on a **POP2-8C-32G whose line rate is 1.6 Gbps**, so by the document's own 30–50%-of-line-rate model the shard yields ~640 Mbps inspected and **~8 spokes, not 28** — "a single NVA shard covers a whole stamp" did not hold. Prod NVA moves to **COMPUTE3-X16C-32G** (4 Gbps, 16 dedicated physical cores, ~22 spokes) and `default_spoke_budget_mbps` becomes **per stamp class** (50 prod / 10 non-prod), since a POP2-2C-8G covers only 2 spokes at 50 Mbps (§7.1, §8, risk #17). POP2-2C-8G rate confirmed at €0.0735/h, replacing the estimate. Cost model restated: prod fixed base €340 → €470/mo (§18) |
| **Changes in v0.9** | **§18 restructured into four priced scenarios** — the landing zone itself, an empty workload, a workload with Kubernetes, and a workload with two 32 GB VMs — each costed *per workload instance* so the estate is an addition rather than a blended formula (§18.1–§18.7). Adds a worked estate example (~€5.6 k/mo for 4 stamps + 10 workloads) and separates the two cost questions that were previously conflated: **peering is 59–74% of the platform bill, compute ~65% of the whole bill** (§18.8). Kapsule's mutualized control plane confirmed **free**; instance and DNS rates folded into a single rate sheet (§18.2). **Block Storage is the one unsourced input** and is flagged as such (open item 9) |
| **Changes in v0.10** | **The NVA inspection stack is now specified rather than asserted.** v0.9 and earlier promised a "code-reviewed domain allowlist" without naming a mechanism — no FQDN enforcement existed anywhere in the design. §7.2 now defines a two-layer stack (**nftables** for L3/L4 and anti-bypass, **Squid** as an explicit CONNECT proxy for per-spoke FQDN policy, **Suricata** for IDS, **Unbound** scoped to the transparent fallback only), with a **three-tier enforcement ladder** for applications that ignore proxy settings — explicit proxy, then **SNI enforcement without decryption**, then expiry-dated IP exceptions (ADR-006, now *Accepted* in scope and *Proposed* on sizing). **TLS is not intercepted**: no corporate CA is distributed. Registry schema extended with per-destination ports and expiry-dated exceptions (§14.2); FQDN policy added to post-apply validation (§15). New risks #18–#20 cover proxy capacity, **Kapsule bootstrap through the proxy** (now a Phase 1 gate) and ECH erosion of SNI filtering; risks #6 and #17 amended for the stateful blast radius the proxy introduces |

---

---

## How this design is organised

The design is split along the **C4 model** — context, containers, components, code — read in that order for a top-down tour, or jump straight to a file.

C4 was built for software systems, so the fit is deliberate rather than literal: *containers* here are the Organization, projects and VPCs that form the isolation boundaries, and *code* is the Terraform that builds them. Roughly a third of the material — cost, risks, exit strategy, operations, decisions — sits outside the hierarchy, which is normal: C4 treats those as supplementary views rather than levels.

| # | File | Level | What it covers |
|---|---|---|---|
| 1 | [Context](01-context.md) | **C1** | Purpose and scope, the architecture overview diagrams, design principles |
| 2 | [Containers](02-containers.md) | **C2** | Organization and projects, identity and access, network architecture and stamps, traffic flows |
| 3 | [Components](03-components.md) | **C3** | Shared resource pools including the NVA inspection stack, capacity planning, DNS |
| 4 | [Code](04-code.md) | **C4** | Repositories and modules, the workload registry, pipelines, validation suite |

**Supplementary views**

| File | What it covers |
|---|---|
| [Operations](operations.md) | Availability and DR, observability and logging, security controls |
| [Cost model](cost.md) | Rate sheet plus four priced scenarios — landing zone, empty workload, Kubernetes workload, two-VM workload |
| [Exit strategy](exit-strategy.md) | Reversible lock-in, exit register, off-provider rebuild rehearsal |
| [Risks and roadmap](risks-and-roadmap.md) | Scale/quota risk register, phased roadmap, open items |
| [Decisions](decisions.md) | ADR-001 to ADR-015 |
| [Implementation plan](implementation-plan.md) | Phased build guide — the companion to this design |

## Section map

Section numbers are **unchanged** by the split, so every `§` cross-reference in the text still resolves. This table says which file to open.

| § | Section | File |
|---|---|---|
| §1 | Purpose and scope | [01-context.md](01-context.md) |
| §2 | Architecture overview | [01-context.md](01-context.md) |
| §3 | Design principles | [01-context.md](01-context.md) |
| §4 | Organization and project structure | [02-containers.md](02-containers.md) |
| §5 | Identity and access model | [02-containers.md](02-containers.md) |
| §6 | Network architecture | [02-containers.md](02-containers.md) |
| §7 | Shared resource pools | [03-components.md](03-components.md) |
| §8 | Capacity planning | [03-components.md](03-components.md) |
| §9 | Traffic flows | [02-containers.md](02-containers.md) |
| §10 | Availability and disaster recovery assumptions | [operations.md](operations.md) |
| §11 | DNS and domains | [03-components.md](03-components.md) |
| §12 | Observability and logging | [operations.md](operations.md) |
| §13 | Security controls | [operations.md](operations.md) |
| §14 | Delivery model (IaC) | [04-code.md](04-code.md) |
| §15 | Validation and testing | [04-code.md](04-code.md) |
| §16 | Exit strategy | [exit-strategy.md](exit-strategy.md) |
| §17 | Scale limits, quotas, and risk register | [risks-and-roadmap.md](risks-and-roadmap.md) |
| §18 | Cost model | [cost.md](cost.md) |
| §19 | Roadmap | [risks-and-roadmap.md](risks-and-roadmap.md) |
| §20 | Open items | [risks-and-roadmap.md](risks-and-roadmap.md) |
| — | Appendix A — Architecture Decision Records | [decisions.md](decisions.md) |

---

## Author and contact

| | |
|---|---|
| **Name** | Pim van Dijk |
| **Location** | Rotterdam Area, Netherlands |
| **Available via** | [Team Rockstars IT](https://www.teamrockstars.nl/) |
| **Email** | [pim@vandijkcloud.nl](mailto:pim@vandijkcloud.nl) · [pim.vandijk@teamrockstars.nl](mailto:pim.vandijk@teamrockstars.nl) |
| **Scaleway certifications** | Scaleway Foundations · Associate Network · Associate Security & Identity (see below) |

Questions about this design, or interested in a landing zone like this for your own estate? Feel free to reach out via either address above.

### Scaleway certifications

| Certification | Issued | Reference | Certificate |
|---|---|---|---|
| Scaleway Foundations | 15 April 2026 | `7646468504112674` | [PDF](../certifications/scaleway-foundations.pdf) · [verify](https://scaleway.360learning.com/redirect/api/certification/7646468504112674/authed/html) |
| Scaleway Associate: Network | 1 August 2026 | `8580743074158647` | [PDF](../certifications/scaleway-associate-network.pdf) · [verify](https://scaleway.360learning.com/redirect/api/certification/8580743074158647/authed/html) |
| Scaleway Associate: Security & Identity | 1 August 2026 | `5697254866632394` | [PDF](../certifications/scaleway-associate-security-identity.pdf) · [verify](https://scaleway.360learning.com/redirect/api/certification/5697254866632394/authed/html) |

> **Note:** the **verify** links go to Scaleway's learning platform (360Learning) and require you to be **signed in** — they will redirect to a login page otherwise. The **PDF** copies in this repository open without an account. Authenticity of any certificate can also be confirmed with Scaleway directly at [contact.certifications@scaleway.com](mailto:contact.certifications@scaleway.com). Each certificate is valid for 24 months from its issue date.

---
