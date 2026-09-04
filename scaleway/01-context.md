# Scaleway Landing Zone — System context

> What the estate is, what it looks like from the outside, and the rules that govern every decision below.
>
> Part of the [Scaleway landing zone design](README.md) · **C1 — System context**
> Section numbers are stable across the split — see the [section map](README.md#section-map) to locate any `§` reference.

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
