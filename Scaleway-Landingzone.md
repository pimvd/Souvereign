# Scaleway Landing Zone — Hub & Spoke Design

| | |
|---|---|
| **Status** | Draft v1.3 |
| **Date** | 3 August 2026 |
| **Owner** | Pim van Dijk |
| **Scope** | Network, identity, delivery, and operations architecture for a multi-environment, multi-workload Scaleway estate |
| **Changes in v1.1** | Architecture diagrams; capacity planning; availability assumptions; observability; security controls; Terraform module layout; validation; ADR appendix; formula-based cost model |
| **Changes in v1.2** | **Fix: spoke CIDR derivation overlapped hub /20 (now `n + 4`, spokes from 10.e.16.0/22)**; hub internal PN layout (§6.3); corrected NVA capacity model (planning formula + validation invariant, 4 dimensions); validation split into pre-/post-apply with rollback classes and Phase 1 LB acceptance gate |
| **Changes in v1.3** | New **Exit strategy** chapter (§16): reversible-lock-in principle (ADR-010), exit register, data-export requirement for stateful services, off-provider rebuild rehearsal and a time-to-exit SLO. Provider-neutral; managed LB pool retained unchanged. Subsequent sections renumbered §17–§20 |
| **Changes in v1.4** | **Delivery model reshaped**: global layer renamed `platform`; **repo + Terraform state per workload** (spoke), networking now lives in the spoke repo under a platform-controlled identity, **two-sided peering as the duty boundary** (§6.6, §14, ADR-011/012). **Environments are instance-numbered stamps** `<class><NN>` and the **250 target is now a global spoke-instance budget**, not per-environment; small stamps → **one /16 per stamp** addressing (§6.1, §6.2, ADR-013). Capacity reworked to single-shard-per-stamp (§8). **Cost model rebuilt with a dated euro snapshot** and the inverted "fixed base × stamp count" driver (§18). Multi-domain DNS made explicit (§11). Single manual bootstrap seam (§14.4, ADR-014). Managed LB retained |

---

## 1. Purpose and scope

This document describes an enterprise-grade landing zone on Scaleway built around a hub-and-spoke network topology. It defines the environment model, the CIDR plan, the identity and segregation-of-duties model, the traffic flows, capacity and availability assumptions, and the delivery (IaC) model. It is designed to scale to **up to ~250 spoke *instances* across the whole estate** — on the order of ~20 workloads, each instantiated across many small, instance-numbered environment stamps — while keeping every network and identity decision under control of a central Platform (Landing Zone) team. Each individual stamp is small (≤~20 spokes in practice, ≤60 by addressing); the 250 is a **global budget**, not a per-environment one (§6.1, §6.2).

Out of scope for v1.0/v1.1: cross-region connectivity, hybrid/on-premises connectivity, session-recording bastion, managed SIEM integration. Availability assumptions and the path toward region resilience are documented in §10.

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
              |         | nva-0 [.. n]    |  (inspection, allowlist)
              |         +--------+--------+
              |                  ^     (1 shard/pool per small stamp;
              |                  |      pool scales out only on demand)
   +----------+------------------+----------------------------+
   |            HUB VPC  10.S.0.0/20  (one numbered stamp)     |
   |          (routes, ingress rules, private DNS)             |
   +---+--------------+--------------+---------------------+---+
       |peering       |peering       |peering              |peering
   +---v----+     +---v----+     +---v----+            +---v----+
   | Spoke 0|     | Spoke 1|     | Spoke 2|   . . .    |Spoke ~19|
   |  /22   |     |  /22   |     |  /22   |            |  /22    |
   +--------+     +--------+     +--------+            +---------+

   Spoke <-> Spoke traffic: none (no routes, denied by policy)

   Estate = many small numbered stamps (prd01; stg01..stg08; dev01..dev05),
   ≤250 spoke instances TOTAL across all of them (§6.1). Each stamp = one /16.
```

### 2.2 Organization and project hierarchy

```
Scaleway Organization
|
+-- plt-connectivity-prd01      (Hub VPC, pools, DNS, hub-side peering)
+-- plt-management-prd01        (Cockpit, audit, TF state, runners)
+-- wl-amazingapp-prd01         (Spoke VPC + workload resources)
+-- wl-<workload>-prd01         ... ≤ ~20 workloads per stamp
|
+-- plt-*/wl-*-stg01 ... stg08  (8 staging stamps)
+-- plt-*/wl-*-dev01 ... dev05  (5 dev stamps)
|
Numbered stamps: <class><NN>, two digits, 01–99 per class.
≤250 spoke INSTANCES total across all stamps (§6.1).

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
+------------ Spoke VPC (10.S.x.0/22 — one workload, one stamp) -----+
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
3. **Environment stamps.** An environment is a complete, independent copy of the architecture, addressed as a **numbered instance** (`prd01`, `stg03`, `dev01`; two digits, up to 99 per class). No resource, network, or peering is shared between stamps. The estate runs many small stamps; the 250 target is a *global* spoke-instance budget across them (§6.1).
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

