# Scaleway Landing Zone — Code

> How the estate is built and proven: repositories, modules, the workload registry, pipelines, and the validation suite.
>
> Part of the [Scaleway landing zone design](README.md) · **C4 — Code**
> Section numbers are stable across the split — see the [section map](README.md#section-map) to locate any `§` reference.

---

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
