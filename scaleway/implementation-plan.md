# Scaleway Landing Zone — Implementation Plan

A build guide for implementing the design in [`landing-zone.md`](./landing-zone.md) incrementally, in a way that suits AI-assisted ("vibe") coding: small, independently verifiable steps, each gated by a concrete Definition of Done. Section references (§) point at the design doc.

> **Golden rule:** the design doc is the source of truth. This plan sequences *how* to build it; it never overrides *what* to build. If the two disagree, fix the code or raise a design PR — do not drift.

---

## Author and contact

| | |
|---|---|
| **Name** | Pim van Dijk |
| **Location** | Rotterdam Area, Netherlands |
| **Available via** | [Team Rockstars IT](https://www.teamrockstars.nl/) |
| **Email** | [pim@vandijkcloud.nl](mailto:pim@vandijkcloud.nl) · [pim.vandijk@teamrockstars.nl](mailto:pim.vandijk@teamrockstars.nl) |
| **Scaleway certifications** | Scaleway Foundations · Associate Network · Associate Security & Identity — listed in [the design document](./landing-zone.md#scaleway-certifications) |

Questions about this plan, or interested in a landing zone like this for your own estate? Feel free to reach out via either address above.

---

## 0. How to use this plan

- **Work one task at a time.** Each task is small enough to implement, `terraform validate` + `plan`, review the plan, and commit before moving on. Never batch a whole phase into one apply.
- **Verify after every step.** Run, in order: `terraform fmt -check`, `terraform validate`, `tflint`, `tfsec`/`checkov`, `conftest` (policies), then `plan`. Only apply after a human/agent has read the plan.
- **Never hand-pick an IP.** All CIDRs come from the formulas in §6.2 (`cidrsubnet("10.${B}.0.0/12", 10, w + 2)`, where `B = S * 16` and stamp id `S` is allocated from the §6.2 block table). A literal `10.x` address in a resource block is a bug.
- **Pin everything.** Pin the Terraform version, provider versions, and — in spoke repos — the `platform` module release tag. Upgrades are deliberate tag bumps, rolled out per spoke.
- **Secrets never touch code or state.** Only GitHub environment secrets (§14.3). CI asserts no secret in state.
- **No Terragrunt.** Each spoke is a single small root per stamp; the DRY mechanism is the versioned module library (§14.1, ADR-011).

### Tech stack

| Concern | Choice |
|---|---|
| IaC | Terraform ≥ 1.7, Scaleway provider, GitHub provider |
| Policy-as-code | OPA/Conftest + tfsec + Checkov (§13) |
| CI/CD | GitHub Actions (only write path, §14.3) |
| NVA stack | nftables + Suricata via cloud-init (ADR-006) |
| State | Object Storage (S3-compatible) in `plt-management-*`, per stamp / per spoke |

### Guardrails for the coding agent (paste into the agent's working rules)

1. Do not create public IPs in a spoke project. Ingress/egress is via the hub only (§9).
2. Do not grant a workload/app identity any VPC/network permission set (§5.1, ADR-002).
3. Do not put hub-side resources in a spoke's state, or spoke-side resources in the hub state (§6.6, ADR-012).
4. Do not invent addresses, shard names, or project names — derive them from the registry + formulas.
5. Every new platform dependency updates the exit register (§16.4) in the same PR.

---

## Conventions (implement these first as shared locals)

- **Stamp id:** allocated from the §6.2 block table, not derived — `prd01 = 0`, `dev01 = 4`, `tst01 = 5`, `acc01 = 6`; reserves `1`–`3`. Base octet `B = S * 16`; stamp prefix `10.B.0.0/12` (§6.2, ADR-015).
- **Hub:** `10.B.0.0/21`, carved into `pn-hub-transit` `10.B.0.0/23` / `-egress` `10.B.2.0/24` / `-ingress` `10.B.3.0/24` / `-management` `10.B.4.0/24` (§6.3).
- **Spoke w (0-based registry index):** `cidrsubnet("10.${B}.0.0/12", 10, w + 2)` → `10.B.8.0/22` for w=0 (§6.2); PNs `pn-nodes` /23, `pn-app` /24, `pn-data` /24 (§6.4).
- **Private Network budget:** `Σ over stamps (4 + 3 × spokes) ≤ 255` — the Organization-wide Scaleway quota and the estate's real ceiling (§6.1, §17). Assert it in CI before it is ever approached.
- **Naming:** projects `plt-connectivity-<stamp>`, `plt-management-<stamp>`, `wl-<name>-<stamp>`; repos `platform`, `spoke-<name>` (§4, §14.1).
- **Kapsule pod/service CIDR:** pinned in `100.64.0.0/10` (§9).

---

## Phase 0 — Manual bootstrap (one-time, human) — ADR-014

**Goal:** the single manual seam, then never again.

- [ ] Create the Object Storage bucket for Terraform state (versioned) — by hand.
- [ ] Create the first `app-platform-pipeline` IAM application + API key — by hand.
- [ ] Store the key as a GitHub environment secret in a protected `bootstrap` environment.
- [ ] Document both steps in `platform/docs/bootstrap.md`.

**Done when:** a trivial `terraform plan` from the `platform` repo authenticates and reaches the state backend. Nothing else is manual after this.

---

## Phase 1 — Platform foundation

**Goal:** `platform` repo scaffold + the foundation layer (org/projects, IAM, DNS zones, observability, state backends, repo-factory) as code.

- [ ] Scaffold `platform/` per the §14.1 tree (`modules/`, `stamps/`, `registry/`, `repo-factory/`, `policies/`).
- [ ] `modules/foundation/`: projects, IAM groups/applications + key rotation (§5.1), state backends, `dns_zones` public zones (§11), GitHub org governance.
- [ ] `modules/observability/`: Cockpit scope, alert rules, immutable audit bucket (§12).
- [ ] `modules/iam/`: the §5.1 matrix as code, plus the IAM-audit CI job (diff live vs matrix).
- [ ] `registry/workloads.hcl`: schema per §14.2 (empty `workloads = {}`, real `dns_zones`).
- [ ] `policies/`: Conftest rules — no public IP in spokes, no network sets on app identities, naming/CIDR conformity, registry consistency (§13, §15 pre-apply).
- [ ] CI: `fmt/validate/tflint/tfsec/checkov/conftest/plan` on PR; protected `production` environment for apply (§5.3).

**Done when:** foundation applies clean; IAM audit job is green; policy suite blocks a deliberately bad PR (e.g. a public IP in a spoke).

**Suggested agent prompt:** *"Implement `modules/foundation` for the Souvereign platform repo per §4, §5.1, §11, §12 of the design doc. Provide variables, outputs (project IDs, DNS zone IDs), and a Conftest policy that fails if any app-* identity is granted a VPC permission set. Do not create hub or spoke resources yet."*

---

## Phase 2 — Hub layer (one stamp, `dev01`)

**Goal:** a working hub for a single numbered stamp with one shard per pool.

- [ ] `modules/hub/`: hub VPC `10.B.0.0/21`, the four hub PNs (§6.3), baseline routing/NACL.
- [ ] `modules/pool-nva/`: one NVA shard (COMPUTE3-X16C-32G for prod, POP2-2C-8G for non-prod — §8) with nftables+Suricata cloud-init + code-reviewed egress allowlist (§7.2, ADR-006).
- [ ] `modules/pool-lb/`: one LB-S shard, frontends/certs plumbing (§7.3).
- [ ] `modules/pool-pgw/`: one PGW shard, NAT + bastion access list (§7.4).
- [ ] `modules/hub-peering/`: hub-**side** connector + route + ingress rule for one spoke, driven by `for_each` over registry (§6.6) — empty until Phase 3.
- [ ] `stamps/dev01/`: root module wiring foundation outputs + hub + pools; `dev01.tfvars` (pools, budgets, `shared_hub = false`).
- [ ] Tag the module library `v0.1.0`.

**Done when:** `dev01` hub applies; packet-path invariants hold (default route only on `pn-hub-egress`, §6.3); dashboards show shard health.

---

## Phase 3 — Spoke template + first spoke (two-sided peering) — ADR-012

**Goal:** one workload end-to-end, proving the split-identity + two-sided peering model.

- [ ] `modules/` (consumed by spokes): `spoke` (VPC, 3 PNs, NACL template, cidrsubnet math) and `spoke-peering` (spoke-side connector + default route).
- [ ] Create the `spoke-<name>` **template** repo: `network/<stamp>/`, `app/<stamp>/`, `CODEOWNERS` (`/network/` → platform), `.github/` environments.
- [ ] Register the first workload in `registry/workloads.hcl` (`index = 0`, `stamps = ["dev01"]`, one ingress host).
- [ ] Platform apply: hub-side wiring for the spoke appears (via `hub-peering` `for_each`); repo-factory mints `spoke-amazingapp`.
- [ ] Spoke `network/dev01` apply (platform-gated identity): spoke VPC, PNs, NACL, spoke-side connector, default route → peering reaches `Peered`.
- [ ] Spoke `app/dev01` apply (workload identity): a trivial backend in `pn-app`, consuming network IDs via the published contract (§14.5) as read-only data.

**Done when:** peering is `Peered`; egress reaches an allowlisted domain via the NVA→PGW path and a non-allowlisted probe is denied; ingress serves TLS through the LB to the spoke backend (§15 steps 1–5). **This is the Phase 1 architecture acceptance gate** (§15) once a second spoke exists.

**Suggested agent prompt:** *"Implement the `spoke` and `spoke-peering` modules and the spoke repo template per §6.4, §6.6, §14.1. The network identity must be scoped to the spoke project only and must NOT create any hub-side resource. Show the two-sided peering coming up when both connectors exist."*

---

## Phase 4 — Validation suite (conformance) — §15

**Goal:** the executable definition of a correct stamp; it is also the exit-conformance gate (§16).

- [ ] **Pre-apply** checks in CI: route invariants, no CIDR overlap (spoke vs hub, **and stamp vs stamp**), IAM diff, registry consistency (unique index, valid `stamps`, Σ instances ≤ 250) (§15).
- [ ] **Post-apply** canary suite from `pn-hub-management` + per-new-spoke canary: peering, routing, filtering, egress, ingress, DNS (§15 steps 1–6).
- [ ] Per-module failure class (`rollback-safe` / `manual-recovery-required` / `forward-fix-only`) wired to responses (§15).
- [ ] Nightly run + promotion gate (`tst01` → `acc01` → `prd01`).

**Done when:** the suite runs green on `dev01`, and a deliberately broken change (e.g. missing NACL allow) is caught with the correct failure class.

---

## Phase 5 — Onboarding automation

**Goal:** onboarding a workload = the two-step in §14.2, hands-off after PRs.

- [ ] Finish `repo-factory/`: create/govern `spoke-<name>` repos (branch protection, CODEOWNERS, environments, per-spoke Scaleway keys) purely from the registry.
- [ ] Wire the published cross-repo contract (§14.5): hub outputs → parameter store/object per stamp; spoke data source reads it.
- [ ] Add a second workload (`index = 1`) and confirm zero hand-editing beyond the two PRs.

**Done when:** a single registry PR + the minted spoke repo's applies produce a fully wired, validated spoke; no manual credential or IP.

---

## Phase 6 — Remaining stamps + instance numbering — ADR-015

**Goal:** prove the stamp abstraction and addressing across the full four-stamp estate.

- [ ] Add `stamps/tst01/`, `stamps/acc01/` and `stamps/prd01/` roots; confirm the allocated stamp id → `/12` mapping (`prd01 = 10.0`, `dev01 = 10.64`, `tst01 = 10.80`, `acc01 = 10.96`).
- [ ] Confirm hub `10.B.0.0/21` and first spoke `10.B.8.0/22` in each stamp.
- [ ] Target a workload at multiple stamps (`stamps = ["prd01","acc01","tst01","dev01"]`); confirm per-stamp state isolation.
- [ ] CI asserts unique stamp ids, no two stamps' `/12`s overlap, and the `Σ (4 + 3 × spokes) ≤ 255` Private Network budget.
- [ ] Promotion (`tst01` → `acc01` → `prd01`) gated on green post-apply in the preceding stamp.

**Done when:** four stamps coexist with deterministic, non-overlapping addressing and independent state, and the PN budget invariant is enforced in CI.

---

## Phase 7 — Scale & cost controls

**Goal:** keep the estate cost sane — the platform base, the €29.20/mo per-spoke peering baseline, and workload compute (§18).

- [ ] Implement the **shared non-prod hub** flag (§10): `dev*`/`tst*`/`acc*` spokes peering into one shared hub; prod never shared.
- [ ] Non-prod pool profile: smaller NVA instance type (POP2-2C-8G at the 10 Mbps per-spoke budget — it covers only 2 spokes at 50 Mbps, §8), single-AZ (§18 rate sheet).
- [ ] TTL/auto-suspend profile for ephemeral dev stamps.
- [ ] Cost dashboard: per-stamp and total fixed-base run-rate (§12).

**Done when:** a non-prod group runs on one shared hub and the cost dashboard reflects the saving. **Blocked-on:** confirm the €/connector peering rate (open item 1) before wide scale-out.

---

## Phase 8 — Exit rehearsal scaffolding — §16

**Goal:** make exit capability real, not asserted.

- [ ] Ensure every stateful service ships logical, provider-independent backups (§16.5).
- [ ] Stand up the exit register as a maintained file, updated per dependency PR (§16.4).
- [ ] Draft one alternate-provider adapter set (hub/spoke/peering/iam/pool-*/dns) behind the same module interfaces.
- [ ] Run the validation suite against a thin dev stamp on the alternate provider; record the time-to-exit (§16.6).

**Done when:** the validation suite passes once on a second provider and a time-to-exit number is published.

---

## Module inventory

| Module | Layer | Design refs | Phase |
|---|---|---|---|
| `foundation` | platform | §4, §5, §11, §12 | 1 |
| `iam` | platform | §5.1, §5.3 | 1 |
| `observability` | platform | §12 | 1 |
| `hub` | platform | §6.1, §6.3 | 2 |
| `pool-nva` / `pool-lb` / `pool-pgw` | platform | §7 | 2 |
| `hub-peering` | platform | §6.6 (hub side) | 2–3 |
| `repo-factory` | platform | §14.1, §5.3 | 3, 5 |
| `spoke` | spoke | §6.4 | 3 |
| `spoke-peering` | spoke | §6.6 (spoke side) | 3 |
| validation suite | both | §15 | 4 |

---

## Decisions to resolve while coding (from §20 open items)

1. ~~€/hour peering connector rate~~ — **closed**: €0.02/h per connector = €29.20/mo per spoke (§18, risk #10). Peering is 59–74% of platform run-rate, so treat each workload's `stamps` list as a cost decision and report peering spend per workload on the dashboard.
2. **Shared non-prod hub vs per-stamp hub** — decide before Phase 7 (§10). Biggest cost lever.
3. **Private Network quota increase** — request from Scaleway Support before the estate approaches 255 PNs; it, not addressing, is the ceiling (§6.1, risk #2). Confirm the per-VPC route quota in the same request.
4. **NVA instance type + inspected throughput** — validate in the Phase 2/3 PoC (§8, risk #17, ADR-006 ratification). Prod default is COMPUTE3-X16C-32G (4 Gbps, dedicated cores); the 30–50%-of-line-rate IDS factor underpins every shard number and is still unmeasured.
5. **PGW and LB shard sizing** — prod defaults are **VPC-GW-M** (1 Gbps) and **LB-GP-M** (500 Mbps); VPC-GW-S at 100 Mbps would throttle the planned ~950 Mbps stamp egress ~10× (§8, risk #16). Validate against measured throughput before ratifying non-prod VPC-GW-S.

---

*Companion to `landing-zone.md` (Draft v0.5). Keep this plan's phase Definitions of Done aligned with the validation suite (§15); if a phase can't be proven by the suite, the suite is missing a check.*