Here `<env>` is a **numbered stamp**: `<class><NN>`, two-digit zero-padded, `01`–`99` per class (`prd01`, `stg03`, `dev01`); classes are `prd`/`stg`/`dev` (extensible). So projects read `plt-connectivity-stg03`, `wl-amazingapp-prd01`. Workload names are short kebab-case identifiers registered in the workload registry (§14.2). Each workload additionally has its own Git repository (§14.1); the platform pipeline holds **GitHub organisation-admin** rights to create and govern those repositories as code (the "repo-factory").

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

Each environment is a **numbered stamp** (`<class><NN>`) consisting of one hub VPC and a small number of spoke VPCs (≤~20 in practice, ≤60 by addressing), joined by VPC peering in a star topology. The estate runs **many small stamps** — currently ~5 dev (`dev01`–`dev05`), 8 staging (`stg01`–`stg08`), and one or more prod — with a **global budget of ≤250 spoke instances** across all of them (a spoke *instance* = one workload deployed into one stamp). So ~20 workloads × their targeted stamps ≈ 250 instances; no single stamp holds 250. Traffic patterns are strictly spoke↔hub; because both egress (spoke → hub NVA → PGW) and ingress (hub LB → spoke) are single peering hops, transitive peering is not functionally required. It is nevertheless enabled on hub VPCs at creation time as insurance, because the setting is immutable after creation (ADR-008).

### 6.2 CIDR plan

Requirements: /22 per spoke, /20 per hub, **small stamps** (≤~60 spokes), non-overlapping and human-legible addressing, deterministic derivation. Because each numbered stamp is small (§6.1), each receives **one /16** — the second octet *is* the stamp id (`10.S.0.0/16`), which makes every log line, route, and NACL rule self-identifying (ADR-013, supersedes the old /14-per-environment scheme).

| Class | Second-octet range (stamp id `S`) | Stamps | Example |
|---|---|---|---|
| prd | `10.0` – `10.31` | up to 32 | `prd01 = 10.0.0.0/16` |
| stg | `10.32` – `10.63` | up to 32 | `stg01 = 10.32.0.0/16` |
| dev | `10.64` – `10.127` | up to 64 | `dev01 = 10.64.0.0/16` |
| reserved | `10.128` – `10.255` | 128 | growth / second region / larger allocations |

Stamp id is derived, not chosen: `S = class_base + (NN − 1)` (`prd01 → 10.0`, `stg01 → 10.32`, `dev01 → 10.64`). Within a stamp the hub takes the first /20 (`10.S.0.0/20`); workload *w* (0-based registry index) receives `cidrsubnet("10.S.0.0/16", 6, w + 4)` — workload 0 = `10.S.16.0/22`, up to workload 59 = /22-block 63. CI invariants assert no generated spoke CIDR overlaps its hub /20 **and** no two stamps' /16s overlap (§15).

