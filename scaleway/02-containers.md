# Scaleway Landing Zone — Containers

> The deployable and isolation units: Organization and projects, identities, stamps and their VPCs, and how traffic moves between them.
>
> Part of the [Scaleway landing zone design](README.md) · **C2 — Containers**
> Section numbers are stable across the split — see the [section map](README.md#section-map) to locate any `§` reference.

---

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

## 9. Traffic flows

**Egress (spoke → internet).** Workload resource → spoke default route → peering → hub ingress rule → assigned NVA shard (inspection, allowlist) → assigned PGW (NAT) → internet. Spokes have no Public Gateways and no public IPs.

**Ingress (internet → spoke).** Client → DNS (landing-zone-managed hostname) → assigned LB shard → TLS termination and health checks → backend private IP in the spoke via peering. A per-spoke flag can insert NVA inspection into the ingress path for high-sensitivity workloads.

**Management (operator → spoke).** Operator → PGW bastion (SSH, access-listed) → target private IP. Console is read-only; bastion access is the logged exception path.

**Spoke ↔ spoke.** Not permitted: no routes exist between spoke CIDRs, and NVA policy denies inter-spoke ranges as defense in depth.

**Kapsule specifics.** Node pools live in `pn-nodes`; nodes require control-plane, registry, and image endpoints on the NVA allowlist to bootstrap. Pod/service CIDRs are pinned in the spoke template from `100.64.0.0/10` to guarantee no overlap with the 10.0.0.0/8 plan.
