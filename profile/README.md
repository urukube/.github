# Internal Developer Platform (IDP)

## What We Are Building

A **greenfield Internal Developer Platform** that gives every Business Unit (BU) a self-service, opinionated way to get production-grade Kubernetes environments — without doing raw infrastructure work.

A BU asks for an environment. The platform provisions it. Everything in between is abstracted behind **golden paths** (paved roads that make the right way the easy way) and **guardrails** (policy enforcement that makes the wrong way impossible).

This is operated by a **central platform team**, centrally funded, with BUs billed only for their own workload clusters (dev + prod). The orchestrator cluster is platform cost.

---

## The Economic Story

### The problem: cost centers that can't share

Large organisations are split into Business Units, each with its own P&L, its own AWS account, its own engineering team, and its own budget. This independence is intentional — it keeps accountability clean and prevents one team's spending from bleeding into another's.

But that independence creates a compounding infrastructure problem:

- Each BU builds its own VPC, its own EKS cluster, its own security baseline — from scratch, every time
- Standards diverge: one BU pins Kubernetes at 1.28, another at 1.32, a third skips IMDSv2 enforcement entirely
- Security posture is inconsistent: compliance reviews become a per-BU audit instead of a platform-wide one
- Platform engineers are hired and paid by each BU separately — the same problem solved four times over
- There is no shared toolchain, no shared observability, no shared golden path — just sprawl

The BUs are not connected. They cannot share a cluster (blast radius, billing, isolation). They will not accept a shared AWS account (cost attribution breaks down). A traditional "central infra" team that owns and operates shared clusters for everyone does not work here — it collapses accountability and becomes a bottleneck.

### The design: shared control plane, isolated data plane

This platform resolves the tension by separating **where decisions are made** from **where resources run**:

```
                    ┌─────────────────────────────────────┐
                    │        Orchestrator Cluster          │
                    │   (platform team · central budget)   │
                    │                                      │
                    │  Crossplane · ArgoCD · Backstage     │
                    │  One control plane for all BUs       │
                    └──────┬──────────┬──────────┬─────────┘
                           │          │          │
              sts:AssumeRole    sts:AssumeRole    sts:AssumeRole
                           │          │          │
               ┌───────────▼─┐  ┌─────▼───────┐  ┌▼────────────┐
               │  BU #1 AWS  │  │  BU #2 AWS  │  │  BU #3 AWS  │
               │   Account   │  │   Account   │  │   Account   │
               │  (BU pays)  │  │  (BU pays)  │  │  (BU pays)  │
               └─────────────┘  └─────────────┘  └─────────────┘
```

- BUs keep their own AWS accounts — cost attribution stays clean, billing is unchanged
- The orchestrator assumes a cross-account IAM role into each BU's account only when provisioning — it never co-mingles credentials or resources
- Each BU's VPC, EKS cluster, and S3 buckets are physically isolated in their own account; a failure in BU #2 cannot affect BU #1
- The platform team owns and pays for exactly one thing: the orchestrator cluster that drives everything else

### How costs scale

```
BU #1 onboards → pays for the spine (foundation + orchestrator cluster + CRD platform)
BU #2 onboards → rides the shared orchestrator, only pays for its own clusters
BU #3 onboards → marginal cost falls further
...
Cost per tenant trends DOWN as adoption grows
```

The orchestrator cluster is a flat tax on the platform team's budget. Every BU after the first rides existing infrastructure — Backstage, Crossplane, and ArgoCD are already running and ready to provision the next tenant on demand.

| Resource | Who pays | Scaling behaviour |
|---|---|---|
| Orchestrator Cluster (PROD + DR) | Platform team — central budget | Flat — does not grow with BU count |
| BU Dev Cluster | BU cost center | Linear — 1 cluster per BU |
| BU Prod Cluster | BU cost center | Linear — 1 cluster per BU |

### What each BU gains without losing autonomy

| Without the platform | With the platform |
|---|---|
| Hire infra engineers per BU | BU focuses on product engineering |
| 4–8 weeks to get a production cluster | Self-service in minutes via Backstage |
| Inconsistent security baseline per BU | Guardrails enforced at the platform layer |
| Compliance audits per BU, per quarter | One platform audit covers all BUs |
| Each BU owns and operates its own toolchain | Shared ArgoCD, Crossplane, observability stack |
| Cost sprawl — nobody knows the real per-BU number | Clean chargeback — BU pays exactly for what it runs |

Onboarding a new BU is a single pull request: one `EnvironmentConfig` file in `platform-tenant-registry` with the BU's cost center, owner email, and BU ID. No infrastructure changes. No new tooling. The next claim they submit gets provisioned with the full org compliance baseline automatically applied.

---

## How It Works