Because stamps are fully independent (ADR-004, principle #3), non-overlap is a convenience (clean logging, future optionality), not a hard requirement. The /16-per-stamp scheme caps the estate at 256 stamps; the two-digit format allows up to 99 per class (297 total), so to exceed 256 either narrow to a /17 per stamp (512 fit) or let **non-prod** stamps overlap (safe, since nothing routes between stamps). Prod always keeps unique, non-overlapping /16s. Kapsule pod/service CIDRs remain pinned in `100.64.0.0/10` (§9), disjoint from the `10.0.0.0/8` plan.

### 6.3 Hub internal layout

The hub /20 is itself carved into Private Networks so that next-hop placement, NVA↔PGW wiring, and runner placement are explicit rather than implied:

| Hub Private Network | CIDR (offset within /20) | Purpose |
|---|---|---|
| `pn-hub-transit` | first /23 | NVA shard interfaces receiving spoke traffic; target of hub ingress rules (next-hop IPs live here) |
| `pn-hub-egress` | next /24 | NVA ↔ PGW leg; PGWs attach here and NAT the NVA-forwarded traffic |
| `pn-hub-ingress` | next /24 | Hub LB shards; backends reach spokes via peering |
| `pn-hub-management` | next /24 | Validation runners (§15), monitoring collectors, bastion-adjacent tooling |
| reserved | remaining /23 | Pool growth, future HA legs |

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

- the **spoke repo** (network path, identity `app-spoke-<name>-network-<env>`, scoped to the spoke project only) creates the **spoke-side** connector, the spoke default route `0.0.0.0/0` toward it, and the route to the hub /20;
- the **platform hub layer** (`app-platform-pipeline`) creates the **hub-side** connector, one hub route to the spoke /22, and one ingress rule directing inbound-from-spoke traffic to the assigned NVA shard (§7) — all generated by `for_each` over the registered spokes targeting that stamp.

Neither identity holds rights in the other's VPC, so no principal needs cross-VPC permissions and a spoke cannot self-connect to the hub. Onboarding is therefore a coordinated two-step that "meets in the middle": a registry PR provisions the hub side, the spoke repo provisions its side, and peering comes up when both halves land. Routine spoke changes never touch the hub state; only onboarding/offboarding/shard-reassignment do (a `for_each` addition, safe). A hub carries ≤~60 routes/ingress-rules/connectors per stamp — well within any plausible quota, so the old "250+ routes / 500 connectors per env" concern is relieved by small stamps; still confirm limits before growing a *single* stamp past ~50 spokes (§17).

## 7. Shared resource pools

Hub services that process per-spoke traffic are modeled as **pools of identical shards** rather than singletons (ADR-005). The pattern applies to the NVA, LB, and PGW tiers and is reusable for future shared services.

### 7.1 The pool pattern

A pool is a map of shard definitions in the environment configuration:

```hcl
nva_pool = {
  "nva-0" = { instance_type = "POP2-8C-32G", zone = "nl-ams-1" }
  "nva-1" = { instance_type = "POP2-8C-32G", zone = "nl-ams-2" }
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

Because a stamp is small (≤~20 spokes, §6.1), the **default is a single shard per pool per stamp**; the pool machinery exists so a hot or growing stamp scales by configuration, but in practice it rarely engages within one stamp. The 250-instance estate total is spread across many single-shard hubs, not concentrated in one large pool (ADR-005).

### 7.2 NVA pool (egress and inspection)

Each NVA shard is an instance running the inspection stack (§ADR-006) with configuration fully declared in the repository and applied via cloud-init — an NVA is disposable and recreatable by pipeline in minutes. Per-spoke hub ingress rules direct each spoke's outbound traffic to its assigned shard; each shard egresses through an assigned PGW. Day-one deployment is a single shard; the sharding mechanics exist from the first apply. The inspection ruleset includes a maintained, code-reviewed **domain allowlist** covering at minimum: Kapsule control-plane endpoints, Scaleway container registry, OS package mirrors, GitHub, and per-workload approved destinations (requested via PR).

Accepted risk: shards are individually single instances (by requirement). Mitigations: health checks with alerting, immutable configuration, pipeline-driven recreate, per-shard blast radius.

### 7.3 Load balancer pool (ingress)

All inbound traffic enters via hub LBs, sharded by spoke assignment; DNS is the distribution layer (each workload hostname resolves to its shard's IP). Backends target private IPs in the spoke across the peering. Certificates (Let's Encrypt via LB, or imported) are per-frontend and landing-zone-owned.

### 7.4 Public Gateway pool

PGWs provide NAT for the NVA shards and host the built-in SSH bastion. The pool is typically small but modeled identically, so throughput or bastion-isolation needs are met by adding shards. Bastion access lists are landing-zone-managed; workload teams request access via PR.

## 8. Capacity planning

Sharding is only meaningful with numbers behind it. Capacity is planned per shard type using the formulas below; the concrete values marked *(validate)* are established in the Phase 1 proof of concept and re-measured after any instance-type change.

**NVA shard.** Effective capacity is the minimum of (a) instance network bandwidth — note Scaleway applies the bandwidth limit *per network connection*, so PN-facing and internet-facing capacity are separate budgets — and (b) inspection throughput, typically 30–50% of line rate with IDS enabled *(validate)*. Capacity is expressed in four dimensions, whichever binds first: `max_throughput_mbps`, `max_concurrent_sessions` (conntrack), `max_new_connections_per_second`, and `max_packets_per_second` — for Suricata, PPS is often the binding constraint before bandwidth *(validate all four in PoC)*.

Two separate models are used. **Planning** (default shard assignment, uniform budgets):

```
spokes_per_nva = floor( shard_inspected_capacity_mbps × headroom_factor
                        / default_spoke_budget_mbps )
```

**Validation** (CI invariant per shard, honoring per-spoke overrides — checked on every registry change):

```
sum(egress_budget_mbps of assigned spokes) <= shard_inspected_capacity_mbps × 0.70
```

and equivalents for the session/CPS/PPS dimensions. With a shard validated at 2,000 Mbps inspected and 70% usable: `floor(2000 × 0.70 / 50) = 28` spokes at the default budget. Because each stamp hosts only ≤~20 spokes (§6.1), **a single NVA shard covers a whole stamp** at the default budget; the pool scales out only for an unusually hot stamp, and measured utilization drives that decision (§12). The 250-instance estate total is distributed across many single-shard hubs, so the old "one big pool of 5–10 shards" picture does not apply — instead the fixed hub base is multiplied across stamps, which is the dominant cost (§18).

**LB shard.** Bounded by frontends/certificates per LB and connections/throughput per LB type. A single LB shard covers a small stamp; default `spokes_per_lb = 50` *(validate against LB type limits)*. Multi-domain workloads (§11) consume extra per-frontend certificates, which counts against the LB cert limit — so effective `spokes_per_lb` drops for cert-heavy stamps.

**PGW shard.** Bounded by NAT throughput and session table. One PGW per stamp is the default; scale to one PGW per 2–3 NVA shards only where a stamp is scaled out *(validate)*.

**Hub control plane.** Per stamp the hub carries only ≤~60 routes/ingress-rules/connectors — comfortably within plausible quotas. Confirm limits with Scaleway only before growing a *single* stamp past ~50 spokes; the org-wide count of stamps/VPCs and public IPs is the newer quota dimension to watch (§17).

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

**Cost-driven variant — shared non-prod hub.** Because every stamp carries a full always-on hub base (§18), and the estate runs many small non-prod stamps, non-prod numbered stamps *may* share one hub — several `dev*`/`stg*` spokes peering into a single shared hub — to collapse N hub bases into one, trading inter-stamp isolation for cost. **Prod stamps are never shared.** Ephemeral dev stamps additionally support a TTL/auto-suspend profile. This is the single biggest cost lever (§18) and is a supported deployment shape, selected by a per-stamp flag rather than a redesign.

## 11. DNS and domains

Public zones are owned by the platform in `plt-connectivity-<env>` (or a shared DNS project where zones span stamps). The estate can host **multiple apex domains** — e.g. `example.com` and `example.nl` — each apex being its own platform-owned public zone; the managed set is declared explicitly as `dns_zones` in the registry (§14.2), and the platform pipeline holds `DomainsDNSFullAccess` across all of them. Workloads receive names under any managed apex (`<workload>.<env>.example.com`, a vanity host, or several domains at once) via records requested by PR. Each published hostname is a landing-zone-owned LB frontend with its own certificate (Let's Encrypt via LB, or imported); publishing on multiple domains therefore consumes more per-frontend certificates, which counts against LB cert limits (§8). Private resolution is strictly spoke↔hub: spokes resolve hub-published service names via Scaleway's built-in private DNS per Private Network; no cross-spoke discovery exists by design.

## 12. Observability and logging

All telemetry converges in `plt-management-<env>` (Cockpit), with prod audit data additionally protected against tampering.

**Log routing.** NVA shards ship flow logs, IDS alerts, and allowlist denials; PGWs ship NAT/bastion session logs; LBs ship access logs; Kapsule and instances ship via the standard agents into the workload's Cockpit scope with copies of security-relevant streams to the platform scope. Scaleway audit trail (IAM changes, resource mutations) is exported continuously.

**Immutable audit.** Audit and bastion logs are additionally written to an Object Storage bucket with versioning and a compliance-style retention lock; the platform pipeline itself cannot delete within the retention window.

**Retention.** Defaults: metrics 13 months, application logs 30 days, security/flow logs 90 days, audit logs 400 days (immutable bucket). Per-workload overrides via the registry.

**Alerting.** Platform alert set: NVA/PGW/LB shard health, drift detection findings, IAM audit diffs, certificate expiry, peering/route validation failures, capacity thresholds (≥70% of any shard budget), and budget anomalies. Routed to the platform team's on-call channel.

**Dashboards.** One platform dashboard per stamp (rolled up across the estate): shard utilization vs. capacity model (§8), spoke count vs. quota headroom, per-stamp and total fixed-base cost run-rate (§18), top egress destinations, denied-flow trends.

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
│   ├── stg01/ … stg08/
│   └── dev01/ … dev05/
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

Because each spoke is its own repo with a single small root per stamp, there is **no "250 roots in one repo" problem and no Terragrunt is required**; the DRY mechanism is the versioned module library that every spoke pins. Module upgrades roll out per-spoke (bump the pinned tag), so blast radius stays per-workload.

### 14.2 Workload registry

One record per workload, single source of truth, kept **central in `platform`**:

```hcl
dns_zones = ["example.com", "example.nl"]   # platform-owned public apexes (§11)

workloads = {
  "amazingapp" = {
    index            = 0        # 0-based; drives per-stamp CIDR (§6.2)
    stamps           = ["prd01", "stg01", "dev01", "dev02"]   # numbered instances
    nva_shard        = null     # null = deterministic default (§7.1)
    lb_shard         = null
    egress_budget    = 50       # Mbps, feeds capacity model (§8)
    ingress = [
      { host = "api.amazingapp.example.com", backend_pn = "pn-app", port = 443 },
      { host = "api.amazingapp.example.nl",  backend_pn = "pn-app", port = 443 },
    ]
    egress_allowlist = ["api.anthropic.com"]
  }
  # … on the order of ~20 workloads; Σ (per-workload stamp counts) ≤ 250 instances
}
```

The registry stays central because the **cross-stamp invariants** — unique `index`, no CIDR overlap, shard-capacity sums, ≤250 total instances — can only be checked where every workload is visible (§15). A workload's own repo carries only the *workload-owned* fields (app config, requested hosts, egress allowlist) that flow into the registry by PR.

Onboarding a workload is a coordinated two-step: **(1)** a registry PR in `platform` — the pipeline provisions the hub-side wiring for each targeted stamp and mints the `spoke-<name>` repo (project, IAM applications + keys, branch protection, `CODEOWNERS`, environments) via the repo-factory; **(2)** the new spoke repo's own `network` then `app` applies build the spoke side and the workload. Validation (§15) runs after each.

### 14.3 Pipelines and authentication

GitHub Actions is the only write path. Each spoke repo holds two protected GitHub environments: **network** (key = `app-spoke-<name>-network-<env>`, scoped to the spoke project, its paths gated to platform `CODEOWNERS`) and **app** (key = `app-wl-<name>-pipeline-<env>`). Scaleway authentication uses IAM application API keys as GitHub environment secrets (no OIDC federation available); issuance and ≤90-day rotation per §5.3. Terraform state lives in Object Storage in `plt-management-*` with locking — **per stamp** for the platform layer and **per spoke per stamp** for spoke repos. Plans post to PRs; applies run only from protected default branches in protected environments.

### 14.4 Bootstrap (the one manual seam)

Everything is IaC except a single root seam, created **manually once** and documented: (1) the Object Storage bucket that holds Terraform state, and (2) the first `app-platform-pipeline` API key. From there the platform pipeline is self-hosting — it creates projects, IAM applications and their keys, the repo-factory, every spoke repo, and rotates all keys (§5.3). No per-spoke credential is ever created by hand; the manual step is exactly **one bucket + one key**.

### 14.5 Cross-repo contract

Spoke repos never read platform state directly (SoD). The platform hub layer **publishes** a small read-only contract per stamp — hub VPC/PN IDs, each workload's assigned CIDR and shard, DNS zone IDs — to a well-known location (a published-outputs object / parameter store) that spoke repos consume via a scoped data source. The reverse direction (a spoke requesting a host, an allowlist entry, bastion access, or a shard move) is always a PR into the `platform` registry. No shared **mutable** state crosses the boundary; the contract is versioned alongside the module release the spoke pins (§14.1).

## 15. Validation and testing

Validation is split into **pre-apply gates** (which genuinely block a change) and **post-apply verification** (a change that already landed cannot be "blocked" — it triggers a defined response).

**Pre-apply (blocking, in CI on every PR):** Terraform validate/plan review; policy-as-code suite (§13); generated-route invariants — exactly one hub route per registered spoke, no route to unregistered CIDRs, **no spoke CIDR overlapping the hub /20**, **no two stamps' /16s overlapping** (§6.2); IAM permission diff against the §5.1 matrix; registry consistency (unique indices, every workload's `stamps` reference a defined stamp, **Σ spoke instances ≤ 250**, budgets within shard validation invariants §8). These cross-stamp checks run in `platform`, where every workload is visible (§14.2); the conformance suite below then runs per stamp and per new spoke.

**Post-apply (verification, from runners in `pn-hub-management` plus a canary per new spoke):**

1. **Peering** — all connectors `Peered`; none `Orphan`/`Conflict`.
2. **Routing** — spoke default route resolves via the correct connector; hub ingress rule delivers to the assigned NVA on `pn-hub-transit`.
3. **Filtering** — canary in `pn-app` reaches `pn-data` on allowed ports and not on denied ones; inter-spoke probe fails; IPv6 probe fails per ADR-007.
4. **Egress** — allowlisted domain reachable via assigned NVA shard; non-allowlisted probe denied and logged; PGW sees the NVA egress-leg source IP (§6.3).
5. **Ingress** — synthetic request to the workload hostname healthy through the assigned LB shard; certificate valid ≥30 days.
6. **DNS** — public records match registry; private hub names resolve from the spoke.

**Failure response per change class** (declared per module, since automatic rollback of routing/NACL changes can itself cause harm):

| Class | Examples | On post-apply failure |
|---|---|---|
| `rollback-safe` | LB frontends, DNS records, allowlist entries | Automatic revert to previous state |
| `manual-recovery-required` | Routes, ingress rules, NACLs | Incident + promotion block; guided runbook, no auto-revert |
| `forward-fix-only` | IAM policies, project/VPC creation | Incident + promotion block; fix rolls forward |

Promotion (stg → prd) requires a green post-apply suite in staging. The same suite runs nightly as continuous verification, and the quarterly dev stamp-rebuild (§10) uses it as its acceptance gate.

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
| 1 | Connectors / routes / ingress-rules per hub VPC | **Relieved** by small stamps (≤~60 per hub, not 500/250) | Confirm only before growing a *single* stamp past ~50 spokes |
| 2 | Org-wide quota: number of stamps/VPCs, projects, and public IPs at up to ~99 stamps/class | Blocks scale-out | New dimension to confirm with Scaleway before wave 2 |
| 3 | LB backend health checks across peering unproven at scale | Ingress design risk | Phase 1 PoC gate |
| 4 | NACL in Public Beta, API-only | Feature risk | Acceptable (IaC-only estate); track GA |
| 5 | No explicit deny in IAM | Segregation by omission | Automated IAM audit (§13) |
| 6 | NVA shard is a single instance | Egress outage for assigned spokes | Health checks, pipeline recreate, per-shard blast radius; HA in roadmap |
| 7 | No OIDC federation for GitHub | Long-lived keys | Automated ≤90-day rotation, least-privilege per repo |
| 8 | Immutable IPv6 /64 per PN | Unfiltered v6 path if ignored | ADR-007: parallel v6 filtering, validated in §15.3 |
| 9 | Region loss = environment loss | Availability | Accepted (§10); quarterly rebuild exercise; region resilience on roadmap |
| 10 | VPC peering connector €/hour rate **not publicly documented** | Unknown standing cost, possibly ≈ hub bases | **Top pricing question** — confirm €/connector before scale-out (§18, open items) |
| 11 | Exit capability asserted but unproven until first off-provider rebuild | Strategic / lock-in | §16: exit register + annual off-provider rebuild, validation suite as acceptance gate |
| 12 | Fixed hub base × stamp count is the dominant cost, multiplied across many small stamps | Standing cost | Shared non-prod hub + TTL/auto-suspend; smaller non-prod instance types (§10, §18) |

## 18. Cost model

Costs are maintained as a **formula plus a living rate sheet** (separate spreadsheet, reviewed quarterly). The euro figures below are a **dated snapshot (Aug 2026, ex-VAT)** for orientation only; the living rate sheet stays authoritative (ADR-009).

**Cost shape.** Each numbered stamp carries a fixed, always-on hub base plus a small per-spoke variable; the estate cost is the sum over stamps — so the fixed base **multiplies by stamp count**:

```
stamp_cost  = nva_shard(instance_rate)                        # ~1 shard/stamp (§8)
            + lb_shard(lb_rate) + pgw_shard(pgw_rate)
            + public_ips + cockpit_floor + object_storage_floor   # fixed base
            + spokes × 2 × connector_hourly_rate                  # per-spoke
            + spokes × marginal(certs, logs)
estate_cost = Σ over stamps ( stamp_cost )
```

**Snapshot rate sheet (Aug 2026, ex-VAT, ~730 h/month):**

| Resource | Rate | ≈ €/mo | Confidence |
|---|---|---|---|
| NVA — POP2-8C-32G (prod) | €0.29/h | 208.80 | confirmed |
| NVA — POP2-2C-8G (non-prod) | ~€0.0725/h | ~52.90 | estimated (linear POP2) |
| Load Balancer LB-S | — | 19.02 | confirmed |
| Public Gateway VPC-GW-S | from €0.0199/h | ~14.53 | confirmed ("from") |
| Flexible IPv4 (each) | €0.005/h | ~3.65 | confirmed (rose Jun 2026) |
| VPC + Private Networks | free | 0 | confirmed |
| VPC peering connector | hourly, split both sides | **unpublished** | **confirm — #1 pricing question (risk #10)** |
| Cockpit + Object Storage floor | usage-based | ~5–10 | estimated |

**Costed stamp (fixed base):**

| Item | Prod stamp | Non-prod stamp |
|---|---|---|
| NVA (+ block storage) | ~213 | ~53 |
| LB-S + IPv4 | 22.6 | 22.6 |
| Public Gateway + IPv4 | 18.2 | 18.2 |
| Cockpit / Object Storage floor | ~8 | ~5 |
| **Fixed base / stamp** | **~€261/mo** | **~€100/mo** |

Once the NVA is shrunk for non-prod, LB + PGW + IPs (~€45/mo) are a **floor that does not shrink** — a non-prod stamp cannot go much below ~€100/mo while keeping the full topology.

**Estate projection (platform only — excludes workload compute):**

| Stamps | Fixed hub bases | + peering (assumed €0.005/h) | ≈ annual |
|---|---|---|---|
| 14 (1 prd + 13 non-prod, today) | ~€1,600/mo | +~€2 k/mo | **~€22–43 k/yr** |
| 50 | ~€5.5 k/mo | + peering | **~€70–90 k/yr** |
| 297 (99×3 max) | ~€30 k/mo | + peering | **~€360 k+/yr** |

Two consequences the numbers make visible:

- **The dominant driver inverted.** With small stamps, `fixed_hub_base × stamp_count` dominates — not connectors × 250-in-one-env as in earlier drafts. Every non-prod stamp lights another ~€100/mo of always-on NVA+LB+PGW meters.
- **Peering is the biggest unknown.** At the 250-instance cap that is 500 connectors; at the assumed €0.005/h ≈ €1.8 k/mo, but the true rate is unpublished — confirm it first (open item 1, risk #10).

**Cost levers:** a **shared non-prod hub** (§10) collapses ~13 non-prod bases toward 1–2 — the single biggest saving; smaller non-prod instance types; TTL/auto-suspend on ephemeral dev stamps. Workload compute (Kapsule nodes, Managed DB — a DEV Postgres ~€11/mo) sits in the **workload budget** and typically dwarfs the platform base, but is per-workload, not landing-zone. *(Rates sourced Aug 2026 from Scaleway pricing/docs and cached aggregators; verify against the living rate sheet.)*

## 19. Roadmap

**Phase 1 — Foundation:** landing zone repo + module skeleton, prod + staging stamps (one shard per pool), first two workloads, validation suite (§15), IAM audit, PoC gates for risks #1–#3.
**Phase 2 — Scale hardening:** quota confirmations, shard scale-out exercised against the capacity model, break-glass tested, dashboards + cost sheet live, first quarterly rebuild.
**Phase 3 — Enhancements:** NVA HA pattern (active/passive per shard), session-recording bastion, dev stamp, per-spoke ingress inspection, NACL GA adoption, region-resilience design study, first off-provider exit rehearsal against the validation suite (§16).

## 20. Open items

1. **Confirm the €/hour VPC peering connector rate** — the top pricing unknown (§18, risk #10); it may rival the hub bases.
2. Scaleway quota confirmation, org-wide dimension: number of stamps/VPCs, projects, and public IPs at up to ~99 stamps/class (risk #2).
3. Decide **shared non-prod hub vs. per-stamp hub** for non-prod (§10, §18) — the biggest cost lever.
4. Confirm **/16-per-stamp vs. /17** addressing if more than 32 prd/stg or 64 dev stamps are ever needed (§6.2).
5. LB → cross-peering backend behaviour PoC (risk #3).
6. Validate NVA inspected-throughput assumptions per chosen instance type (§8).
7. Confirm Kapsule pod/service CIDR pinning against `100.64.0.0/10`.
8. Bastion access-review cadence and immutable log destination sizing.

---

## Appendix A — Architecture Decision Records

**ADR-001 — Hub & Spoke over flat/single-VPC.** *Accepted.* Central inspection, shared ingress/egress, and per-workload isolation outweigh peering cost and hub complexity. Alternative (single VPC, PNs as spokes) rejected: insufficient blast-radius and IAM isolation at 250 workloads.

**ADR-002 — Workload teams hold no network rights.** *Accepted (amended by ADR-011/012).* Segregation of duties requires the platform to own VPC/PN/peering/DNS everywhere, enforced by omission of permission sets (no explicit deny exists) plus automated audit. Network resources now live *inside* each `spoke-<name>` repo but are applied by a **platform-controlled** identity (`app-spoke-<name>-network-<env>`) scoped to the spoke project, its paths gated to platform `CODEOWNERS` — so workload teams still hold no network rights. Exception unchanged: `PrivateNetworksReadOnly` where product APIs require attachment from workload Terraform.

**ADR-003 — Deterministic CIDR derivation.** *Accepted; addressing scheme superseded by ADR-013.* The `cidrsubnet()`-from-registry-index determinism (no hand-picked addresses) stands; the original /14-per-environment allocation is replaced by /16-per-numbered-stamp (ADR-013) now that stamps are small and numerous.

**ADR-004 — All traffic through the hub, strictly spoke↔hub.** *Accepted.* Full inspection of ingress and egress; no spoke↔spoke routes; consequence: transitive peering unnecessary (single-hop flows).

**ADR-005 — Pools instead of singleton hub services.** *Accepted; default now single-shard-per-stamp.* NVA/LB/PGW remain shard maps with deterministic assignment, but because each stamp is small (§6.1) the default is **one shard per pool per stamp**; the pool exists so a hot/growing stamp scales by configuration. The 250-instance total is distributed across many single-shard hubs, so the fixed hub base multiplies across stamps rather than a single pool growing to 5–10 shards (§8, §18).

**ADR-006 — NVA stack: nftables + Suricata over OPNsense.** *Proposed.* Rationale: fully declarative via cloud-init/Git (OPNsense config is GUI-centric and harder to reconcile with drift detection), higher raw throughput per vCPU, smaller attack surface, no license/GUI exposure. Trade-off accepted: no GUI, HA is manual — mitigated by pool pattern and pipeline recreate. Decision to be ratified after Phase 1 throughput validation.

**ADR-007 — IPv6: v4-only edges, v6 filtered internally.** *Accepted.* PGWs lack IPv6; auto-assigned per-PN /64s are immutable, so v6 is explicitly filtered (mirrored NACLs, NVA v6 drop) rather than ignored. NAT64/DNS64/dual-stack deferred; revisit yearly.

**ADR-008 — Transitive peering enabled on hubs at creation.** *Accepted.* Not functionally needed (ADR-004), but the flag is immutable post-creation; enabling preserves future options at zero current cost.

**ADR-009 — Costs as formula + living rate sheet.** *Accepted.* Point-in-time prices in design documents go stale within a quarter; the model (§18) stays valid, the rate sheet stays current.

**ADR-010 — Reversible lock-in over cloud-agnostic abstraction.** *Accepted.* The portability goal is **exit capability** — a bounded-time rebuild on another provider — not active multi-cloud and not a lowest-common-denominator abstraction layer. The estate stays provider-native (managed LB, Kapsule, Managed DB, Cockpit) and pays no continuous portability tax; in exchange, every dependency must be *reversible* — portable state export **plus** a single replaceable adapter module (§16.2) — and the exit is rehearsed off-provider at least annually against the validation suite (§16.6). One-way-door services (no export or no equivalent) require an exit-impact ADR. Rejected alternatives: a cloud-agnostic abstraction (forfeits provider-native strengths, pays a standing tax for a capability rarely exercised) and self-hosting everything (ops burden without commensurate benefit where managed services have tested exports). The managed LB pool is retained under this ADR; replacing it with a self-managed reverse proxy is a possible future adapter-surface reduction, not a requirement.

**ADR-011 — Repo and Terraform state per workload; platform as repo-factory.** *Accepted.* The global layer is one `platform` repo (foundation + hub + versioned modules + registry + policies + repo-factory); each workload is its own `spoke-<name>` repo with its own state (network state + app state). State per spoke gives per-workload blast radius and lets onboarding be a single spoke apply; repo per spoke adds independent CI, access control, and a clean future path to fully separate ownership. The platform pipeline gains **GitHub org-admin** to mint and govern spoke repos as code. Rejected: one monolithic per-environment state (multi-thousand-resource plans, one lock serialises all onboarding) and Terragrunt (unneeded once each spoke is a single small root; the versioned module library is the DRY mechanism). Cost: ~one repo per workload to govern — handled by the repo-factory — and module-versioning discipline.

**ADR-012 — Networking lives in the spoke repo; two-sided peering is the duty boundary.** *Accepted.* Scaleway peering requires a connector in **each** VPC and only reaches `Peered` when both exist. The spoke repo (platform-controlled network identity, scoped to the spoke project only) builds the spoke-side connector, PNs, NACL and default route; the platform hub layer builds the hub-side connector, route and ingress rule (`for_each` over the registry). Neither identity needs rights in the other's VPC, and a spoke cannot self-connect — segregation of duties enforced by the platform's own topology rather than by permission bookkeeping. Cost: onboarding is a coordinated two-step (registry PR + spoke apply) and hub-side per-spoke resources touch the shared hub state at lifecycle events only (safe `for_each` additions; shard the hub state per stamp if it ever grows unwieldy).

**ADR-013 — Instance-numbered stamps; 250 as a global spoke-instance budget; /16 per stamp.** *Accepted (supersedes ADR-003 addressing).* Environments are numbered instances `<class><NN>` (two-digit, 01–99 per class). The 250 target is the **estate-wide** count of spoke instances (~20 workloads × their targeted stamps), not 250-per-environment; each stamp is small (≤~20 in practice, ≤60 by addressing). Each stamp gets one /16 with the second octet as the stamp id, making addressing self-identifying and non-overlapping within 10/8; class ranges prd `10.0–10.31`, stg `10.32–10.63`, dev `10.64–10.127`, reserved `10.128+`. Because stamps are independent (ADR-004), overlap is safe if ever needed; a /17-per-stamp or non-prod overlap covers the full 99×3 ceiling. This reframing also relieves the old per-hub quota risks (≤~60 per hub) and inverts the cost driver to fixed-base-×-stamp-count (§18).

**ADR-014 — Single manual bootstrap seam.** *Accepted.* Full IaC has one unavoidable chicken-and-egg: the Terraform state bucket and the first platform pipeline key must exist before anything can be managed as code. These two artefacts are created **manually once** and documented; everything downstream — projects, all IAM applications and keys, the repo-factory, every spoke repo and credential, and ≤90-day key rotation — is IaC. The manual step is exactly one bucket + one key; per-spoke credentials are never hand-made.
