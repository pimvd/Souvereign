# Scaleway Landing Zone — Risks, roadmap and open items

> The scale/quota risk register, the phased roadmap, and everything still undecided.
>
> Part of the [Scaleway landing zone design](README.md) · **Risks, roadmap and open items**
> Section numbers are stable across the split — see the [section map](README.md#section-map) to locate any `§` reference.

---

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
