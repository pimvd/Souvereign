# Scaleway Landing Zone — Cost model

> Four priced scenarios — the landing zone itself, an empty workload, a Kubernetes workload and a two-VM workload — plus the rate sheet.
>
> Part of the [Scaleway landing zone design](README.md) · **Cost model**
> Section numbers are stable across the split — see the [section map](README.md#section-map) to locate any `§` reference.

---

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
