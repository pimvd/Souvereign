# Scaleway Landing Zone — Exit strategy

> Reversible lock-in: the exit register, portable state export, and the rehearsed off-provider rebuild.
>
> Part of the [Scaleway landing zone design](README.md) · **Exit strategy**
> Section numbers are stable across the split — see the [section map](README.md#section-map) to locate any `§` reference.

---

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
