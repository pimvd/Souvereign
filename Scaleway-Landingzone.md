# Scaleway Landing Zone — Hub & Spoke Design

| | |
|---|---|
| **Status** | Draft v0.4 |
| **Date** | 3 August 2026 |
| **Owner** | Pim van Dijk |
| **Scope** | Network, identity, delivery, and operations architecture for a multi-environment, multi-workload Scaleway estate |
| **Changes in v0.2** | Architecture diagrams; capacity planning; availability assumptions; observability; security controls; Terraform module layout; validation; ADR appendix; formula-based cost model |
| **Changes in v0.3** | **Fix: spoke CIDR derivation overlapped hub /20 (now `n + 4`, spokes from 10.e.16.0/22)**; hub internal PN layout (§6.3); corrected NVA capacity model (planning formula + validation invariant, 4 dimensions); validation split into pre-/post-apply with rollback classes and Phase 1 LB acceptance gate |
| **Changes in v0.4** | New **Exit strategy** chapter (§16): reversible-lock-in principle (ADR-010), exit register, data-export requirement for stateful services, off-provider rebuild rehearsal and a time-to-exit SLO. Provider-neutral; managed LB pool retained unchanged. Subsequent sections renumbered §17–§20 |

---

## Author and contact

| | |
|---|---|
| **Name** | Pim van Dijk |
| **Location** | Rotterdam Area, Netherlands |
| **Available via** | [Team Rockstars IT](https://www.teamrockstars.nl/) |
| **Email** | [pim@vandijkcloud.nl](mailto:pim@vandijkcloud.nl) · [pim.vandijk@teamrockstars.nl](mailto:pim.vandijk@teamrockstars.nl) |

Questions about this design, or interested in a landing zone like this for your own estate? Feel free to reach out via either address above.

---

## 1. Purpose and scope

This document describes an enterprise-grade landing zone on Scaleway built around a hub-and-spoke network topology. It defines the environment model, the CIDR plan, the identity and segregation-of-duties model, the traffic flows, capacity and availability assumptions, and the delivery (IaC) model. It is designed to scale to **250 spokes (workloads) per environment** while keeping every network and identity decision under control of a central Landing Zone team.

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
     |  lb-0 ... lb-n  |                   |  pgw-0 ... pgw-n|  (NAT + bastion)
     +--------+--------+                   +--------+--------+
              |                                     ^
              |         +-----------------+         |
              |         |    NVA Pool     |---------+
              |         | nva-0 ... nva-n |  (inspection, allowlist)
              |         +--------+--------+
              |                  ^
   +----------+------------------+----------------------------+
   |                     HUB VPC  10.e.0.0/20                  |
   |          (routes, ingress rules, private DNS)             |
   +---+--------------+--------------+---------------------+---+
       |peering       |peering       |peering              |peering
   +---v----+     +---v----+     +---v----+            +---v----+
   | Spoke 0|     | Spoke 1|     | Spoke 2|   . . .    |Spoke249|
   |  /22   |     |  /22   |     |  /22   |            |  /22   |
   +--------+     +--------+     +--------+            +--------+

   Spoke <-> Spoke traffic: none (no routes, denied by policy)
```

### 2.2 Organization and project hierarchy

```
Scaleway Organization
|
+-- plt-connectivity-prd        (Hub VPC, pools, DNS, peering)
+-- plt-management-prd          (Cockpit, audit, TF state, runners)
+-- wl-amazingapp-prd           (Spoke VPC + workload resources)
+-- wl-awesomeapp-prd
+-- wl-<workload>-prd           ... up to 250 per environment
|
+-- plt-connectivity-stg
+-- plt-management-stg
+-- wl-<workload>-stg
|
+-- (plt-*/wl-*-dev — future stamp, same shape)

IAM (Organization level)
|
+-- grp-platform-humans          read-only, all projects
+-- app-platform-pipeline        network/IAM/DNS write, all projects
+-- grp-wl-<name>-humans         read-only, own wl-* projects
+-- app-wl-<name>-pipeline-<env> workload write, own project only
```

### 2.3 Spoke internal layout (/22)

```
+--------------------- Spoke VPC (10.e.x.0/22) ---------------------+
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
3. **Environment stamps.** An environment (prod, staging, dev, …) is a complete, independent copy of the architecture. No resource, network, or peering is shared between environments.
4. **Strictly spoke↔hub.** Spokes never communicate with each other. All ingress and egress traverses the hub for inspection.
5. **Deterministic addressing.** Every CIDR is derived by formula from an environment base and a spoke index. No IP address is ever hand-picked.
6. **Horizontal scalability of shared services.** Hub resources that could bottleneck (firewall NVAs, load balancers, public gateways) are deployed as **pools** with deterministic shard assignment, so scaling out is a configuration change rather than a redesign.
7. **Fail visible, not open.** Filtering defaults to explicit allow + trailing deny from the first apply; a missing rule blocks traffic rather than silently permitting it.
8. **Verify, then trust.** Every apply is followed by automated validation of routes, DNS, filtering, and connectivity (§15).

## 4. Organization and project structure

One Scaleway Organization contains all environments. Projects are the isolation and IAM-scoping boundary (see diagram §2.2).

| Project | Purpose | Contents |
|---|---|---|
| `plt-connectivity-<env>` | Hub for one environment | Hub VPC, NVA pool, LB pool, Public Gateway pool, DNS zones, peering connectors (hub side) |
| `plt-management-<env>` | Platform operations | Cockpit/observability, audit log sinks, Terraform state (Object Storage), optional GitHub runner pool |
| `wl-<workload>-<env>` | One workload, one environment | Spoke VPC (created by Landing Zone), workload resources (Kapsule, Instances, Managed Databases, storage) |

Naming convention: environments are `prd`, `stg`, `dev` (extensible). Workload names are short kebab-case identifiers registered in the workload registry (§14.2).

## 5. Identity and access model

### 5.1 Principals

Scaleway IAM permission sets are **product-scoped and project-scoped**; there is no resource-level RBAC and no explicit deny. Segregation therefore works by omission: each principal receives only the product permission sets it needs, in only the projects it needs.

| Principal | Type | Scope | Permission sets (indicative) |
|---|---|---|---|
| `grp-platform-humans` | IAM group | All projects | `*ReadOnly` across all products; **no** write sets |
| `app-platform-pipeline` | IAM application | Org + all current & future projects | `VPCFullAccess`, `PrivateNetworksFullAccess`, `VPCGatewayFullAccess`, `LoadBalancerFullAccess`, `DomainsDNSFullAccess`, `IPAMFullAccess`, `ProjectManager`, `IAMManager` |
| `grp-wl-<name>-humans` | IAM group | `wl-<name>-*` projects only | `*ReadOnly` for compute, K8s, DB, storage products |
| `app-wl-<name>-pipeline-<env>` | IAM application | `wl-<name>-<env>` project only | `InstancesFullAccess`, `KubernetesFullAccess`, `RelationalDatabaseFullAccess`, `ObjectStorageFullAccess`, `ContainerRegistryFullAccess` (as needed). **No VPC/network sets.** |

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

## 6. Network architecture

### 6.1 Environment stamps

Each environment is a stamp consisting of one hub VPC and up to 250 spoke VPCs, joined by VPC peering in a star topology. Traffic patterns are strictly spoke↔hub; because both egress (spoke → hub NVA → PGW) and ingress (hub LB → spoke) are single peering hops, transitive peering is not functionally required. It is nevertheless enabled on hub VPCs at creation time as insurance, because the setting is immutable after creation (ADR-008).

### 6.2 CIDR plan

Requirements: /22 per spoke, /20 per hub, up to 250 spokes per environment, clean per-environment summarization. Each environment receives a **/14** (256 × /22 blocks; 252 usable for spokes after the hub /20).

| Environment | Summary prefix | Hub | Spoke range |
|---|---|---|---|
| prod | `10.0.0.0/14` | `10.0.0.0/20` | `10.0.16.0/22` … `10.3.252.0/22` |
| staging | `10.4.0.0/14` | `10.4.0.0/20` | `10.4.16.0/22` … `10.7.252.0/22` |
| dev | `10.8.0.0/14` | `10.8.0.0/20` | `10.8.16.0/22` … `10.11.252.0/22` |
| reserved | `10.12.0.0/14` onward | — | future environments / second region |

Every firewall, NACL, and route rule can address an entire environment as a single /14; 10.0.0.0/8 accommodates 64 environments. Spoke CIDRs are computed, never chosen: the hub /20 consumes /22-blocks 0–3, so spoke *n* (0-based registry index) in environment *e* receives `cidrsubnet(env_base[e], 8, n + 4)` — spoke 0 = `10.e.16.0/22`, spoke 249 = block 253. A CI invariant asserts no generated spoke CIDR overlaps the hub /20 (§15).

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

Each spoke peers with its environment hub via a pair of connectors, both created by the landing zone pipeline (it holds VPC rights on both sides). Routing across a peering is never automatic; the pipeline generates: on the **spoke side**, a default route `0.0.0.0/0` toward the hub connector plus a route for the hub /20; on the **hub side**, one route per spoke /22 toward that spoke's connector, plus an ingress rule per spoke connector directing inbound-from-spoke traffic to the NVA shard assigned to that spoke (§7). The hub route table carries 250+ routes at full scale; this and connectors-per-VPC have no publicly documented quota and must be confirmed with Scaleway before scale-out (§17).

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

and equivalents for the session/CPS/PPS dimensions. With a shard validated at 2,000 Mbps inspected and 70% usable: `floor(2000 × 0.70 / 50) = 28` spokes at the default budget — in practice more, since budgets are peak allocations; measured utilization drives rebalancing (§12). Full scale (250 spokes) therefore implies an NVA pool of roughly 5–10 shards depending on real traffic, not 1.

**LB shard.** Bounded by frontends/certificates per LB and connections/throughput per LB type. Default `spokes_per_lb = 50` *(validate against LB type limits)*; TLS-heavy workloads may warrant overrides.

**PGW shard.** Bounded by NAT throughput and session table. One PGW per 2–3 NVA shards as a starting ratio *(validate)*.

**Hub control plane.** Routes (250+), ingress rules (250+), and connectors (500) are quota-bound, not performance-bound; confirmation with Scaleway is a hard prerequisite (§17).

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

Future path to region resilience (not committed): second-region stamp on the reserved 10.12.0.0/14+ ranges, DNS-based failover for ingress, per-workload data replication. Cross-region peering does not exist on Scaleway; stamps would be fully independent, which the environment-stamp model already supports.

## 11. DNS and domains

Public zones are owned by the landing zone in `plt-connectivity-<env>` (or a shared prod DNS project if zones span environments). Workloads receive delegated names (`<workload>.<env>.example.com` or vanity domains) via records managed in the landing zone repository; workload teams request records by PR. Private resolution is strictly spoke↔hub: spokes resolve hub-published service names via Scaleway's built-in private DNS per Private Network; no cross-spoke discovery exists by design.

## 12. Observability and logging

All telemetry converges in `plt-management-<env>` (Cockpit), with prod audit data additionally protected against tampering.

**Log routing.** NVA shards ship flow logs, IDS alerts, and allowlist denials; PGWs ship NAT/bastion session logs; LBs ship access logs; Kapsule and instances ship via the standard agents into the workload's Cockpit scope with copies of security-relevant streams to the platform scope. Scaleway audit trail (IAM changes, resource mutations) is exported continuously.

**Immutable audit.** Audit and bastion logs are additionally written to an Object Storage bucket with versioning and a compliance-style retention lock; the platform pipeline itself cannot delete within the retention window.

**Retention.** Defaults: metrics 13 months, application logs 30 days, security/flow logs 90 days, audit logs 400 days (immutable bucket). Per-workload overrides via the registry.

**Alerting.** Platform alert set: NVA/PGW/LB shard health, drift detection findings, IAM audit diffs, certificate expiry, peering/route validation failures, capacity thresholds (≥70% of any shard budget), and budget anomalies. Routed to the platform team's on-call channel.

**Dashboards.** One platform dashboard per environment: shard utilization vs. capacity model (§8), spoke count vs. quota headroom, top egress destinations, denied-flow trends, cost run-rate.

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

| Repository | Owned by | Contains |
|---|---|---|
| `landing-zone` | Platform | Everything platform: projects, IAM, hubs, pools, spokes' network layer, peering, DNS, registry, GitHub config |
| `workload-<name>` | Workload team | Application infrastructure deployed into `wl-<name>-<env>`, consuming network IDs as data sources |

```
landing-zone/
├── modules/
│   ├── environment/        # composition: one full stamp
│   ├── hub/                # hub VPC, routes, ingress rules
│   ├── spoke/              # spoke VPC, 3 PNs, NACL template, cidrsubnet math
│   ├── peering/            # connector pair + routes for one spoke
│   ├── pool-nva/           # NVA shard (instance, cloud-init, ruleset)
│   ├── pool-lb/            # LB shard (frontends, certs, backends)
│   ├── pool-pgw/           # PGW shard (NAT, bastion, access lists)
│   ├── dns/                # zones + per-workload records
│   ├── iam/                # groups, applications, policies, key rotation
│   ├── project/            # project + baseline (Cockpit scope, state access)
│   └── observability/      # Cockpit, alert rules, immutable audit bucket
├── environments/
│   ├── prd/                # backend.tf, prd.tfvars (pools, budgets)
│   ├── stg/
│   └── dev/                # reserved
├── registry/
│   └── workloads.hcl       # single source of truth (§14.2)
└── policies/               # OPA/Conftest rules (§13)
```

### 14.2 Workload registry

One record per workload, single source of truth:

```hcl
workloads = {
  "amazingapp" = {
    index            = 0        # drives CIDR derivation
    envs             = ["prd", "stg"]
    nva_shard        = null     # null = deterministic default (§7.1)
    lb_shard         = null
    egress_budget    = 50       # Mbps, feeds capacity model (§8)
    ingress          = [{ host = "api.amazingapp.example.com", backend_pn = "pn-app", port = 443 }]
    egress_allowlist = ["api.anthropic.com"]
  }
  "awesomeapp" = {
    index            = 1        # -> next /22 in each environment
    envs             = ["prd", "stg"]
    nva_shard        = null
    lb_shard         = null
    egress_budget    = 50
    ingress          = [{ host = "api.awesomeapp.example.com", backend_pn = "pn-app", port = 443 }]
    egress_allowlist = []
  }
}
```

Onboarding a workload = one PR adding a record. The pipeline creates the project, spoke network, peering + routes, IAM application + policy, GitHub wiring, DNS, and LB frontend — then runs validation (§15).

### 14.3 Pipelines and authentication

GitHub Actions is the only write path. Scaleway authentication uses IAM application API keys as GitHub environment secrets (no OIDC federation available); issuance and rotation per §5.3. Terraform state lives in Object Storage in `plt-management-<env>` with locking; workload states are isolated per workload per environment. Plans post to PRs; applies run only from protected default branches in protected environments.

## 15. Validation and testing

Validation is split into **pre-apply gates** (which genuinely block a change) and **post-apply verification** (a change that already landed cannot be "blocked" — it triggers a defined response).

**Pre-apply (blocking, in CI on every PR):** Terraform validate/plan review; policy-as-code suite (§13); generated-route invariants — exactly one hub route per registered spoke, no route to unregistered CIDRs, **no spoke CIDR overlapping the hub /20**; IAM permission diff against the §5.1 matrix; registry consistency (unique indices, budgets within shard validation invariants §8).

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
| 1 | Connectors per VPC quota undocumented (500/env at scale) | Blocks scale-out | **Confirm with Scaleway before wave 2 — top priority with #2** |
| 2 | Hub route-table and ingress-rule limits undocumented (250+ each) | Blocks scale-out | Confirm alongside #1 |
| 3 | LB backend health checks across peering unproven at scale | Ingress design risk | Phase 1 PoC gate |
| 4 | NACL in Public Beta, API-only | Feature risk | Acceptable (IaC-only estate); track GA |
| 5 | No explicit deny in IAM | Segregation by omission | Automated IAM audit (§13) |
| 6 | NVA shard is a single instance | Egress outage for assigned spokes | Health checks, pipeline recreate, per-shard blast radius; HA in roadmap |
| 7 | No OIDC federation for GitHub | Long-lived keys | Automated ≤90-day rotation, least-privilege per repo |
| 8 | Immutable IPv6 /64 per PN | Unfiltered v6 path if ignored | ADR-007: parallel v6 filtering, validated in §15.3 |
| 9 | Region loss = environment loss | Availability | Accepted (§10); quarterly rebuild exercise; region resilience on roadmap |
| 10 | Peering billing × 500 connectors/env | Standing cost | Cost model §18; connectors created only at onboarding |
| 11 | Exit capability asserted but unproven until first off-provider rebuild | Strategic / lock-in | §16: exit register + annual off-provider rebuild, validation suite as acceptance gate |

## 18. Cost model

Costs are maintained as a **formula plus a living rate sheet** (separate spreadsheet, reviewed quarterly) — no point-in-time prices in this document.

```
env_cost = Σ nva_pool(instance_rate)
         + Σ lb_pool(lb_rate)
         + Σ pgw_pool(pgw_rate)
         + spokes × 2 × connector_hourly_rate
         + cockpit(ingestion, retention)
         + object_storage(state + audit)
         + spokes × marginal(certs, logs)
```

Consequences the formula makes visible: connector cost is the only strictly linear-per-spoke platform cost and dominates at 250 spokes; NVA pool cost steps with the capacity model (§8), so egress budgets directly drive platform cost; a dev stamp costs the full fixed base — a reduced dev pool profile (smaller shard types) is a supported variant.

## 19. Roadmap

**Phase 1 — Foundation:** landing zone repo + module skeleton, prod + staging stamps (one shard per pool), first two workloads, validation suite (§15), IAM audit, PoC gates for risks #1–#3.
**Phase 2 — Scale hardening:** quota confirmations, shard scale-out exercised against the capacity model, break-glass tested, dashboards + cost sheet live, first quarterly rebuild.
**Phase 3 — Enhancements:** NVA HA pattern (active/passive per shard), session-recording bastion, dev stamp, per-spoke ingress inspection, NACL GA adoption, region-resilience design study, first off-provider exit rehearsal against the validation suite (§16).

## 20. Open items

1. Scaleway quota confirmation: peering connectors per VPC, hub route-table entries, ingress rules per VPC.
2. LB → cross-peering backend behavior PoC (risk #3).
3. Validate NVA inspected-throughput assumptions per chosen instance type (§8).
4. Confirm Kapsule pod/service CIDR pinning against `100.64.0.0/10`.
5. Bastion access-review cadence and immutable log destination sizing.

---

## Appendix A — Architecture Decision Records

**ADR-001 — Hub & Spoke over flat/single-VPC.** *Accepted.* Central inspection, shared ingress/egress, and per-workload isolation outweigh peering cost and hub complexity. Alternative (single VPC, PNs as spokes) rejected: insufficient blast-radius and IAM isolation at 250 workloads.

**ADR-002 — Workload teams hold no network rights.** *Accepted.* Segregation of duties requires the landing zone to own VPC/PN/peering/DNS everywhere, enforced by omission of permission sets (no explicit deny exists) plus automated audit. Exception: `PrivateNetworksReadOnly` where product APIs require attachment from workload Terraform.

**ADR-003 — Deterministic CIDR derivation.** *Accepted.* /14 per environment, /20 hub, /22 spokes computed via `cidrsubnet()` from registry index. No hand-picked addresses; clean per-environment summarization; 64-environment headroom in 10/8.

**ADR-004 — All traffic through the hub, strictly spoke↔hub.** *Accepted.* Full inspection of ingress and egress; no spoke↔spoke routes; consequence: transitive peering unnecessary (single-hop flows).

**ADR-005 — Pools instead of singleton hub services.** *Accepted.* NVA/LB/PGW as shard maps with deterministic spoke assignment; capacity scales by configuration; per-shard blast radius. Driven by 250-spoke target and single-instance NVA constraint.

**ADR-006 — NVA stack: nftables + Suricata over OPNsense.** *Proposed.* Rationale: fully declarative via cloud-init/Git (OPNsense config is GUI-centric and harder to reconcile with drift detection), higher raw throughput per vCPU, smaller attack surface, no license/GUI exposure. Trade-off accepted: no GUI, HA is manual — mitigated by pool pattern and pipeline recreate. Decision to be ratified after Phase 1 throughput validation.

**ADR-007 — IPv6: v4-only edges, v6 filtered internally.** *Accepted.* PGWs lack IPv6; auto-assigned per-PN /64s are immutable, so v6 is explicitly filtered (mirrored NACLs, NVA v6 drop) rather than ignored. NAT64/DNS64/dual-stack deferred; revisit yearly.

**ADR-008 — Transitive peering enabled on hubs at creation.** *Accepted.* Not functionally needed (ADR-004), but the flag is immutable post-creation; enabling preserves future options at zero current cost.

**ADR-009 — Costs as formula + living rate sheet.** *Accepted.* Point-in-time prices in design documents go stale within a quarter; the model (§18) stays valid, the rate sheet stays current.

**ADR-010 — Reversible lock-in over cloud-agnostic abstraction.** *Accepted.* The portability goal is **exit capability** — a bounded-time rebuild on another provider — not active multi-cloud and not a lowest-common-denominator abstraction layer. The estate stays provider-native (managed LB, Kapsule, Managed DB, Cockpit) and pays no continuous portability tax; in exchange, every dependency must be *reversible* — portable state export **plus** a single replaceable adapter module (§16.2) — and the exit is rehearsed off-provider at least annually against the validation suite (§16.6). One-way-door services (no export or no equivalent) require an exit-impact ADR. Rejected alternatives: a cloud-agnostic abstraction (forfeits provider-native strengths, pays a standing tax for a capability rarely exercised) and self-hosting everything (ops burden without commensurate benefit where managed services have tested exports). The managed LB pool is retained under this ADR; replacing it with a self-managed reverse proxy is a possible future adapter-surface reduction, not a requirement.
