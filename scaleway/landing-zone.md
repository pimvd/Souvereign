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

## 1. Purpose and scope

This document describes an enterprise-grade landing zone on Scaleway built around a hub-and-spoke network topology. It defines the environment model, the CIDR plan, the identity and segregation-of-duties model, the traffic flows, capacity and availability assumptions, and the delivery (IaC) model. It is built around **four instance-numbered environment stamps** — `prd01`, `acc01`, `tst01`, `dev01` — with three reserve blocks held for second instances, while keeping every network and identity decision under control of a central Platform (Landing Zone) team. Each stamp holds on the order of ~19 spokes, a ceiling set by Scaleway's Organization-wide Private Network quota rather than by addressing: the /12 allocated to each stamp would accommodate 1,022 (§6.1, §6.2, §17).

Out of scope for v0.1/v0.2: cross-region connectivity, hybrid/on-premises connectivity, session-recording bastion, managed SIEM integration. Availability assumptions and the path toward region resilience are documented in §10.

## 2. Architecture overview

### 2.1 Environment stamp — logical view

```
                              INTERNET
                                  |
              +-------------------+-------------------+
              |  ingress                       egress |
              v                                       ^
     +-----------------+                   +-----------------+
     |   LB Pool       |                   |   PGW Pool      |
     |  lb-0 [.. n]    |                   |  pgw-0 [.. n]   |  (NAT + bastion)
     +--------+--------+                   +--------+--------+
              |                                     ^
              |         +-----------------+         |
              |         |    NVA Pool     |---------+
              |         | nva-0 [.. n]    |
              |         |-----------------|
              |         | L3/L4  nftables |
              |         | FQDN   Squid    |  (explicit CONNECT proxy,
              |         | IDS    Suricata |   no TLS interception)
              |         | DNS    Unbound  |  (transparent fallback only)
              |         |  default deny   |
              |         +--------+--------+
              |                  ^     (1 shard/pool per small stamp;
              |                  |      pool scales out only on demand)
   +----------+------------------+----------------------------+
   |            HUB VPC  10.B.0.0/21  (one numbered stamp)     |
   |          (routes, ingress rules, private DNS)             |
   +---+--------------+--------------+---------------------+---+
       |peering       |peering       |peering              |peering
   +---v----+     +---v----+     +---v----+            +---v----+
   | Spoke 0|     | Spoke 1|     | Spoke 2|   . . .    |Spoke ~19|
   |  /22   |     |  /22   |     |  /22   |            |  /22    |
   +--------+     +--------+     +--------+            +---------+

   Spoke <-> Spoke traffic: none (no routes, denied by policy)

   Estate = four named stamps (prd01, acc01, tst01, dev01) + 3 reserve blocks.
   Each stamp = one /12; hub = first /21; spokes = /22 from block 2 up (§6.2).
   Spokes/stamp bounded by the 255 Private-Network-per-Org quota, not by
   addressing: ~19 per stamp across four stamps (§6.1, §17).
```

### 2.2 Organization and project hierarchy

```
Scaleway Organization
|
+-- plt-connectivity-prd01      (Hub VPC, pools, DNS, hub-side peering)
+-- plt-management-prd01        (Cockpit, audit, TF state, runners)
+-- wl-amazingapp-prd01         (Spoke VPC + workload resources)
+-- wl-<workload>-prd01         ... ≤ ~19 workloads per stamp (§17)
|
+-- plt-*/wl-*-acc01            (acceptance stamp)
+-- plt-*/wl-*-tst01            (test stamp)
+-- plt-*/wl-*-dev01            (dev stamp)
|
Numbered stamps: <class><NN>; classes prd / acc / tst / dev.
Four stamps allocated; 3 reserve /12 blocks held for second
instances (prd02, dev02, ...); 9 blocks unallocated (§6.2).

IAM (Organization level)
|
+-- grp-platform-humans              read-only, all projects
+-- app-platform-pipeline            platform write (network/IAM/DNS + GitHub admin)
+-- app-spoke-<name>-network-<env>   spoke-SIDE network write, own spoke project only
+-- grp-wl-<name>-humans             read-only, own wl-* projects
+-- app-wl-<name>-pipeline-<env>     workload app write, own project only

Repos: one `platform` repo + one `spoke-<name>` repo per workload (§14).
```

### 2.3 Spoke internal layout (/22)

```
+------------ Spoke VPC (10.B.x.0/22 — one workload, one stamp) -----+
|                                                                   |
|  +--------------------+   +---------------+   +---------------+   |
|  | pn-nodes   /23     |   | pn-app  /24   |   | pn-data /24   |   |
|  | Kapsule node pools |   | Instances     |   | Managed DBs   |   |
|  | node LBs, surge    |   | private LBs   |   | storage EPs   |   |
|  +--------------------+   +---------------+   +---------------+   |
|            ^                     ^                    ^           |
|            +------- VPC NACL: explicit allow + deny --+           |
|                                                                   |
+---------------------------- peering ------------------------------+
                                  |
                               HUB VPC
```

### 2.4 Traffic flows

```
EGRESS   workload -> spoke default route -> peering -> hub ingress rule
         -> assigned NVA shard (inspect/allowlist) -> assigned PGW (NAT) -> internet

INGRESS  client -> DNS -> assigned LB shard (TLS, health checks)
         -> peering -> backend private IP in spoke

MGMT     operator -> PGW bastion (SSH allowlist) -> private IP
         console: read-only for all humans
```

## 3. Design principles

1. **Segregation of duties.** The Landing Zone team owns everything network, identity, DNS, and platform configuration — including inside workload projects. Workload teams own only their application resources.
2. **Everything as code.** All changes flow through GitHub pull requests. Humans have read-only console access; only pipeline identities can write.
3. **Environment stamps.** An environment is a complete, independent copy of the architecture, addressed as a **numbered instance** (`prd01`, `acc01`, `tst01`, `dev01`). No resource, network, or peering is shared between stamps. The estate allocates four stamps today and holds three reserve blocks for second instances; numbering is kept so growth is an allocation rather than a redesign (§6.1, §6.2).
4. **Strictly spoke↔hub.** Spokes never communicate with each other. All ingress and egress traverses the hub for inspection.
5. **Deterministic addressing.** Every CIDR is derived by formula from a per-stamp base and a workload index. No IP address is ever hand-picked.
6. **Horizontal scalability of shared services.** Hub resources that could bottleneck (firewall NVAs, load balancers, public gateways) are deployed as **pools** with deterministic shard assignment, so scaling out is a configuration change rather than a redesign.
7. **Fail visible, not open.** Filtering defaults to explicit allow + trailing deny from the first apply; a missing rule blocks traffic rather than silently permitting it.
8. **Verify, then trust.** Every apply is followed by automated validation of routes, DNS, filtering, and connectivity (§15).
9. **Reversible lock-in.** Managed services are used freely, but every dependency is kept reversible — portable state export plus a single replaceable adapter module — and the exit is rehearsed, not assumed (§16, ADR-010).
10. **Repo and state per workload.** Each workload is its own Git repository with its own Terraform state; the Platform team owns a single `platform` repo that mints and governs the spoke repos as code (§14, ADR-011).

## 4. Organization and project structure

One Scaleway Organization contains all environments. Projects are the isolation and IAM-scoping boundary (see diagram §2.2).

| Project | Purpose | Contents |
|---|---|---|
| `plt-connectivity-<env>` | Hub for one environment | Hub VPC, NVA pool, LB pool, Public Gateway pool, DNS zones, peering connectors (hub side) |
| `plt-management-<env>` | Platform operations | Cockpit/observability, audit log sinks, Terraform state (Object Storage), optional GitHub runner pool |
| `wl-<workload>-<env>` | One workload, one stamp | Spoke VPC + workload resources (Kapsule, Instances, Managed Databases, storage) — both built from the workload's own `spoke-<name>` repo, network under a platform-controlled identity (§14) |

Here `<env>` is a **numbered stamp**: `<class><NN>`, two-digit zero-padded; classes are `prd`/`acc`/`tst`/`dev` (extensible). Four stamps are allocated — `prd01`, `acc01`, `tst01`, `dev01` — so projects read `plt-connectivity-acc01`, `wl-amazingapp-prd01`. Numbering is retained although only one instance per class exists today: a second instance (`dev02`) is an allocation from a reserve block (§6.2), not a rename of everything. Workload names are short kebab-case identifiers registered in the workload registry (§14.2). Each workload additionally has its own Git repository (§14.1); the platform pipeline holds **GitHub organisation-admin** rights to create and govern those repositories as code (the "repo-factory").

## 5. Identity and access model

### 5.1 Principals

Scaleway IAM permission sets are **product-scoped and project-scoped**; there is no resource-level RBAC and no explicit deny. Segregation therefore works by omission: each principal receives only the product permission sets it needs, in only the projects it needs.

| Principal | Type | Scope | Permission sets (indicative) |
|---|---|---|---|
| `grp-platform-humans` | IAM group | All projects | `*ReadOnly` across all products; **no** write sets |
| `app-platform-pipeline` | IAM application | Org + all current & future projects | `VPCFullAccess`, `PrivateNetworksFullAccess`, `VPCGatewayFullAccess`, `LoadBalancerFullAccess`, `DomainsDNSFullAccess`, `IPAMFullAccess`, `ProjectManager`, `IAMManager`; **plus GitHub organisation-admin** (repo-factory, outside Scaleway IAM). Builds every **hub-side** resource incl. the hub-side peering connector, hub routes and ingress rules |
| `app-spoke-<name>-network-<env>` | IAM application | `wl-<name>-<env>` project **only** | `VPCFullAccess`, `PrivateNetworksFullAccess`, `VPCGatewayFullAccess`, `LoadBalancerFullAccess` scoped to the spoke project. Builds the **spoke-side** resources (spoke VPC, PNs, NACL, spoke-side peering connector, spoke default route). **No rights in the hub VPC.** Platform-controlled: lives in the `spoke-<name>` repo's *network* environment, its key gated to platform `CODEOWNERS` (ADR-011/012) |
| `grp-wl-<name>-humans` | IAM group | `wl-<name>-*` projects only | `*ReadOnly` for compute, K8s, DB, storage products |
| `app-wl-<name>-pipeline-<env>` | IAM application | `wl-<name>-<env>` project only | `InstancesFullAccess`, `KubernetesFullAccess`, `RelationalDatabaseFullAccess`, `ObjectStorageFullAccess`, `ContainerRegistryFullAccess` (as needed). **No VPC/network sets.** Lives in the same repo's *app* environment |

### 5.2 Rules and caveats

The `IAMManager` permission set allows self-escalation by design; it is granted **only** to `app-platform-pipeline`. No human holds a write permission set in steady state; a break-glass procedure (time-boxed policy attached to a dedicated group, logged and reviewed within 24h) covers emergencies.

Workload pipelines deliberately lack Private Network permission sets. Attachment of resources to Private Networks is performed by the landing zone pipeline where possible; where a workload's Terraform must perform the attachment itself (e.g. Kapsule cluster creation referencing a PN ID), the workload pipeline receives `PrivateNetworksReadOnly` plus the product's FullAccess set, and consumes the PN ID as a read-only data source. This minimal exception is recorded per workload.

Because explicit deny does not exist, an **IAM policy audit** (diff of all live policies against the matrix above) runs in CI on every landing zone apply and nightly (§15).

### 5.3 Protection of the platform pipeline

The platform pipeline is the most privileged identity in the estate; its protection is explicit:

