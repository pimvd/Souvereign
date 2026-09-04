# Scaleway Landing Zone — Components

> What runs inside the containers: the shared resource pools (including the NVA inspection stack), capacity planning, and DNS.
>
> Part of the [Scaleway landing zone design](README.md) · **C3 — Components**
> Section numbers are stable across the split — see the [section map](README.md#section-map) to locate any `§` reference.

---

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

## 11. DNS and domains

Public zones are owned by the platform in `plt-connectivity-<env>` (or a shared DNS project where zones span stamps). The estate can host **multiple apex domains** — e.g. `example.com` and `example.nl` — each apex being its own platform-owned public zone; the managed set is declared explicitly as `dns_zones` in the registry (§14.2), and the platform pipeline holds `DomainsDNSFullAccess` across all of them. Workloads receive names under any managed apex (`<workload>.<env>.example.com`, a vanity host, or several domains at once) via records requested by PR. Each published hostname is a landing-zone-owned LB frontend with its own certificate (Let's Encrypt via LB, or imported); publishing on multiple domains therefore consumes more per-frontend certificates, which counts against LB cert limits (§8). Each public zone is billed per zone (€0.007/h ≈ €5.11/mo, 5 M requests included, then €0.0005/million), so the managed apex set is a small standing estate-level cost rather than a per-stamp one — two apexes ≈ €10/mo (§18). Private resolution is strictly spoke↔hub: spokes resolve hub-published service names via Scaleway's built-in private DNS per Private Network; no cross-spoke discovery exists by design. **Unbound on the NVA (§7.2) does not replace this.** Private-name resolution stays on Scaleway's per-PN resolver; Unbound serves and logs *public* name resolution for the tier-2 transparent path only. Where tier 1 (explicit proxy) is in force the proxy already sees the hostname, so Unbound is not on the enforcement path at all — a distinction worth keeping explicit, because putting resolution on the NVA otherwise widens its blast radius for no benefit (risk #6).