![urukube IDP — Provisioning and Delivery Flows](https://raw.githubusercontent.com/urukube/.github/master/.github/assets/idp-flow.svg)

There are two distinct flows in the platform:

### Flow ①: Self-Service Provisioning (once per BU environment)

A BU Admin opens Backstage and fills in an environment request form. They supply:

- AWS Account ID for the **dev** environment
- AWS Account ID for the **prod** environment
- Desired AWS region, node sizing, cost-center tag, and team contacts

Backstage converts this into an **Environment CRD** applied to the Orchestrator Cluster. Crossplane picks up that CRD, assumes a cross-account IAM role into the BU's AWS account, and builds the full environment:

| Resource | Detail |
|---|---|
| VPC | One isolated VPC per environment (dev and prod get separate VPCs in separate accounts) |
| Subnets | 1 public subnet (NAT GW + LB ENIs) · 2 private subnets (EKS worker nodes) |
| EKS Cluster | Self-managed node group · IMDSv2 enforced · EBS encryption enabled |
| IRSA + OIDC | Per-workload IAM roles mapped to Kubernetes service accounts |
| Add-ons | ArgoCD agent · ESO · Cilium CNI — bootstrapped at cluster creation |

The ArgoCD agent installed on the BU cluster registers it with the orchestrator's ArgoCD, enabling remote GitOps management from that point forward. **The orchestrator tracks all provisioned clusters in its CRD store but hosts none of their workloads.**

### Flow ②: Code Delivery (every git push)

Once a BU's clusters are provisioned, their developers work entirely through Git:

1. Developer pushes code to their GitHub repo (tagged as belonging to BU A)
2. GitHub CI runs: builds the image, signs it with cosign, generates an SBOM, pushes to JFrog Artifactory
3. The image tag is bumped in the Git overlay for the dev environment
4. ArgoCD on the orchestrator detects the Git change and reconciles **only BU A's dev cluster** — the Application scoping at provision time enforces isolation; BU B's cluster is never touched
5. Promotion to prod is a Git operation: bump the image tag in the prod overlay (via Kargo pipeline in Phase 2) and ArgoCD reflects it, with Argo Rollouts executing a canary or blue-green rollout

**Key principle: ArgoCD does not "push" code. It continuously reconciles each cluster to whatever Git declares. If a cluster drifts, ArgoCD corrects it automatically.**

---

## Cluster Topology

![Cluster Topology](https://raw.githubusercontent.com/urukube/.github/master/.github/assets/cluster-topology.svg)

> Full topology diagram, component breakdown, and DR strategy: [CLUSTER-TOPOLOGY.md](https://github.com/urukube/.github/blob/master/.github/CLUSTER-TOPOLOGY.md)

---

## GitOps Promotion Flow

![GitOps Promotion Flow](https://raw.githubusercontent.com/urukube/.github/master/.github/assets/gitops-promotion.svg)

> Full flow diagram, stage breakdown, and key principles: [GITOPS-PROMOTION.md](https://github.com/urukube/.github/blob/master/.github/GITOPS-PROMOTION.md)

---

## Secret Management

![ESO Secret Provisioning and Sync Flow](https://raw.githubusercontent.com/urukube/.github/master/.github/assets/secret-management-flow.svg)

> Full provisioning and sync flow, IRSA chain, ExternalSecret mappings, and how to add new secrets: [SECRET-MANAGEMENT.md](https://github.com/urukube/.github/blob/master/.github/SECRET-MANAGEMENT.md)

---

## Repository Architecture

![urukube Repository Calling Graph](https://raw.githubusercontent.com/urukube/.github/master/.github/assets/repo-calling-diagram.svg)

> Full calling graph, layer breakdown, module version pins, and S3 state coupling: [REPO-ARCHITECTURE.md](https://github.com/urukube/.github/blob/master/.github/REPO-ARCHITECTURE.md)

---

## Crossplane XRD Ecosystem

![Platform XP Ecosystem](https://raw.githubusercontent.com/urukube/.github/master/.github/assets/platform-ecosystem.svg)

> Crossplane concepts, shared pipeline, cross-account IRSA pattern, networking → EKS hand-off, and how to add new XRDs: [PLATFORM_ECOSYSTEM.md](https://github.com/urukube/.github/blob/master/.github/PLATFORM_ECOSYSTEM.md)

---

## Five CNCF Platform Planes

![Five CNCF Platform Planes](https://raw.githubusercontent.com/urukube/.github/master/.github/assets/cncf-planes.svg)

> Observability and Security are **cross-cutting** — they span all three core planes.

---

## Toolchain at a Glance

| Plane | Tool | Role |
|---|---|---|
| Developer Control | Backstage | Self-service portal — hosted on Orchestrator Cluster |
| Integration & Delivery | GitHub | VCS, CI |
| Integration & Delivery | Crossplane | CRD-driven infra provisioning (clusters, VPCs, IAM, etc.) |
| Integration & Delivery | JFrog Artifactory | Image & artifact registry |
| Integration & Delivery | ArgoCD | GitOps CD — hosted on Orchestrator Cluster |
| Integration & Delivery | Kargo | Multi-stage promotion pipelines |
| Integration & Delivery | Argo Rollouts | Progressive delivery (canary/blue-green) |
| Resource | Orchestrator Cluster | PROD + DR · runs Crossplane, ArgoCD, Backstage |
| Resource | BU Clusters | per-BU dev + prod · provisioned by Crossplane via CRDs |
| Observability | Prometheus / Thanos | Metrics (per-tenant scoped) |
| Observability | Grafana | Dashboards |
| Observability | OpenTelemetry | Telemetry collection |
| Observability | Loki / Tempo | Logs + traces |
| Security | External Secrets Operator | Secrets from cloud secret manager |
| Security | Kyverno | Admission-enforced policy |
| Security | Cilium | CNI + network policy |
| Security | cosign / sigstore | Image signing |
| Security | SPIFFE/SPIRE | Cross-cluster workload identity |

---

## KPIs

| Category | What to track |
|---|---|
| **Adoption** | BUs onboarded · time-to-first-environment |
| **Outcomes** | DORA: lead time · deploy frequency · change-failure rate · MTTR |
| **Governance** | % workloads behind enforced policy + signed supply chain |
| **Unit economics** | Cost per tenant trending down · spend attributed per cost center |