| Control | Implementation |
|---|---|
| Branch protection | `main` requires PR, no direct pushes, linear history |
| Required review | ≥1 reviewer from `CODEOWNERS` (platform team), stale approvals dismissed on new commits |
| CODEOWNERS | `/*` owned by platform team; IAM and peering paths require a second platform reviewer |
| Signed commits | Required on `main` |
| GitHub environments | Applies run in a protected `production` environment holding the API keys; plans run keyless or with read-only keys |
| API key handling | Keys exist only as GitHub environment secrets; issuance and ≤90-day rotation automated by the platform pipeline itself; no key ever in state, logs, or local machines |
| Drift detection | Nightly `terraform plan` against live estate; any drift raises an alert (drift implies out-of-band change — a policy violation by definition) |
| Repo-factory (GitHub admin) | The platform pipeline creates and governs every `spoke-<name>` repo as code — branch protection, `CODEOWNERS`, protected environments, and per-spoke Scaleway keys. The org-admin credential is held only as a protected environment secret and rotated ≤90 days |
| Split spoke repo | In each spoke repo the *network* path (spoke-side identity) is gated to platform `CODEOWNERS`; the *app* path is the workload team's. SoD is enforced by who may approve/apply each path, not by which repo (ADR-011) |

## 6. Network architecture

### 6.1 Environment stamps

Each environment is a **numbered stamp** (`<class><NN>`) consisting of one hub VPC and a small number of spoke VPCs, joined by VPC peering in a star topology. The estate allocates **four stamps** — `prd01`, `acc01`, `tst01`, `dev01` — and holds **three reserve /12 blocks** for second instances (`prd02`, `dev02`, …), with nine further blocks unallocated (§6.2). Instance numbering is retained even though one instance per class exists today, so growth is an allocation rather than a redesign.

**Stamp size is bounded by quota, not by addressing.** Each hub consumes 4 Private Networks (§6.3) and each spoke 3 (§6.4), against an Organization-wide Scaleway quota of **255 Private Networks**:

```
Σ over stamps ( 4 + 3 × spokes_in_stamp )  ≤  255
```

That yields **≈19 spokes per stamp** with four stamps allocated, or ≈10 if all seven blocks are ever built out (§17). Addressing allows 1,022 per stamp (§6.2), so the ceiling is a quota the registry must assert (§15), not a limit the formula reaches on its own. The registry therefore also carries an explicit per-stamp soft cap (default 60) so a mistake fails CI rather than silently minting spokes.

Traffic patterns are strictly spoke↔hub; because both egress (spoke → hub NVA → PGW) and ingress (hub LB → spoke) are single peering hops, transitive peering is not functionally required — Scaleway limits transitivity to four chained VPCs in any case. It is nevertheless enabled on hub VPCs at creation time as insurance, because the setting is immutable after creation (ADR-008).

### 6.2 CIDR plan

Requirements: /22 per spoke, a hub large enough for the §6.3 Private Networks, non-overlapping and human-legible addressing, deterministic derivation, and room for four stamps plus reserve. Each stamp receives **one /12** — the high nibble of the second octet *is* the stamp id — which keeps every log line, route, and NACL rule self-identifying (ADR-015, supersedes the /16-per-stamp scheme of ADR-013).

`10.0.0.0/8` divides into exactly **16 /12 blocks**, so:

```
S = octet2 >> 4          # stamp id, 0..15
B = S * 16               # base second octet
stamp_cidr = "10.${B}.0.0/12"
```

| S | Stamp | Block | Range |
|---|---|---|---|
| 0 | `prd01` | `10.0.0.0/12` | `10.0.0.0` – `10.15.255.255` |
| 1 | *reserve-1* | `10.16.0.0/12` | `10.16.0.0` – `10.31.255.255` |
| 2 | *reserve-2* | `10.32.0.0/12` | `10.32.0.0` – `10.47.255.255` |
| 3 | *reserve-3* | `10.48.0.0/12` | `10.48.0.0` – `10.63.255.255` |
| 4 | `dev01` | `10.64.0.0/12` | `10.64.0.0` – `10.79.255.255` |
| 5 | `tst01` | `10.80.0.0/12` | `10.80.0.0` – `10.95.255.255` |
| 6 | `acc01` | `10.96.0.0/12` | `10.96.0.0` – `10.111.255.255` |
| 7–15 | *unallocated* | `10.112.0.0` – `10.255.255.255` | 9 further blocks |

Reading a stamp off any address is one shift: `10.83.4.17` → `83 >> 4 = 5` → `tst01`. Reserve blocks are the landing spots for second instances; they sit adjacent to `prd01` so a `prd02` stays in the low range.

