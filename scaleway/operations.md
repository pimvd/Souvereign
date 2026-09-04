# Scaleway Landing Zone — Operations

> Availability and disaster recovery, observability, and the delivery-chain security controls.
>
> Part of the [Scaleway landing zone design](README.md) · **Operations**
> Section numbers are stable across the split — see the [section map](README.md#section-map) to locate any `§` reference.

---

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

**Cost-driven variant — shared non-prod hub.** Because every stamp carries a full always-on hub base (§18), non-prod stamps *may* share one hub — `dev*`, `tst*` and `acc*` spokes peering into a single shared hub — to collapse three hub bases into one, trading inter-stamp isolation for cost. It also reclaims Private Network headroom (8 PNs saved, §6.1). **Prod stamps are never shared.** Ephemeral dev stamps additionally support a TTL/auto-suspend profile. This is the single biggest cost lever (§18) and is a supported deployment shape, selected by a per-stamp flag rather than a redesign.

## 12. Observability and logging

All telemetry converges in `plt-management-<env>` (Cockpit), with prod audit data additionally protected against tampering.

**Log routing.** NVA shards ship flow logs, IDS alerts, and allowlist denials; PGWs ship NAT/bastion session logs; LBs ship access logs; Kapsule and instances ship via the standard agents into the workload's Cockpit scope with copies of security-relevant streams to the platform scope. Scaleway audit trail (IAM changes, resource mutations) is exported continuously.

**Immutable audit.** Audit and bastion logs are additionally written to an Object Storage bucket with versioning and a compliance-style retention lock; the platform pipeline itself cannot delete within the retention window.

**Retention.** Defaults: metrics 13 months, application logs 30 days, security/flow logs 90 days, audit logs 400 days (immutable bucket). Per-workload overrides via the registry.

**Alerting.** Platform alert set: NVA/PGW/LB shard health, drift detection findings, IAM audit diffs, certificate expiry, peering/route validation failures, capacity thresholds (≥70% of any shard budget), and budget anomalies. Routed to the platform team's on-call channel.

**Dashboards.** One platform dashboard per stamp (rolled up across the estate): shard utilization vs. capacity model (§8), spoke count vs. quota headroom, per-stamp and total fixed-base cost run-rate, **peering spend per workload** and **compute spend per stamp** (§18.8), top egress destinations, denied-flow trends.

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
| Egress destination control | Per-spoke FQDN + port allowlist enforced on the NVA (§7.2), generated from the registry; wildcards and IP exceptions require named sign-off and expire | Blocking (PR review); denied flows logged and alerted (§12) |

GitHub Advanced Security features beyond the free tier (code scanning on private repos) are a licensing decision, not an architectural one; the controls above function without it.