**Stamp id is assigned once, everything below it is derived.** Unlike ADR-013 — where `S` fell out of `class_base + (NN − 1)` — the id is now an explicit, reviewed registry entry chosen from the 16-block table. With only 16 blocks and 7 in use, an allocation table is clearer than a formula, and it lets `prd02` take a low block instead of whatever arithmetic dictates. Below the stamp, derivation is unchanged in spirit: **no address is ever hand-picked** (principle #5).

Within a stamp the hub takes the **first /21** (`10.B.0.0/21`, §6.3) — that is /22-blocks 0 and 1 — and workload *w* (0-based registry index) receives:

```hcl
spoke_cidr = cidrsubnet("10.${B}.0.0/12", 10, w + 2)
```

| w | `dev01` (B=64) | `tst01` (B=80) | `acc01` (B=96) | `prd01` (B=0) |
|---|---|---|---|---|
| 0 | `10.64.8.0/22` | `10.80.8.0/22` | `10.96.8.0/22` | `10.0.8.0/22` |
| 1 | `10.64.12.0/22` | `10.80.12.0/22` | `10.96.12.0/22` | `10.0.12.0/22` |
| 18 | `10.64.80.0/22` | `10.80.80.0/22` | `10.96.80.0/22` | `10.0.80.0/22` |
| 1021 (max) | `10.79.252.0/22` | `10.95.252.0/22` | `10.111.252.0/22` | `10.15.252.0/22` |

A /12 holds 1,024 /22 blocks; the hub takes 2, leaving **1,022 addressable spokes per stamp** — roughly 50× what the Private Network quota permits (§6.1). Addressing has deliberately stopped being the constraint, so CI must assert the quota budget instead (§15).

CI invariants assert no generated spoke CIDR overlaps its hub /21, no two stamps' /12s overlap, and that every stamp id is unique and drawn from the table above (§15). Because stamps are fully independent (ADR-004, principle #3), non-overlap is a convenience (clean logging, future optionality), not a hard requirement — but with 16 blocks against 7 in use there is no reason to overlap. Kapsule pod/service CIDRs remain pinned in `100.64.0.0/10` (§9), disjoint from the `10.0.0.0/8` plan.

### 6.3 Hub internal layout

The hub takes the **first /21** of the stamp's /12 (`10.B.0.0/21`), carved into Private Networks so that next-hop placement, NVA↔PGW wiring, and runner placement are explicit rather than implied:

| Hub Private Network | CIDR within the /21 | `dev01` example (B=64) | Purpose |
|---|---|---|---|
| `pn-hub-transit` | `10.B.0.0/23` | `10.64.0.0/23` | NVA shard interfaces receiving spoke traffic; target of hub ingress rules (next-hop IPs live here) |
| `pn-hub-egress` | `10.B.2.0/24` | `10.64.2.0/24` | NVA ↔ PGW leg; PGWs attach here and NAT the NVA-forwarded traffic |
| `pn-hub-ingress` | `10.B.3.0/24` | `10.64.3.0/24` | Hub LB shards; backends reach spokes via peering |
| `pn-hub-management` | `10.B.4.0/24` | `10.64.4.0/24` | Validation runners (§15), monitoring collectors, bastion-adjacent tooling |
| reserved | `10.B.5.0/24` + `10.B.6.0/23` | — | Pool growth, future HA legs |

Four Private Networks are created per hub, using 1,280 of the /21's 2,048 addresses and leaving 768 in reserve. The four count against the Organization-wide 255 Private Network quota (§6.1, §17); the reserved ranges cost nothing until a PN is actually created in them.

This answers the packet-path questions explicitly: a spoke's outbound packet arrives via its peering, the hub ingress rule delivers it to the assigned NVA's IP on `pn-hub-transit`, the NVA inspects and forwards out its `pn-hub-egress` interface, and the assigned PGW (attached to `pn-hub-egress`, advertising the default route on that PN only) NATs it — the PGW therefore sees the NVA's egress-leg IP as source. Return traffic reverses the same path. Because the default route toward a PGW exists only on `pn-hub-egress`, spoke traffic cannot bypass the NVA. Hub NACL rules enforce this as defense in depth (spoke ranges may only reach `pn-hub-transit` and published LB/management endpoints).

### 6.4 Spoke internal layout

Scaleway has no subnets within a Private Network; the equivalent of "subnets with ACLs" is multiple Private Networks within the spoke VPC, filtered by the VPC's Network ACL (diagram §2.3):

| Private Network | CIDR (offset within /22) | Purpose |
|---|---|---|
| `pn-nodes` | first /23 (512 IPs) | Kapsule node pools, node-attached LBs, upgrade surge headroom |
| `pn-app` | next /24 (256 IPs) | Instances, private load balancers, serverless attachments |
| `pn-data` | last /24 (256 IPs) | Managed Databases, storage endpoints |

The spoke NACL template ships with every spoke from first apply: explicit allows (app→data on database ports, nodes→app, hub ranges→all tiers for management and LB health checks) followed by a trailing deny. NACL rules are stateless, so both directions are declared. NACLs are currently API/Terraform-only (Public Beta), compatible with the everything-as-code model.

### 6.5 IPv6 position (ADR-007)

Each Private Network receives an immutable, auto-assigned IPv6 /64. The estate's stance: **IPv4-only for egress and ingress** (Public Gateways do not support IPv6; all public entry points are v4), **IPv6 filtered internally**. The IPv6 NACL mirrors the IPv4 ruleset (allow tier flows, trailing deny) and NVA rules drop v6 forwarding, so the auto-assigned /64s cannot become an unfiltered side channel. NAT64/DNS64 and dual-stack ingress are explicitly out of scope until Scaleway's gateway stack supports v6; revisit yearly.

### 6.6 Peering and routing

Each spoke peers with its stamp's hub via a **pair of connectors — one in each VPC** (Scaleway peering requires a connector on both sides, and only reaches `Peered` status once both exist). This two-sidedness *is* the duty boundary (ADR-012):

- the **spoke repo** (network path, identity `app-spoke-<name>-network-<env>`, scoped to the spoke project only) creates the **spoke-side** connector, the spoke default route `0.0.0.0/0` toward it, and the route to the hub /21;
- the **platform hub layer** (`app-platform-pipeline`) creates the **hub-side** connector, one hub route to the spoke /22, and one ingress rule directing inbound-from-spoke traffic to the assigned NVA shard (§7) — all generated by `for_each` over the registered spokes targeting that stamp.

Neither identity holds rights in the other's VPC, so no principal needs cross-VPC permissions and a spoke cannot self-connect to the hub. Onboarding is therefore a coordinated two-step that "meets in the middle": a registry PR provisions the hub side, the spoke repo provisions its side, and peering comes up when both halves land. Routine spoke changes never touch the hub state; only onboarding/offboarding/shard-reassignment do (a `for_each` addition, safe). **Published quotas confirm this scales.** Scaleway states no per-VPC peering cap at all; the limit is **512 peering connectors per Organization** (raisable on request). A stamp at 19 spokes uses 38 connectors, so four stamps use ~152 of 512. Hub-side filtering is bounded by the **255 IPv4 + 255 IPv6 Network ACL rules per VPC** quota — comfortable at ~19 spokes, but the dimension to watch if a stamp is ever grown, since NACL rules are stateless and declared in both directions. Scaleway publishes no per-VPC route quota; routes scale 1:1 with spokes and have not been observed to bind. The speculative "confirm before ~50 spokes" threshold of earlier drafts is retired — see §17 for the constraint that actually binds (Private Networks per Organization).

## 7. Shared resource pools

Hub services that process per-spoke traffic are modeled as **pools of identical shards** rather than singletons (ADR-005). The pattern applies to the NVA, LB, and PGW tiers and is reusable for future shared services.

### 7.1 The pool pattern

A pool is a map of shard definitions in the environment configuration:

```hcl
nva_pool = {
  "nva-0" = { instance_type = "COMPUTE3-X16C-32G", zone = "nl-ams-1" }   # prod (§8)
  "nva-1" = { instance_type = "COMPUTE3-X16C-32G", zone = "nl-ams-2" }   # only if a stamp is scaled out
}
lb_pool  = { "lb-0"  = { type = "LB-S",     zone = "nl-ams-1" } }
pgw_pool = { "pgw-0" = { type = "VPC-GW-M", zone = "nl-ams-1" } }
```

Each spoke registry record carries shard assignments, defaulted deterministically and overridable per spoke:

```hcl
default: nva_shard = "nva-${floor(spoke_index / spokes_per_nva)}"
         lb_shard  = "lb-${floor(spoke_index / spokes_per_lb)}"
```

Adding capacity is a two-line change (add a shard, adjust assignments); the pipeline regenerates ingress rules, routes, and LB frontends. Moving a spoke between shards is a single-field change with per-spoke blast radius.

Because a stamp is small (≤~19 spokes, §6.1), the **default is a single shard per pool per stamp**; the pool machinery exists so a hot stamp scales by configuration, but in practice it rarely engages within one stamp. The estate is spread across four single-shard hubs, not concentrated in one large pool (ADR-005).

### 7.2 NVA pool (egress and inspection)

Each NVA shard is an instance running the inspection stack defined in ADR-006, with configuration fully declared in the repository and applied via cloud-init — an NVA is disposable and recreatable by pipeline in minutes. Per-spoke hub ingress rules direct each spoke's outbound traffic to its assigned shard; each shard egresses through an assigned PGW. Day-one deployment is a single shard; the sharding mechanics exist from the first apply.

#### The stack

| Layer | Component | Responsibility |
|---|---|---|
| L3/L4 | **nftables** | IP/port/protocol rules; **anti-bypass** — a spoke may reach only the shard's DNS and proxy ports and its approved IP exceptions, never the internet directly |
| L7 name policy | **Squid** | Per-spoke **FQDN + port** allow rules, enforced on the `CONNECT` request line |
| Detection | **Suricata** | IDS signatures, anomaly detection, logging of attempted-but-denied flows (§12) |
| Resolution | **Unbound** | Scoped to the transparent fallback below — **not** the primary enforcement path, and not a replacement for private DNS (§11) |

Default policy at every layer is **deny**. The stack is generated from the workload registry (§14.2) — the same PR that registers an egress destination emits `nftables.conf`, `squid.conf`, `suricata.yaml` and `unbound.conf`, so there is one source of truth and no hand-edited appliance.

#### Two layers of enforcement

**nftables** fences the proxy. Without this, an FQDN policy is decoration — a workload that can open arbitrary outbound sockets simply routes around Squid:

```
# per spoke, generated
ALLOW  10.0.8.0/22 → nva :53    (DNS)
ALLOW  10.0.8.0/22 → nva :3128  (proxy)
ALLOW  10.0.8.0/22 → <approved IP exceptions>
DENY   10.0.8.0/22 → 0.0.0.0/0
```

**Squid** then enforces source spoke × destination FQDN × destination port:

```
acl spoke_amazingapp  src 10.0.8.0/22
acl amazingapp_hosts  dstdomain api.github.com login.microsoftonline.com
acl SSL_ports         port 443
http_access allow spoke_amazingapp amazingapp_hosts SSL_ports
http_access deny all
```

Wildcard destinations (`.example.com`) match a large surface and therefore require **named platform sign-off** recorded in the registry entry; they are not the default form.

#### TLS is not intercepted

Enforcement uses the **explicit `CONNECT` proxy**, so Squid sees the requested hostname while the TLS session stays end-to-end between the application and its destination:

```
app --CONNECT api.github.com:443--> Squid --policy check--> api.github.com
                                      TLS remains app <-> destination
```

**No corporate CA is generated or distributed.** This is a deliberate choice: it removes a private key of extraordinary value from the estate, keeps the platform out of the custody of workload payload data, and avoids the trust-distribution burden in every base image and language runtime. The trade-off accepted is that policy is expressed on hostnames, not payloads — content inspection of HTTPS is explicitly out of scope.

#### Applications that ignore proxy settings

Not everything honours `HTTP(S)_PROXY`. Enforcement therefore has three tiers, in order of preference:

| Tier | Mechanism | Applies to |
|---|---|---|
| 1 — **default** | Explicit proxy via `HTTP_PROXY`/`HTTPS_PROXY`/`NO_PROXY`, baked into node and workload base images | apt/yum, curl, Go, Java, most SDKs — the majority |
| 2 — **fallback** | **SNI enforcement**: nftables redirects :443 to the shard, which peeks at the TLS `ClientHello` SNI and applies the same registry policy. **Still no decryption, still no CA** | Clients that ignore proxy environment |
| 3 — **last resort** | Pinned IP/CIDR exception in the registry with a **mandatory expiry date**, reviewed quarterly | Genuinely proxy-hostile clients |

**Rejected: DNS-derived dynamic IP rules.** Resolving allowlisted names and programming the resulting addresses into nftables is fragile against CDNs and creates a race between resolution and connection. Unbound exists to serve and log tier 2, not to generate firewall state.

#### Accepted risks

Shards are individually single instances (by requirement) — mitigations: health checks with alerting, immutable configuration, pipeline-driven recreate, per-shard blast radius (risk #6). Note that Squid and Unbound make the shard **stateful**: in-flight proxied sessions and stamp DNS both terminate on the box, so "recreate in minutes" is a larger disruption than for a pure packet forwarder, and a proxy misconfiguration falls into §15's `manual-recovery-required` class. The proxy also introduces a capacity dimension the §8 forwarding model does not cover (risk #18).

### 7.3 Load balancer pool (ingress)

All inbound traffic enters via hub LBs, sharded by spoke assignment; DNS is the distribution layer (each workload hostname resolves to its shard's IP). Backends target private IPs in the spoke across the peering. Certificates (Let's Encrypt via LB, or imported) are per-frontend and landing-zone-owned.

### 7.4 Public Gateway pool

PGWs provide NAT for the NVA shards and host the built-in SSH bastion. The pool is typically small but modeled identically, so throughput or bastion-isolation needs are met by adding shards. Bastion access lists are landing-zone-managed; workload teams request access via PR.

## 8. Capacity planning

Sharding is only meaningful with numbers behind it. Capacity is planned per shard type using the formulas below; the concrete values marked *(validate)* are established in the Phase 1 proof of concept and re-measured after any instance-type change.

**NVA shard.** Effective capacity is the minimum of (a) instance network bandwidth — note Scaleway applies the bandwidth limit *per network connection*, so PN-facing and internet-facing capacity are separate budgets — and (b) inspection throughput, typically 30–50% of line rate with IDS enabled *(validate)*. Capacity is expressed in four dimensions, whichever binds first: `max_throughput_mbps`, `max_concurrent_sessions` (conntrack), `max_new_connections_per_second`, and `max_packets_per_second` — for Suricata, PPS is often the binding constraint before bandwidth *(validate all four in PoC)*.

**The proxy adds a fifth dimension this model does not cover.** Squid (§7.2) terminates TCP rather than forwarding packets, so its binding constraint is **concurrent proxied connections and CPU per `CONNECT` setup**, not PPS — a different capacity curve on the same instance. The shard sizing below is derived from the forwarding-plus-IDS model only; **`max_concurrent_proxy_connections` must be measured alongside the other four in the Phase 1 PoC** before the numbers are trusted (risk #18). Tier-2 SNI enforcement sits between the two models: it terminates nothing but does per-flow `ClientHello` parsing.

Two separate models are used. **Planning** (default shard assignment, uniform budgets):

```
spokes_per_nva = floor( shard_inspected_capacity_mbps × headroom_factor
                        / default_spoke_budget_mbps )
```

**Validation** (CI invariant per shard, honoring per-spoke overrides — checked on every registry change):

```
sum(egress_budget_mbps of assigned spokes) <= shard_inspected_capacity_mbps × 0.70
```

and equivalents for the session/CPS/PPS dimensions.

**Instance bandwidth is the ceiling on (a), and it binds harder than earlier drafts assumed.** Published line rates (Sept 2026) put POP2-8C-32G at **1.6 Gbps** and POP2-2C-8G at **400 Mbps** — so the "shard validated at 2,000 Mbps inspected → 28 spokes" example carried by v0.3–v0.7 was not achievable on the instance type it was costed against: inspected throughput cannot exceed line rate, and at the model's own 30–50% IDS factor a POP2-8C-32G yields ~480–800 Mbps. Because Scaleway applies the bandwidth limit *per network connection* (Scaleway VPC FAQ), the transit and egress legs each get the full rate, so the binding factor is inspection CPU rather than the NIC — which is why **dedicated physical cores matter** for the nftables+Suricata stack of ADR-006, and why the prod shard moves off the shared-vCPU POP2 line.

Sizing candidates at 40% of line rate inspected and the 70% usable invariant:

| Candidate | vCPU | Line rate | ≈ inspected | usable | spokes @ 50 Mbps | spokes @ 10 Mbps | €/mo |
|---|---|---|---|---|---|---|---|
| POP2-2C-8G | 2 (shared) | 400 Mbps | 160 | 112 | **2** | 11 | 53.65 |
| POP2-8C-32G *(old prod default)* | 8 (shared) | 1.6 Gbps | 640 | 448 | **8** | 44 | 211.70 |
| COMPUTE3-X8C-16G | 8 (dedicated) | 2 Gbps | 800 | 560 | 11 | 56 | 170.89 |
| **COMPUTE3-X16C-32G** *(prod default)* | 16 (dedicated) | 4 Gbps | 1,600 | 1,120 | **22** | 112 | 341.79 |
| POP2-24C-96G | 24 (shared) | 4.8 Gbps | 1,920 | 1,344 | 26 | 134 | 646.05 |

A 19-spoke stamp on the 50 Mbps default plans for 950 Mbps of aggregate egress and therefore needs ≥1,357 Mbps of inspected capacity at the 70% invariant. **COMPUTE3-X16C-32G is the prod default**: it clears that at 22 spokes with dedicated cores, and costs half of the only POP2 that also clears it.

**`default_spoke_budget_mbps` is now per stamp class** — **50 Mbps for prod, 10 Mbps for non-prod**. A POP2-2C-8G covers 2 spokes at 50 Mbps but 11 at 10 Mbps, which is the honest way to keep a ~€54/mo non-prod NVA; the alternative is a larger non-prod shard, which the cost model does not justify. With these two changes **a single NVA shard again covers a whole stamp** (§6.1), and the pool scales out only for an unusually hot stamp, driven by measured utilization (§12). Every number in the table above is derived from the 30–50% IDS heuristic and remains *(validate in the Phase 1 PoC — risk #17)*; the estate is still four single-shard hubs, so the fixed hub base multiplies across stamps (§18).

**LB shard.** Bounded by frontends/certificates per LB and connections/throughput per LB type. Published bandwidth per type is **LB-S 200 Mbps, LB-GP-M 500 Mbps, LB-GP-L 1 Gbps, LB-GP-XL 4 Gbps** (Sept 2026). The nominal `spokes_per_lb = 50` is a *frontend/certificate* bound, not a throughput one: at ~19 spokes an LB-S offers ~10 Mbps of ingress per spoke, so **LB-GP-M is the prod default** and LB-S is retained only for non-prod stamps with light ingress *(validate per stamp against measured ingress, risk #16)*. Multi-domain workloads (§11) consume extra per-frontend certificates, which counts against the LB cert limit — so effective `spokes_per_lb` drops for cert-heavy stamps.

**PGW shard.** Bounded by NAT throughput and session table. Published bandwidth is **VPC-GW-S 100 Mbps, VPC-GW-M 1 Gbps, VPC-GW-L 3 Gbps, VPC-GW-XL 10 Gbps** (Sept 2026). This is a hard constraint on the egress path: at ~19 spokes on the default 50 Mbps budget the stamp plans for ~950 Mbps of aggregate egress, which a VPC-GW-S (100 Mbps) would throttle by an order of magnitude well before the NVA's ~1,600 Mbps inspected capacity binds. **VPC-GW-M is therefore the prod default**; VPC-GW-S is acceptable only for non-prod stamps whose summed egress budget stays under ~70 Mbps. One PGW per stamp remains the default count; scale to one PGW per 2–3 NVA shards only where a stamp is scaled out *(validate — risk #16)*.

**Hub control plane.** Per stamp the hub carries one route, one ingress rule, and one connector per spoke — ~19 of each at the quota-bounded stamp size (§6.1). Against published Scaleway quotas (512 peering connectors per Organization, 255 NACL rules per VPC per address family, no per-VPC peering or route cap) this is not close to binding. The constraint that *does* bind the estate is the **255 Private Network per Organization** quota (§6.1, §17); VPCs (256/Org) and public IPs are the next dimensions to watch.

Capacity assumptions, measured values, and current shard assignments are published on the platform dashboard (§12) so rebalancing decisions are data-driven.

## 9. Traffic flows

**Egress (spoke → internet).** Workload resource → spoke default route → peering → hub ingress rule → assigned NVA shard (inspection, allowlist) → assigned PGW (NAT) → internet. Spokes have no Public Gateways and no public IPs.

**Ingress (internet → spoke).** Client → DNS (landing-zone-managed hostname) → assigned LB shard → TLS termination and health checks → backend private IP in the spoke via peering. A per-spoke flag can insert NVA inspection into the ingress path for high-sensitivity workloads.

**Management (operator → spoke).** Operator → PGW bastion (SSH, access-listed) → target private IP. Console is read-only; bastion access is the logged exception path.

**Spoke ↔ spoke.** Not permitted: no routes exist between spoke CIDRs, and NVA policy denies inter-spoke ranges as defense in depth.

**Kapsule specifics.** Node pools live in `pn-nodes`; nodes require control-plane, registry, and image endpoints on the NVA allowlist to bootstrap. Pod/service CIDRs are pinned in the spoke template from `100.64.0.0/10` to guarantee no overlap with the 10.0.0.0/8 plan.

## 10. Availability and disaster recovery assumptions

Current explicit target: **zone-resilient where the platform allows it, not region-resilient.**

| Layer | Current posture |
|---|---|
| VPC / Private Networks | Regional by design; survive zone loss |
| Peering, routes | Regional; survive zone loss |
| NVA shards | Zonal instances; pool *may* span zones (shards in different AZs), but a zone loss takes down that zone's shards and their assigned spokes' egress until recreate/reassign |
| LB / PGW shards | Zonal; same model — pool spread across zones limits blast radius |
| Kapsule, Managed DB | Multi-AZ options chosen per workload; workload teams' responsibility within their spoke |
| Region (`nl-ams`) loss | **Full environment outage. Accepted for now.** |

Recovery model: the entire estate is recreatable from Git (infrastructure) plus workload data restores (each workload documents RPO/RTO for its own data; Managed DB backups and Object Storage versioning are the defaults). A quarterly **stamp-rebuild exercise** in dev validates that "recreate from repo" actually works and is timed.

Future path to region resilience (not committed): second-region stamps on the reserved ranges (§6.2), DNS-based failover for ingress, per-workload data replication. Cross-region peering does not exist on Scaleway; stamps would be fully independent, which the environment-stamp model already supports.

**Cost-driven variant — shared non-prod hub.** Because every stamp carries a full always-on hub base (§18), non-prod stamps *may* share one hub — `dev*`, `tst*` and `acc*` spokes peering into a single shared hub — to collapse three hub bases into one, trading inter-stamp isolation for cost. It also reclaims Private Network headroom (8 PNs saved, §6.1). **Prod stamps are never shared.** Ephemeral dev stamps additionally support a TTL/auto-suspend profile. This is the single biggest cost lever (§18) and is a supported deployment shape, selected by a per-stamp flag rather than a redesign.

## 11. DNS and domains

Public zones are owned by the platform in `plt-connectivity-<env>` (or a shared DNS project where zones span stamps). The estate can host **multiple apex domains** — e.g. `example.com` and `example.nl` — each apex being its own platform-owned public zone; the managed set is declared explicitly as `dns_zones` in the registry (§14.2), and the platform pipeline holds `DomainsDNSFullAccess` across all of them. Workloads receive names under any managed apex (`<workload>.<env>.example.com`, a vanity host, or several domains at once) via records requested by PR. Each published hostname is a landing-zone-owned LB frontend with its own certificate (Let's Encrypt via LB, or imported); publishing on multiple domains therefore consumes more per-frontend certificates, which counts against LB cert limits (§8). Each public zone is billed per zone (€0.007/h ≈ €5.11/mo, 5 M requests included, then €0.0005/million), so the managed apex set is a small standing estate-level cost rather than a per-stamp one — two apexes ≈ €10/mo (§18). Private resolution is strictly spoke↔hub: spokes resolve hub-published service names via Scaleway's built-in private DNS per Private Network; no cross-spoke discovery exists by design. **Unbound on the NVA (§7.2) does not replace this.** Private-name resolution stays on Scaleway's per-PN resolver; Unbound serves and logs *public* name resolution for the tier-2 transparent path only. Where tier 1 (explicit proxy) is in force the proxy already sees the hostname, so Unbound is not on the enforcement path at all — a distinction worth keeping explicit, because putting resolution on the NVA otherwise widens its blast radius for no benefit (risk #6).

## 12. Observability and logging

All telemetry converges in `plt-management-<env>` (Cockpit), with prod audit data additionally protected against tampering.

**Log routing.** NVA shards ship flow logs, IDS alerts, and allowlist denials; PGWs ship NAT/bastion session logs; LBs ship access logs; Kapsule and instances ship via the standard agents into the workload's Cockpit scope with copies of security-relevant streams to the platform scope. Scaleway audit trail (IAM changes, resource mutations) is exported continuously.

**Immutable audit.** Audit and bastion logs are additionally written to an Object Storage bucket with versioning and a compliance-style retention lock; the platform pipeline itself cannot delete within the retention window.

**Retention.** Defaults: metrics 13 months, application logs 30 days, security/flow logs 90 days, audit logs 400 days (immutable bucket). Per-workload overrides via the registry.

**Alerting.** Platform alert set: NVA/PGW/LB shard health, drift detection findings, IAM audit diffs, certificate expiry, peering/route validation failures, capacity thresholds (≥70% of any shard budget), and budget anomalies. Routed to the platform team's on-call channel.

**Dashboards.** One platform dashboard per stamp (rolled up across the estate): shard utilization vs. capacity model (§8), spoke count vs. quota headroom, per-stamp and total fixed-base cost run-rate, **peering spend per workload** and **compute spend per stamp** (§18.8), top egress destinations, denied-flow trends.

## 13. Security controls

Beyond the IAM model (§5) and network segmentation (§6), the delivery chain enforces:

| Control | Tooling | Gate |
|---|---|---|
| IaC static analysis | tfsec + Checkov on every PR | Blocking on high severity |
| Policy-as-code | OPA/Conftest rules: no public IPs in spokes, no network resources in workload repos, NACL template present, naming/CIDR conformity | Blocking |
| Dependency updates | Dependabot on all repos (providers, modules, actions) | Auto-PR |
| Container/image scanning | Trivy in workload pipelines; base-image policy | Blocking on critical |
| Secret hygiene | Push protection / secret scanning on all repos; no secrets in Terraform state (validated in CI) | Blocking |
| Drift detection | Nightly plan against live estate (§5.3) | Alert |
| IAM audit | Live policies diffed against §5.1 matrix, nightly + on apply | Alert + blocking on unexpected grants |
| Key lifecycle | ≤90-day automated rotation of IAM application keys | Alert on overdue |
| Egress destination control | Per-spoke FQDN + port allowlist enforced on the NVA (§7.2), generated from the registry; wildcards and IP exceptions require named sign-off and expire | Blocking (PR review); denied flows logged and alerted (§12) |

GitHub Advanced Security features beyond the free tier (code scanning on private repos) are a licensing decision, not an architectural one; the controls above function without it.

## 14. Delivery model (IaC)

### 14.1 Repositories and module layout

Fully IaC, one repo for the platform plus **one repo per workload**. Two Terraform state boundaries per workload (network, app) and a per-stamp state for the platform layer (ADR-011).

| Repository | Owned / governed by | Contains |
|---|---|---|
| `platform` (one) | Platform team | Foundation (org/projects, IAM, DNS zones, observability, state backends, GitHub governance / repo-factory), the hub layer (hub VPC, PNs, pools, **hub-side** peering + routes + ingress rules), the **versioned** module library, the workload registry, and policy-as-code |
| `spoke-<name>` (one per workload) | Platform (network paths) **+** workload team (app paths) | The workload's **spoke network** state (spoke VPC, 3 PNs, NACL, **spoke-side** peering + default route) and its **application** state (Kapsule, Instances, Managed DB, storage). Two states, two protected GitHub environments, two identities; `CODEOWNERS` gates `/network/` to the platform team |

```
platform/
├── modules/                    # versioned & tagged; spoke repos pin a release
│   ├── foundation/             # org, projects, IAM, DNS zones, state backends, repo-factory
│   ├── hub/                    # hub VPC, hub PNs, baseline routing
│   ├── pool-nva/ pool-lb/ pool-pgw/
│   ├── hub-peering/            # hub-SIDE connector + route + ingress rule for one spoke (for_each)
│   ├── observability/          # Cockpit, alerts, immutable audit bucket
│   └── iam/                    # groups, applications, policies, key rotation
├── stamps/                     # one root module (state) per numbered stamp
│   ├── prd01/                  # backend.tf, prd01.tfvars (pools, budgets, shared-hub flag)
│   ├── acc01/ tst01/
│   └── dev01/                  # reserve blocks -> prd02/, dev02/, ... when allocated (§6.2)
├── registry/
│   └── workloads.hcl           # single source of truth (§14.2)
├── repo-factory/               # GitHub-provider config: mints & governs spoke repos
└── policies/                   # OPA/Conftest rules (§13)

spoke-<name>/                   # minted from a template by the repo-factory
├── network/                    # identity: app-spoke-<name>-network-<env> (platform-gated)
│   └── <stamp>/                # one dir/state per numbered stamp the workload targets
├── app/                        # identity: app-wl-<name>-pipeline-<env> (workload team)
│   └── <stamp>/
├── CODEOWNERS                  # /network/ → platform ; /app/ → workload team
└── .github/                    # per-stamp environments + secrets (injected by repo-factory)
```

Because each spoke is its own repo with a single small root per stamp, there is **no "many roots in one repo" problem and no Terragrunt is required**; the DRY mechanism is the versioned module library that every spoke pins. Module upgrades roll out per-spoke (bump the pinned tag), so blast radius stays per-workload.

### 14.2 Workload registry

One record per workload, single source of truth, kept **central in `platform`**:

```hcl
dns_zones = ["example.com", "example.nl"]   # platform-owned public apexes (§11)

workloads = {
  "amazingapp" = {
    index            = 0        # 0-based; drives per-stamp CIDR (§6.2)
    stamps           = ["prd01", "acc01", "tst01", "dev01"]   # numbered instances
    nva_shard        = null     # null = deterministic default (§7.1)
    lb_shard         = null
    egress_budget    = 50       # Mbps, feeds capacity model (§8)
    ingress = [
      { host = "api.amazingapp.example.com", backend_pn = "pn-app", port = 443 },
      { host = "api.amazingapp.example.nl",  backend_pn = "pn-app", port = 443 },
    ]
    # egress policy -> generated into nftables/squid/suricata/unbound (§7.2)
    egress = {
      # tier 1/2: FQDN + port, enforced by Squid on CONNECT (or on SNI)
      fqdn = [
        { host = "api.anthropic.com", ports = [443] },
        { host = "api.github.com",    ports = [443] },
        # wildcards need named platform sign-off - large surface (§7.2)
        { host = ".pkg.example.com",  ports = [443], wildcard_approved_by = "platform-team", ticket = "LZ-142" },
      ]
      # tier 3: proxy-hostile clients only. expires_on is MANDATORY; CI fails on
      # a past date, and the quarterly review re-justifies or drops the entry.
      ip_exceptions = [
        { cidr = "203.0.113.7/32", ports = [8443], reason = "vendor SFTP, no proxy support",
          expires_on = "2027-03-31", ticket = "LZ-118" },
      ]
    }
  }
  # … Σ over stamps (4 + 3 × spokes) ≤ 255 Private Networks per Org (§6.1)
}
```

The registry stays central because the **cross-stamp invariants** — unique `index`, no CIDR overlap, shard-capacity sums, and the Organization-wide Private Network budget (§6.1) — can only be checked where every workload is visible (§15). A workload's own repo carries only the *workload-owned* fields (app config, requested hosts, requested egress destinations) that flow into the registry by PR. Egress entries are **requested** by the workload and **granted** by the platform: the registry is in the `platform` repo under platform `CODEOWNERS`, so a workload cannot widen its own egress policy — the same segregation that keeps network rights out of workload hands (§5, principle 1).

Onboarding a workload is a coordinated two-step: **(1)** a registry PR in `platform` — the pipeline provisions the hub-side wiring for each targeted stamp and mints the `spoke-<name>` repo (project, IAM applications + keys, branch protection, `CODEOWNERS`, environments) via the repo-factory; **(2)** the new spoke repo's own `network` then `app` applies build the spoke side and the workload. Validation (§15) runs after each.

### 14.3 Pipelines and authentication

GitHub Actions is the only write path. Each spoke repo holds two protected GitHub environments: **network** (key = `app-spoke-<name>-network-<env>`, scoped to the spoke project, its paths gated to platform `CODEOWNERS`) and **app** (key = `app-wl-<name>-pipeline-<env>`). Scaleway authentication uses IAM application API keys as GitHub environment secrets (no OIDC federation available); issuance and ≤90-day rotation per §5.3. Terraform state lives in Object Storage in `plt-management-*` with locking — **per stamp** for the platform layer and **per spoke per stamp** for spoke repos. Plans post to PRs; applies run only from protected default branches in protected environments.

### 14.4 Bootstrap (the one manual seam)

Everything is IaC except a single root seam, created **manually once** and documented: (1) the Object Storage bucket that holds Terraform state, and (2) the first `app-platform-pipeline` API key. From there the platform pipeline is self-hosting — it creates projects, IAM applications and their keys, the repo-factory, every spoke repo, and rotates all keys (§5.3). No per-spoke credential is ever created by hand; the manual step is exactly **one bucket + one key**.

### 14.5 Cross-repo contract

Spoke repos never read platform state directly (SoD). The platform hub layer **publishes** a small read-only contract per stamp — hub VPC/PN IDs, each workload's assigned CIDR and shard, DNS zone IDs — to a well-known location (a published-outputs object / parameter store) that spoke repos consume via a scoped data source. The reverse direction (a spoke requesting a host, an allowlist entry, bastion access, or a shard move) is always a PR into the `platform` registry. No shared **mutable** state crosses the boundary; the contract is versioned alongside the module release the spoke pins (§14.1).

## 15. Validation and testing

Validation is split into **pre-apply gates** (which genuinely block a change) and **post-apply verification** (a change that already landed cannot be "blocked" — it triggers a defined response).

**Pre-apply (blocking, in CI on every PR):** Terraform validate/plan review; policy-as-code suite (§13); generated-route invariants — exactly one hub route per registered spoke, no route to unregistered CIDRs, **no spoke CIDR overlapping the hub /21**, **no two stamps' /12s overlapping** (§6.2); IAM permission diff against the §5.1 matrix; registry consistency (unique indices, unique stamp ids drawn from the §6.2 table, every workload's `stamps` reference a defined stamp, the **Private Network budget `Σ over stamps (4 + 3 × spokes) ≤ 255`** and the per-stamp soft cap, budgets within shard validation invariants §8); **egress-policy hygiene** — every wildcard FQDN carries `wildcard_approved_by`, every `ip_exceptions` entry carries a future `expires_on`, and no entry grants a port outside the workload's declared protocol set (§14.2). These cross-stamp checks run in `platform`, where every workload is visible (§14.2); the conformance suite below then runs per stamp and per new spoke.

**Post-apply (verification, from runners in `pn-hub-management` plus a canary per new spoke):**

1. **Peering** — all connectors `Peered`; none `Orphan`/`Conflict`.
2. **Routing** — spoke default route resolves via the correct connector; hub ingress rule delivers to the assigned NVA on `pn-hub-transit`.
3. **Filtering** — canary in `pn-app` reaches `pn-data` on allowed ports and not on denied ones; inter-spoke probe fails; IPv6 probe fails per ADR-007.
4. **Egress (L3/L4)** — allowlisted destination reachable via assigned NVA shard; non-allowlisted probe denied and logged; PGW sees the NVA egress-leg source IP (§6.3); **direct-to-internet probe from the spoke fails** (anti-bypass, §7.2).
4b. **Egress (FQDN policy)** — for each spoke: a registered FQDN succeeds through the proxy; an *unregistered* FQDN is refused by Squid and logged; a registered FQDN on an *unregistered port* is refused; the same three probes repeat over the tier-2 transparent path with proxy environment unset; and a canary confirms **TLS is not intercepted** (the destination's own certificate chain is presented, no platform CA).
5. **Ingress** — synthetic request to the workload hostname healthy through the assigned LB shard; certificate valid ≥30 days.
6. **DNS** — public records match registry; private hub names resolve from the spoke.

**Failure response per change class** (declared per module, since automatic rollback of routing/NACL changes can itself cause harm):

| Class | Examples | On post-apply failure |
|---|---|---|
| `rollback-safe` | LB frontends, DNS records, allowlist entries | Automatic revert to previous state |
| `manual-recovery-required` | Routes, ingress rules, NACLs | Incident + promotion block; guided runbook, no auto-revert |
| `forward-fix-only` | IAM policies, project/VPC creation | Incident + promotion block; fix rolls forward |

Promotion (`tst01` → `acc01` → `prd01`) requires a green post-apply suite in the preceding stamp. The same suite runs nightly as continuous verification, and the quarterly dev stamp-rebuild (§10) uses it as its acceptance gate.

**Phase 1 architecture acceptance gate (risk #3):** no production stamp is approved until a hub LB has health-checked backends in at least two different spoke VPCs, served TLS to both, survived backend and route changes, demonstrated the expected source IP at the backend, and demonstrated symmetric return-path routing.

## 16. Exit strategy

This chapter defines the estate's stance on provider lock-in. The goal is **exit capability** — the ability to rebuild the estate on a different cloud provider within a bounded, agreed time — **not** active multi-cloud operation and **not** a lowest-common-denominator abstraction layer. We stay deliberately provider-native (managed LB pool, Kapsule, Managed Databases, Cockpit) and pay no continuous portability tax; in return, every lock-in point is kept *reversible* and the exit is rehearsed so the capability is measured rather than asserted. This chapter is provider-neutral: the target provider is left unnamed, since the design commits to a mechanism, not a destination.

### 16.1 Objective and non-goals

**Objective.** From a standing start, stand up a functioning environment stamp (§6.1) on a target provider and restore workload data such that the same validation suite (§15) passes, within an agreed **time-to-exit** budget (§16.6).

**Non-goals.**

- *Not* active multi-cloud — workloads run on one provider at a time; we do not pay to run everywhere simultaneously.
- *Not* a cloud-agnostic abstraction — no lowest-common-denominator layer that would forfeit provider-native strengths and leak anyway.
- *Not* self-hosting for its own sake — managed services are retained wherever they carry a tested export path.

The managed LB pool (§7.3) is retained. On exit, its `pool-lb` module is re-implemented against the target provider's load balancer — bounded adapter work (§16.3), not a blocker. (A self-managed reverse-proxy ingress would shrink this adapter but is out of scope for now.)

### 16.2 Principle — reversible lock-in (ADR-010)

Lock-in is acceptable when it is **reversible**. A dependency is reversible when *both* hold:

- **(a) State export** — any state it holds can be extracted in a provider-independent format; and
- **(b) Bounded replacement** — it can be replaced by re-implementing a single adapter module against an equivalent primitive on the target provider, without redesign.

A dependency that fails either test is a **one-way door**, governed by §16.7. The operative discipline is therefore *not* "avoid managed services" but "ensure each managed service satisfies (a) and (b)" — which preserves the lean, provider-native posture of the rest of this document.

### 16.3 Portable core and provider adapters

The estate is already structured as a portable core plus provider-specific adapters (ports & adapters). Exit re-implements the adapters, never the core.

**Portable core** (provider-independent, unchanged on exit):

- the workload registry contract (§14.2) — contains no provider nouns;
- the validation/conformance suite (§15) — the executable definition of a correct stamp;
- deterministic addressing math (§6.2) and the pool/shard pattern (§7);
- policy-as-code intent (§13) and the IAM matrix expressed as *role → permission intent* (§5.1);
- the stamp / recreate-from-Git model (§10).

**Provider adapters** (re-implemented per provider): the `hub`, `spoke`, `peering`, `iam`, `pool-nva`, `pool-lb`, `pool-pgw`, `dns`, and `observability` modules (§14.1).

Exit is thus defined operationally as **a new adapter set that makes the existing validation suite green on the target provider** — not the same Terraform running everywhere.

### 16.4 Exit register

The living inventory of lock-in points, each with its state-export path and its replacement-adapter effort — maintained beside the risk register (§17) and reviewed on the same cadence. Exit class: **config** (rebuilds from Git, holds no state), **data** (requires export + restore), **one-way** (fails §16.2 — none permitted without an ADR, §16.7).

| Lock-in point | State export path | Replacement adapter (effort) | Exit class |
|---|---|---|---|
| VPC / PNs / peering / NACLs (§6) | None — topology is config | `hub`, `spoke`, `peering` on target primitives | config (medium) |
| IAM model (§5) | Policy intent in Git (role→intent matrix) | `iam` re-mapped to target policy language | config (medium) |
| Managed Databases (§4) | Logical dumps (e.g. SQL) to Object Storage | Provision target DB, restore dump | **data (medium — gravity)** |
| Object Storage | S3-API copy-out / off-provider replication | Repoint to target S3-compatible store | data (low) |
| Container Registry | Mirror images to target registry | Repoint registry | config (low) |
| Kapsule — managed k8s (§4) | Manifests/Helm in Git; avoid proprietary CRDs | Target managed or self-managed k8s | config (low–med) |
| **LB pool (§7.3) — retained** | None — frontends/certs re-issued (ACME/import) | `pool-lb` on target load balancer | config (low–med) |
| Public Gateway pool (§7.4) | None | `pool-pgw` on target NAT + bastion | config (low) |
| NVA pool (§7.2) | None — cloud-init config in Git | Re-target compute; stack unchanged | config (low — already portable) |
| Cockpit / observability (§12) | Dashboards/alerts as code; history disposable | `observability` on Prometheus/Grafana or managed equiv | config (low; telemetry loss accepted) |
| DNS / Domains (§11) | Zones as code; registration independent of host | `dns` on target provider | config (low) |
| Terraform state backend (§14.3) | S3-compatible state object | Migrate backend to target store | config (low) |

Secrets carry no cloud lock-in: they live as GitHub environment secrets (§14.3), already provider-neutral.

### 16.5 Data export — the binding constraint

Everything except persistent state rebuilds from Git; the exit ceiling is therefore set by how fast state can be extracted and restored. Requirements:

- every stateful managed service ships **logical, provider-independent backups** (e.g. SQL dumps) to Object Storage, **in addition to** provider-native snapshots — the latter restore only on the originating provider and do not count toward exit;
- Object Storage stays within the portable S3-API subset, with a copy-out / replication path off-provider;
- container images are mirrored to a registry the target can pull from;
- each workload's §10 RPO/RTO is extended with a **restore-on-target-provider** time, which feeds the time-to-exit SLO (§16.6).

### 16.6 Rehearsal and the time-to-exit SLO

An unexercised exit plan rots. The quarterly stamp-rebuild exercise (§10) already proves "recreate from Git" *on*-provider; exit capability additionally requires proving it *off*-provider.

- **Time-to-exit SLO:** a functioning dev stamp plus restored workload data on a target provider, validation suite (§15) green, within *(validate — set target, e.g. N weeks, in the Phase 1 PoC)*.
- **Cadence:** at least annually, the dev stamp is rebuilt on an alternate provider using a maintained adapter set; the validation suite is the acceptance gate; measured wall-clock time becomes the current time-to-exit, published on the platform dashboard (§12).
- Any adapter that has **never** passed the suite on the target provider is *not* counted toward exit capability — the exit register (§16.4) marks it unproven.

### 16.7 Governing new lock-in

Because exit capability degrades silently, adoption of new dependencies is gated:

- any net-new service that fails the reversible-lock-in test (§16.2) — no portable state export, or no cross-provider equivalent — is a **one-way door** and requires an **exit-impact ADR** with platform sign-off before adoption;
- where mechanically checkable, policy-as-code (§13) flags use of services outside the approved, register-listed set;
- the exit register (§16.4) is updated in the **same PR** that introduces or changes a platform dependency, exactly as the registry drives everything else.

## 17. Scale limits, quotas, and risk register

| # | Item | Impact | Status / mitigation |
|---|---|---|---|
| 1 | Connectors / routes / ingress-rules per hub VPC | **Closed** — Scaleway publishes **no per-VPC peering cap**; the cap is **512 connectors per Organization** (raisable). ~152 used at four stamps × 19 spokes | Confirmed against Scaleway VPC Peering FAQ and quotas docs (Sept 2026). The earlier "~50 spokes" threshold was self-imposed, not a Scaleway limit, and is retired |
| 2 | **Private Networks per Organization = 255** (hub 4 + spoke 3 each) | **Binding constraint on the whole estate** — caps ~19 spokes/stamp at four stamps, ~10 at seven | CI invariant `Σ (4 + 3 × spokes) ≤ 255` (§15); request a quota increase from Scaleway Support before exceeding it; collapsing `pn-data` into `pn-app` would buy ~33% headroom at the cost of §6.4 tiering |
| 3 | LB backend health checks across peering unproven at scale | Ingress design risk | Phase 1 PoC gate |
| 4 | NACL in Public Beta, API-only | Feature risk | Acceptable (IaC-only estate); track GA |
| 5 | No explicit deny in IAM | Segregation by omission | Automated IAM audit (§13) |
| 6 | NVA shard is a single instance, and with Squid/Unbound it is now **stateful** | Egress outage for assigned spokes; in-flight proxied sessions and tier-2 DNS die on recreate, so "recreate in minutes" is a larger disruption than for a packet forwarder | Health checks with alerting, immutable configuration, pipeline recreate, per-shard blast radius; private DNS deliberately left on Scaleway's per-PN resolver so it survives NVA loss (§11); HA (active/passive per shard) pulled forward in the roadmap |
| 7 | No OIDC federation for GitHub | Long-lived keys | Automated ≤90-day rotation, least-privilege per repo |
| 8 | Immutable IPv6 /64 per PN | Unfiltered v6 path if ignored | ADR-007: parallel v6 filtering, validated in §15.3 |
| 9 | Region loss = environment loss | Availability | Accepted (§10); quarterly rebuild exercise; region resilience on roadmap |
| 10 | VPC peering connector rate | **Closed** — published at **€0.02/h per connector** (~€14.60/mo), i.e. **€29.20/mo per spoke** for the two-sided connection. 4× the previously assumed €0.005/h | Confirmed against Scaleway Network Pricing (Sept 2026). It does not merely rival the hub bases — it **exceeds them**: peering is 59–74% of platform run-rate (§18). Spoke count is now the primary cost lever |
| 11 | Exit capability asserted but unproven until first off-provider rebuild | Strategic / lock-in | §16: exit register + annual off-provider rebuild, validation suite as acceptance gate |
| 12 | Fixed hub base × stamp count is the dominant cost, multiplied across stamps | Standing cost | Shared non-prod hub + TTL/auto-suspend; smaller non-prod instance types (§10, §18) |
| 13 | Network ACL rules capped at **255 IPv4 + 255 IPv6 per VPC**; NACLs are stateless so every flow is declared twice | Hub filtering ceiling | Comfortable at ~19 spokes; re-check before any stamp grows, and prefer summarised spoke ranges over per-spoke rules where the policy allows |
| 14 | Scaleway publishes **no per-VPC route quota** | Unknown ceiling | Routes scale 1:1 with spokes (~19/hub); confirm with Support alongside the item #2 quota request |
| 15 | Transitive peering limited to **four chained VPCs** | Would block any future spoke↔spoke-via-hub design | Not required today — all flows are single-hop (ADR-004); transitivity is enabled at hub creation anyway because the setting is immutable (ADR-008) |
| 16 | Default shard sizes undersized against the §8 capacity model: **VPC-GW-S is 100 Mbps** vs ~950 Mbps planned stamp egress; **LB-S is 200 Mbps** | Egress throttled ~10× before the NVA binds; ingress ceiling on prod | Prod defaults moved to **VPC-GW-M** (1 Gbps) and **LB-GP-M** (500 Mbps) in §8, at +€50/mo and +€23/mo per prod stamp. Validate measured egress/ingress in the Phase 1 PoC before ratifying the non-prod VPC-GW-S choice |
| 17 | **NVA shard was undersized against its own instance's line rate** — the 2,000 Mbps / 28-spoke example was impossible on a 1.6 Gbps POP2-8C-32G (~8 spokes at the 50 Mbps budget) | Capacity model overstated a stamp's spoke ceiling by ~3.5× | Prod NVA moved to **COMPUTE3-X16C-32G** (4 Gbps, dedicated cores, ~22 spokes, +€130/mo); `default_spoke_budget_mbps` split per class (50 prod / 10 non-prod) so POP2-2C-8G stays viable for non-prod (§8). The 30–50%-of-line-rate IDS factor is still an assumption — **the Phase 1 PoC must measure it across all four dimensions** before these numbers are trusted — now five, see #18 |
| 18 | **Squid's capacity curve is not the §8 forwarding model.** A proxy terminates TCP; its constraint is concurrent connections and CPU per `CONNECT`, not PPS | Shard sizing (COMPUTE3-X16C-32G) was derived for forwarding + IDS only, so the spoke ceiling may not hold once the proxy carries real traffic | Measure `max_concurrent_proxy_connections` alongside the existing four in the Phase 1 PoC (§8); ADR-006 stays *Proposed on sizing* until it is measured. Tier-2 SNI parsing is a third, intermediate curve and needs its own figure |
| 19 | **Kapsule nodes must bootstrap through the proxy** — they need the control plane and container registry before any workload runs, and Scaleway manages part of node configuration | Nodes fail to join; takes out the Kubernetes workload scenario (§18.5) entirely | **Phase 1 gate (§19)**: prove `HTTP(S)_PROXY`/`NO_PROXY` injection reaches kubelet and containerd on a real node pool, with the control-plane and registry endpoints allowlisted, before any workload depends on it. Fall back to tier-3 IP exceptions for those endpoints only if injection proves unreliable |
| 20 | **Encrypted Client Hello (ECH) erodes SNI-based enforcement** as it deploys | Tier-2 fallback (§7.2) degrades over time; tier-1 explicit proxy is unaffected | Track adoption; review annually alongside ADR-007. Prefer tier 1 wherever a client can be made proxy-aware, so tier 2 covers a shrinking set. If ECH becomes widespread the choice narrows to explicit proxy or TLS interception — a decision that would reopen ADR-006 |

## 18. Cost model

Costs are maintained as a **formula plus a living rate sheet** (separate spreadsheet, reviewed quarterly). The euro figures below are a **dated snapshot (Sept 2026, ex-VAT, 730 h/month)** for orientation only; the living rate sheet stays authoritative (ADR-009).

### 18.1 How to read this chapter

The estate's cost is built from **four priced scenarios**, each costed once and then multiplied:

```
estate_cost = landing_zone (§18.3)                    # once, whole estate
            + Σ over workload instances ( scenario cost )   # §18.4 / §18.5 / §18.6
```

A **workload instance** is one workload deployed into one stamp (§6.1), so a workload targeting all four stamps costs four times its scenario price. Every scenario below is quoted **per instance, per month**, and *includes* the €29.20 peering baseline from §18.4 — that baseline is what a spoke costs before it runs anything, so it is never double-counted, only inherited.

### 18.2 Rate sheet (Sept 2026, ex-VAT, 730 h/month)

| Resource | Rate | ≈ €/mo | Source / confidence |
|---|---|---|---|
| **VPC peering connector** | €0.02/h | 14.60 | Network Pricing — confirmed |
| VPC, Private Networks | free | 0 | Network Pricing — confirmed (only Elastic Metal PN bandwidth tiers are charged) |
| **Kubernetes control plane (mutualized)** | free | 0 | Kapsule docs — confirmed ("provided without additional costs"); dedicated control planes are hourly with a 30-day commitment, rate not published |
| NVA — COMPUTE3-X16C-32G (prod, 4 Gbps, 16 dedicated cores) | €0.4682/h | 341.79 | Compute Pricing — confirmed |
| NVA — POP2-2C-8G (non-prod, 400 Mbps) | €0.0735/h | 53.65 | Compute Pricing — confirmed |
| Load Balancer LB-GP-M (500 Mbps) | €0.054/h | 39.42 | confirmed |
| Load Balancer LB-S (200 Mbps) | €0.023/h | 16.79 | confirmed |
| Public Gateway VPC-GW-M (1 Gbps) | €0.095/h | 69.35 | confirmed |
| Public Gateway VPC-GW-S (100 Mbps) | €0.026/h | 18.98 | confirmed |
| Flexible IPv4 (each) | €0.005/h | 3.65 | confirmed |
| DNS public zone (each) | €0.007/h | 5.11 | confirmed (5 M requests included, then €0.0005/M) |
| Instance — BASIC3-X4C-16G (4 vCPU / 16 GB, 700 Mbps) | €0.11845/h | 86.47 | confirmed — default Kapsule node |
| Instance — BASIC3-X8C-32G (8 vCPU / 32 GB, 1.5 Gbps) | €0.2368/h | 172.86 | confirmed — default general-purpose VM |
| **Block Storage** | **€0.08/GB/mo** | — | **ASSUMED — not sourced. The one unconfirmed input (open item 9)** |
| Cockpit + Object Storage floor | usage-based | ~5–10 | estimated |

> **On the block-storage assumption.** It is used for system volumes only — 50 GB per NVA, per Kapsule node and 100 GB per VM. It contributes €4–16/mo to any scenario below, so even a 100% error moves no total by more than ~€16/mo and changes no conclusion. Replace it from the Block Storage pricing page when confirming the rate sheet.

### 18.3 Scenario A — the landing zone (no workloads)

What the platform costs before a single workload exists. Shard sizes are the §8 capacity-model defaults (risks #16–#17).

| Item | Prod stamp | Non-prod stamp |
|---|---|---|
| NVA instance | 341.79 (COMPUTE3-X16C-32G) | 53.65 (POP2-2C-8G) |
| NVA system volume (50 GB) | 4.00 | 4.00 |
| Load Balancer + IPv4 | 43.07 (LB-GP-M) | 20.44 (LB-S) |
| Public Gateway + IPv4 | 73.00 (VPC-GW-M) | 22.63 (VPC-GW-S) |
| Hub VPC + 4 Private Networks | 0 | 0 |
| Cockpit / Object Storage floor | ~8 | ~5 |
| **Per stamp** | **~€470/mo** | **~€106/mo** |

| Landing zone total | €/mo |
|---|---|
| `prd01` | 469.86 |
| `acc01` + `tst01` + `dev01` (3 × non-prod) | 317.17 |
| DNS — 2 public apex zones (§11) | 10.22 |
| **Estate platform base** | **~€797/mo — ~€9.6 k/yr** |

Once the NVA is shrunk for non-prod, LB + PGW + IPs (~€43/mo) are a **floor that does not shrink**: a non-prod stamp cannot go much below ~€106/mo while keeping the full topology.

### 18.4 Scenario B — an empty workload

A registered workload with a spoke VPC and no resources in it. This is the **cost of onboarding**, and it is not zero.

| Item | Qty | €/mo |
|---|---|---|
| VPC peering connectors (spoke side + hub side, §6.6) | 2 | **29.20** |
| Spoke VPC | 1 | 0 |
| Private Networks — `pn-nodes`, `pn-app`, `pn-data` (§6.4) | 3 | 0 |
| Hub route + ingress rule + LB frontend + Let's Encrypt cert | 1 each | 0 |
| DNS records under an existing apex | — | 0 (within the zone's included requests) |
| Terraform state object, log volume | — | negligible |
| **Per instance** | | **€29.20/mo** |
| **Targeted at all four stamps** | ×4 | **€116.80/mo — €1,402/yr** |

Everything except peering is free at this scale. **An empty workload across the estate costs ~€1.4 k/yr before it runs a line of code**, which is why `stamps = [...]` in the registry (§14.2) is a cost decision and not a formality.

### 18.5 Scenario C — a workload with Kubernetes

One Kapsule cluster per instance, sized as a small production pool: **3 nodes × 4 vCPU / 16 GB**. Nodes sit in `pn-nodes` with no public IPs — egress leaves via the hub NVA and PGW (§9), so the spoke adds no gateway or IP cost. Ingress terminates on the **hub** LB (§7.3), so no per-workload Load Balancer is required.

| Item | Qty | €/mo |
|---|---|---|
| Kubernetes control plane (mutualized) | 1 | **0** |
| Node instances — BASIC3-X4C-16G | 3 | 259.41 |
| Node system volumes (50 GB each) | 150 GB | 12.00 |
| Peering baseline (§18.4) | — | 29.20 |
| **Per instance** | | **~€301/mo** |
| **Targeted at all four stamps** | ×4 | **~€1,202/mo — ~€14.4 k/yr** |

The free control plane means a Kapsule workload costs essentially its node pool. Scaling levers are node count and node size; a dedicated control plane (for regional HA, §10) is priced hourly with a 30-day commitment and is **not** costed here — confirm the rate before selecting it. Non-prod instances of the same workload would normally run a smaller pool, so ×4 is an upper bound.

### 18.6 Scenario D — a workload with 2 VMs

Two general-purpose Instances at 32 GB RAM, the ordinary "lift a server into the landing zone" shape. Same networking assumptions as §18.5 — no public IPs, no per-workload LB.

| Item | Qty | €/mo |
|---|---|---|
| Instances — BASIC3-X8C-32G (8 vCPU / 32 GB) | 2 | 345.73 |
| System volumes (100 GB each) | 200 GB | 16.00 |
| Peering baseline (§18.4) | — | 29.20 |
| **Per instance** | | **~€391/mo** |
| **Targeted at all four stamps** | ×4 | **~€1,564/mo — ~€18.8 k/yr** |

**32 GB alternatives** — the instance line dominates this scenario, so the choice is worth making deliberately:

| Type | vCPU | Bandwidth | €/mo | Note |
|---|---|---|---|---|
| BASIC2-A8C-32G | 8 | 800 Mbps | 100.59 | previous generation, cheapest |
| GP1-S | 8 | 800 Mbps | 139.24 | previous generation |
| POP2-HM-4C-32G | 4 | 400 Mbps | 150.38 | memory-optimised, fewer cores |
| PRO2-S | 8 | 1.5 Gbps | 163.07 | |
| **BASIC3-X8C-32G** | 8 | 1.5 Gbps | **172.86** | **default** |
| POP2-8C-32G | 8 | 1.6 Gbps | 211.70 | |
| STANDARD3-X8C-32G | 8 | 2 Gbps | 232.87 | highest bandwidth |

### 18.7 Worked estate example

Four stamps, ten workloads, each targeting all four — 40 spoke instances, within the ~19-per-stamp Private Network ceiling (§6.1):

| Component | Calculation | €/mo |
|---|---|---|
| Landing zone (§18.3) | once | 797.25 |
| 2 Kubernetes workloads (§18.5) | 8 instances × 300.61 | 2,404.84 |
| 1 two-VM workload (§18.6) | 4 instances × 390.93 | 1,563.71 |
| 7 light workloads (§18.4 baseline only) | 28 instances × 29.20 | 817.60 |
| **Total** | | **~€5,583/mo — ~€67 k/yr** |

Share of that total: **landing zone 14%, peering 21%, workload compute 65%.**

### 18.8 What the numbers say

- **Two different questions have two different answers.** Of the *platform* bill — landing zone plus peering, the part the Landing Zone team owns — **peering is 59–74%** (§18.3 vs §18.4). Of the *whole* bill once real workloads land, **compute is ~65%** and the landing zone is ~14%. Neither figure contradicts the other; quote the one that matches the question being asked.
- **The marginal cost of a workload is peering, then compute.** Onboarding costs €29.20/instance before anything runs (§18.4). That is small against a Kubernetes pool but large against nothing — and it accrues per stamp, so the cheapest saving available is a workload that does not need to exist in all four stamps.
- **The fixed hub base no longer dominates.** Earlier drafts concluded `fixed_hub_base × stamp_count` was the driver. At the published connector rate it is not: at four stamps the platform base is €797/mo against €1,168/mo of peering for 40 instances.
- **A shared non-prod hub is a second-order lever now.** It collapses three non-prod bases into one (~€212/mo) but leaves every spoke's two connectors intact — ~11% of the platform base, ~4% of the worked estate.

**Cost levers, re-ranked:** (1) **right-size workload compute** — node pools and instance types dominate the total; (2) **fewer spoke instances** — review each workload's `stamps` list (§14.2), €29.20/mo each; (3) **shared non-prod hub** (§10); (4) smaller non-prod NVA types and the 10 Mbps non-prod budget (§8); (5) TTL/auto-suspend on ephemeral dev stamps. The platform dashboard (§12) should report peering spend per workload and compute spend per stamp so both top levers are visible.

*(Rates sourced Sept 2026 from Scaleway Network Pricing (PAR-1), Compute Instance pricing (AMS-1) and the Kapsule documentation, ex-VAT; Block Storage is assumed pending confirmation — open item 9. Instance line rates used by §8 come from the same Compute page. Verify all against the living rate sheet.)*

## 19. Roadmap

**Phase 1 — Foundation:** landing zone repo + module skeleton, prod + staging stamps (one shard per pool), first two workloads, validation suite (§15), IAM audit, PoC gates for risks #1–#3, **the NVA stack PoC** — inspection throughput plus proxy concurrency (risks #17–#18) — and the **Kapsule-bootstrap-through-proxy gate** (risk #19).
**Phase 2 — Scale hardening:** quota confirmations, shard scale-out exercised against the capacity model, break-glass tested, dashboards + cost sheet live, first quarterly rebuild.
**Phase 3 — Enhancements:** NVA HA pattern (active/passive per shard — pulled forward now that the shard is stateful, risk #6), session-recording bastion, dev stamp, per-spoke ingress inspection, NACL GA adoption, region-resilience design study, first off-provider exit rehearsal against the validation suite (§16).

## 20. Open items

1. ~~Confirm the €/hour VPC peering connector rate~~ — **closed**: €0.02/h per connector, €29.20/mo per spoke (Sept 2026, risk #10). It exceeds the hub bases; peering is 59–74% of platform run-rate, so **peering spend per workload belongs on the platform dashboard** (§12) and each workload's `stamps` targeting is now a reviewed cost decision (§18).
2. **Request a Private Network quota increase** from Scaleway Support (default 255/Organization) — the binding estate constraint (risk #2, §6.1). Confirm the per-VPC route quota in the same request (risk #14).
3. Decide **shared non-prod hub vs. per-stamp hub** for non-prod (§10, §18) — worth ~€210/mo, but no longer the biggest lever now that peering dominates (§18).
4. ~~Confirm /16-per-stamp vs. /17 addressing~~ — **closed by ADR-015**: /12 per stamp gives 1,022 addressable spokes and 16 blocks, so addressing is no longer a limiting dimension (§6.2).
5. LB → cross-peering backend behaviour PoC (risk #3).
6. Validate NVA inspected-throughput assumptions per chosen instance type, **and the PGW/LB shard sizing against measured egress and ingress** (§8, risks #16–#17) — ratify COMPUTE3-X16C-32G, VPC-GW-M and LB-GP-M for prod, and confirm POP2-2C-8G + VPC-GW-S suffice for non-prod at the 10 Mbps budget. The 30–50%-of-line-rate IDS factor underpins every shard number and is still unmeasured.
7. Confirm Kapsule pod/service CIDR pinning against `100.64.0.0/10`.
8. Bastion access-review cadence and immutable log destination sizing.
9. **Confirm the Block Storage €/GB/month rate** — the only unsourced input in the §18 rate sheet (assumed €0.08/GB/mo for system volumes). It moves no total by more than ~€16/mo, so it is a tidiness item, not a blocker.
10. Confirm the **dedicated Kapsule control-plane** rates if regional HA (§10) is ever selected — hourly with a 30-day commitment, not published (§18.5).
11. **State the driver for egress destination control in ADR-006** — which obligation (data-exfiltration control, sovereignty, audit, NIS2) requires it. The mechanism is now specified (§7.2); the *requirement* is not, and without it the NVA's ~€346/mo and risks #6/#17–#20 are hard to defend at review.
12. Decide the **proxy-awareness baseline for base images**: which runtimes get `HTTP(S)_PROXY` injected by the platform vs. by workload teams, and what `NO_PROXY` must contain (private ranges, `100.64.0.0/10`, hub names) so tier 1 covers as much as possible (§7.2).

---

## Appendix A — Architecture Decision Records

**ADR-001 — Hub & Spoke over flat/single-VPC.** *Accepted.* Central inspection, shared ingress/egress, and per-workload isolation outweigh peering cost and hub complexity. Alternative (single VPC, PNs as spokes) rejected: insufficient blast-radius and IAM isolation at 250 workloads.

**ADR-002 — Workload teams hold no network rights.** *Accepted (amended by ADR-011/012).* Segregation of duties requires the platform to own VPC/PN/peering/DNS everywhere, enforced by omission of permission sets (no explicit deny exists) plus automated audit. Network resources now live *inside* each `spoke-<name>` repo but are applied by a **platform-controlled** identity (`app-spoke-<name>-network-<env>`) scoped to the spoke project, its paths gated to platform `CODEOWNERS` — so workload teams still hold no network rights. Exception unchanged: `PrivateNetworksReadOnly` where product APIs require attachment from workload Terraform.

**ADR-003 — Deterministic CIDR derivation.** *Accepted; addressing scheme superseded by ADR-013, then ADR-015.* The `cidrsubnet()`-from-registry-index determinism (no hand-picked addresses) stands; the original /14-per-environment allocation became /16-per-numbered-stamp (ADR-013) and is now /12-per-stamp with an allocated stamp id (ADR-015).

**ADR-004 — All traffic through the hub, strictly spoke↔hub.** *Accepted.* Full inspection of ingress and egress; no spoke↔spoke routes; consequence: transitive peering unnecessary (single-hop flows).

**ADR-005 — Pools instead of singleton hub services.** *Accepted; default now single-shard-per-stamp.* NVA/LB/PGW remain shard maps with deterministic assignment, but because each stamp is small (§6.1) the default is **one shard per pool per stamp**; the pool exists so a hot/growing stamp scales by configuration. The estate is distributed across four single-shard hubs, so the fixed hub base multiplies across stamps rather than a single pool growing to 5–10 shards (§8, §18).

**ADR-006 — NVA stack: nftables + Squid + Suricata + Unbound, over OPNsense or a managed firewall.** *Accepted in scope; **Proposed on sizing** pending the Phase 1 PoC (risks #17–#18).*

*Why an appliance at all.* Scaleway's native controls cannot express this policy. Network ACLs are documented to filter only between Private Networks of a VPC and explicitly **not** traffic to or from the internet. Public Gateways provide NAT, static NAT, DHCP, DNS and an SSH bastion — no egress filtering primitive. Instance security groups are the only native internet filter, and they are per-Instance (not Kapsule nodes, Managed Databases or serverless), IP/port only with no name awareness, attached to resources **inside workload projects** where workload teams hold write rights — and Scaleway IAM has no explicit deny (risk #5) to prevent a team widening its own egress. Scaleway offers no managed NGFW; Edge Services WAF is ingress HTTP only. Native controls are still used where they fit: NACLs for east-west tier separation (§6.4) and hub defence in depth (§6.3), the PGW for NAT, the LB for TLS termination.

*Why these components.* **nftables** for L3/L4 and anti-bypass; **Squid** as an explicit `CONNECT` proxy for per-spoke FQDN × port policy; **Suricata** for IDS and denial logging; **Unbound** scoped to the tier-2 transparent path only. All four are declarative via cloud-init/Git — OPNsense was rejected as GUI-centric and hard to reconcile with drift detection, with a larger attack surface and license/GUI exposure.

*Why no TLS interception.* Enforcement is on the `CONNECT` hostname, leaving TLS end-to-end. This avoids creating and distributing a corporate CA — a key of extraordinary value, a trust-distribution burden in every image and runtime, and custody of workload payload data the platform team should not hold. Accepted consequence: policy is expressed on names and ports, never payloads; HTTPS content inspection is out of scope.

*Consequences.* The shard becomes **stateful**, enlarging the blast radius of a single-instance failure (risk #6) and pulling NVA HA forward in the roadmap. It adds a capacity dimension the §8 forwarding model does not cover (risk #18). Applications that ignore proxy settings are handled by the tiered ladder in §7.2 — explicit proxy, then SNI enforcement without decryption, then expiry-dated IP exceptions — with DNS-derived dynamic IP rules explicitly rejected as fragile against CDNs. ECH will erode tier 2 over time (risk #20). Kapsule bootstrap through the proxy is a Phase 1 gate (risk #19).

*Still open.* The obligation that requires egress destination control is not stated anywhere in this document (open item 11); this ADR records *how*, not *why*.

**ADR-007 — IPv6: v4-only edges, v6 filtered internally.** *Accepted.* PGWs lack IPv6; auto-assigned per-PN /64s are immutable, so v6 is explicitly filtered (mirrored NACLs, NVA v6 drop) rather than ignored. NAT64/DNS64/dual-stack deferred; revisit yearly.

**ADR-008 — Transitive peering enabled on hubs at creation.** *Accepted.* Not functionally needed (ADR-004), but the flag is immutable post-creation; enabling preserves future options at zero current cost.

**ADR-009 — Costs as formula + living rate sheet.** *Accepted.* Point-in-time prices in design documents go stale within a quarter; the model (§18) stays valid, the rate sheet stays current.

**ADR-010 — Reversible lock-in over cloud-agnostic abstraction.** *Accepted.* The portability goal is **exit capability** — a bounded-time rebuild on another provider — not active multi-cloud and not a lowest-common-denominator abstraction layer. The estate stays provider-native (managed LB, Kapsule, Managed DB, Cockpit) and pays no continuous portability tax; in exchange, every dependency must be *reversible* — portable state export **plus** a single replaceable adapter module (§16.2) — and the exit is rehearsed off-provider at least annually against the validation suite (§16.6). One-way-door services (no export or no equivalent) require an exit-impact ADR. Rejected alternatives: a cloud-agnostic abstraction (forfeits provider-native strengths, pays a standing tax for a capability rarely exercised) and self-hosting everything (ops burden without commensurate benefit where managed services have tested exports). The managed LB pool is retained under this ADR; replacing it with a self-managed reverse proxy is a possible future adapter-surface reduction, not a requirement.

**ADR-011 — Repo and Terraform state per workload; platform as repo-factory.** *Accepted.* The global layer is one `platform` repo (foundation + hub + versioned modules + registry + policies + repo-factory); each workload is its own `spoke-<name>` repo with its own state (network state + app state). State per spoke gives per-workload blast radius and lets onboarding be a single spoke apply; repo per spoke adds independent CI, access control, and a clean future path to fully separate ownership. The platform pipeline gains **GitHub org-admin** to mint and govern spoke repos as code. Rejected: one monolithic per-environment state (multi-thousand-resource plans, one lock serialises all onboarding) and Terragrunt (unneeded once each spoke is a single small root; the versioned module library is the DRY mechanism). Cost: ~one repo per workload to govern — handled by the repo-factory — and module-versioning discipline.

**ADR-012 — Networking lives in the spoke repo; two-sided peering is the duty boundary.** *Accepted.* Scaleway peering requires a connector in **each** VPC and only reaches `Peered` when both exist. The spoke repo (platform-controlled network identity, scoped to the spoke project only) builds the spoke-side connector, PNs, NACL and default route; the platform hub layer builds the hub-side connector, route and ingress rule (`for_each` over the registry). Neither identity needs rights in the other's VPC, and a spoke cannot self-connect — segregation of duties enforced by the platform's own topology rather than by permission bookkeeping. Cost: onboarding is a coordinated two-step (registry PR + spoke apply) and hub-side per-spoke resources touch the shared hub state at lifecycle events only (safe `for_each` additions; shard the hub state per stamp if it ever grows unwieldy).

**ADR-013 — Instance-numbered stamps; 250 as a global spoke-instance budget; /16 per stamp.** *Accepted for instance numbering; the /16 addressing and the 250 budget are superseded by ADR-015.* Environments are numbered instances `<class><NN>` (two-digit, 01–99 per class). The 250 target is the **estate-wide** count of spoke instances (~20 workloads × their targeted stamps), not 250-per-environment; each stamp is small (≤~20 in practice, ≤60 by addressing). Each stamp gets one /16 with the second octet as the stamp id, making addressing self-identifying and non-overlapping within 10/8; class ranges prd `10.0–10.31`, stg `10.32–10.63`, dev `10.64–10.127`, reserved `10.128+`. Because stamps are independent (ADR-004), overlap is safe if ever needed; a /17-per-stamp or non-prod overlap covers the full 99×3 ceiling. This reframing also relieves the old per-hub quota risks (≤~60 per hub) and inverts the cost driver to fixed-base-×-stamp-count (§18).

**ADR-014 — Single manual bootstrap seam.** *Accepted.* Full IaC has one unavoidable chicken-and-egg: the Terraform state bucket and the first platform pipeline key must exist before anything can be managed as code. These two artefacts are created **manually once** and documented; everything downstream — projects, all IAM applications and keys, the repo-factory, every spoke repo and credential, and ≤90-day key rotation — is IaC. The manual step is exactly one bucket + one key; per-spoke credentials are never hand-made.

**ADR-015 — Four named stamps; /12 per stamp; hub /21; quota as the real ceiling.** *Accepted (supersedes ADR-013 addressing and the 250 budget).* The estate is **four allocated stamps** — `prd01`, `acc01`, `tst01`, `dev01` — with **three reserve /12 blocks** and nine unallocated. Instance numbering from ADR-013 is **retained** so a second instance (`dev02`) is an allocation from a reserve block rather than an estate-wide rename.

*Addressing.* Each stamp receives one **/12**; `10.0.0.0/8` divides into exactly 16, and the stamp id is the high nibble of the second octet (`S = octet2 >> 4`), so any address still names its stamp in one shift. The hub takes the first **/21** — the tightest prefix that fits the five §6.3 Private Networks (1,280 of 2,048 addresses used) — and spokes are `cidrsubnet("10.${S*16}.0.0/12", 10, w + 2)`, i.e. /22-blocks 2…1023.

*Why /12 and not /16.* A /16 held 60 spokes and, at a /20 hub, could not be stretched past 62 — a ceiling set by arithmetic rather than by anything operational. A /12 holds 1,022, which deliberately removes addressing from the list of constraints so that capacity discussions are about quota and inspection throughput, where the real limits are. The cost is coarser blocks (16 rather than 256), which is ample against an estate of four.

*Stamp ids are allocated, not derived.* ADR-013 computed `S` from `class_base + (NN − 1)`. With 16 blocks and 7 in use, an explicit registry allocation is clearer and lets `prd02` take a low block. Everything below the stamp stays formula-derived — no address is hand-picked (principle #5).

*The 250 budget is replaced.* Published Scaleway quotas (Sept 2026) show the binding constraint is **255 Private Networks per Organization**: 4 per hub, 3 per spoke, giving `Σ over stamps (4 + 3 × spokes) ≤ 255` — about 19 spokes per stamp at four stamps. Peering was never the limit: there is no per-VPC peering cap and the Organization cap is 512 connectors. The CI invariant therefore asserts the PN budget, not a spoke-instance count (§15).
