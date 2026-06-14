# Internal Developer Platform (IDP)

## What we are building

A **greenfield Internal Developer Platform** that gives every business unit (BU) a self-service, opinionated way to get production-grade Kubernetes environments — without doing raw infrastructure work.

A BU asks for an environment. The platform provisions it. Everything in between is abstracted behind **golden paths** (paved roads that make the right way the easy way) and **guardrails** (policy enforcement that makes the wrong way impossible).

This is operated by a **central platform team**, centrally funded, with BUs billed only for their own spoke clusters.

---

## The Economic Story

```
BU #1 onboards → pays for the spine (hub + foundation)
BU #2 onboards → rides shared infrastructure
BU #3 onboards → marginal cost falls further
...
Cost per tenant trends DOWN as adoption grows
```

The inverse of hub-per-BU, where every new tenant adds a full triple-cluster footprint and cost climbs in lockstep with adoption.

---

## Five CNCF Platform Planes

```mermaid
block-beta
  columns 1

  block:developer["🖥️ DEVELOPER CONTROL PLANE"]
    A["Backstage Portal"]
    B["Golden-path Templates"]
    C["Service Catalog"]
  end

  space

  block:delivery["🔁 INTEGRATION & DELIVERY PLANE"]
    D["GitHub (VCS)"]
    E["Spacelift (IaC + Policy)"]
    F["JFrog Artifactory"]
    G["ArgoCD · Kargo · Argo Rollouts"]
  end

  space

  block:resource["☸️ RESOURCE PLANE"]
    H["Shared Hub Cluster"]
    I["per-BU Dev Spoke"]
    J["per-BU Prod Spoke"]
  end

  space

  block:foundation["🏗️ FOUNDATION"]
    K["Landing Zone (1 account/BU/env)"]
    L["Network Hub-Spoke (VPC peering)"]
    M["Identity / SSO · Git & IaC layout"]
  end

  developer --> delivery
  delivery --> resource
  resource --> foundation
```

> Security (ESO · OIDC · Kyverno · Cilium · cosign · SBOM) and Observability (Prometheus · Thanos · Grafana · OTel · Loki · Tempo) are **cross-cutting** — they span all planes.

---

## Cluster Topology: Shared Hub + Per-BU Spokes

```mermaid
graph TD
    HUB["🎛️ SHARED HUB\n(platform team owns)\nArgoCD fleet control\nAddon controllers\nNo BU workloads"]

    subgraph BUA["BU-A  (own cost center)"]
        A_DEV["dev spoke"]
        A_PROD["prod spoke"]
    end

    subgraph BUB["BU-B  (own cost center)"]
        B_DEV["dev spoke"]
        B_PROD["prod spoke"]
    end

    subgraph BUC["BU-C  (onboards later)"]
        C_DEV["dev spoke"]
        C_PROD["prod spoke"]
    end

    HUB --> A_DEV
    HUB --> A_PROD
    HUB --> B_DEV
    HUB --> B_PROD
    HUB --> C_DEV
    HUB --> C_PROD
```

| Model          | 3 BUs = ? clusters                | Cost trend            |
| -------------- | --------------------------------- | --------------------- |
| Hub-per-BU     | 9 clusters (3 hubs + 6 spokes)    | Grows linearly        |
| **Shared hub** | **7 clusters (1 hub + 6 spokes)** | **Hub is a flat tax** |

---

## How a BU Gets an Environment

```mermaid
flowchart TD
    DEV["👩‍💻 BU Engineer\n'I need a Kubernetes environment'"]

    PORTAL["Backstage Portal\nDeveloper Control Plane\n(Phase 4 — manual trigger for now)"]

    SPACELIFT["Spacelift Stack\nRuns Terraform\nEnforces OPA policy\n(cost-center tags · approved regions · no public LBs)"]

    TF["Terraform Module\nnetwork · cluster · node-pools · bootstrap\nProvisions cloud substrate\nInstalls ONE thing inside cluster"]

    ARGOCD["ArgoCD (in spoke)\napp-of-apps\nRegistered with hub\nInstalls everything else:\ningress · cert-manager · observability agents\npolicy controllers · secrets operator · apps"]

    DEV --> PORTAL --> SPACELIFT --> TF --> ARGOCD
```

**Principle: "Terraform bootstraps, GitOps runs"**

---

## GitOps Promotion Flow

```mermaid
flowchart TD
    PUSH["👩‍💻 Developer pushes code"]
    PR["GitHub PR → merge"]
    CI["CI Build\nimage built + signed (cosign)\nSBOM generated\nimage pushed to JFrog Artifactory"]
    DEV_GIT["Bump image tag in Git\n(dev overlay)"]
    DEV_SYNC["ArgoCD reconciles\ndev spoke ✅"]
    PROMOTE["Promotion = Git operation\n(Kargo pipeline or manual overlay bump)"]
    PROD_GIT["Bump image tag in Git\n(prod overlay)"]
    PROD_SYNC["ArgoCD reconciles\nprod spoke\nArgo Rollouts: canary / blue-green ✅"]

    PUSH --> PR --> CI --> DEV_GIT --> DEV_SYNC --> PROMOTE --> PROD_GIT --> PROD_SYNC
```

---

## Phased Roadmap

```mermaid
timeline
    title IDP Adoption Roadmap
    Phase 0 - Foundations      : Landing zone (1 account per BU per env)
                               : Network hub-spoke (peered VPCs)
                               : Identity / SSO + workload identity
                               : Git & IaC repo layout
    Phase 1 - Spine · 1 BU    : Hub cluster (future-shared)
    (TODAY)                    : Dev + prod spokes for pilot BU
                               : ArgoCD fleet registration
                               : Reference app deployed
                               : Dev → prod promotion via Git
    Phase 2 - Productionize    : Spacelift + OPA policy-as-code
                               : ApplicationSets
                               : Kargo promotion pipelines
                               : Argo Rollouts (canary / blue-green)
    Phase 3 - Shared Planes    : Org-wide observability (per-tenant scoped)
                               : External Secrets Operator
                               : Kyverno admission policy
                               : cosign / SBOM / supply-chain controls
                               : Network policy (Cilium)
    Phase 4 - Self-Service     : Backstage golden path
                               : request env → Spacelift → spoke + repo scaffold
                               : (once 2–3 BUs exist and golden path is stable)
```

---

## Toolchain at a Glance

| Plane                  | Tool                      | Role                                     |
| ---------------------- | ------------------------- | ---------------------------------------- |
| Developer Control      | Backstage                 | Self-service portal (Phase 4)            |
| Integration & Delivery | GitHub                    | VCS, CI                                  |
| Integration & Delivery | Spacelift                 | IaC execution + OPA policy-as-code       |
| Integration & Delivery | JFrog Artifactory         | Image & artifact registry                |
| Integration & Delivery | ArgoCD                    | GitOps continuous delivery               |
| Integration & Delivery | Kargo                     | Multi-stage promotion pipelines          |
| Integration & Delivery | Argo Rollouts             | Progressive delivery (canary/blue-green) |
| Resource               | Terraform                 | Cluster + network provisioning           |
| Observability          | Prometheus / Thanos       | Metrics (per-tenant scoped)              |
| Observability          | Grafana                   | Dashboards                               |
| Observability          | OpenTelemetry             | Telemetry collection                     |
| Observability          | Loki / Tempo              | Logs + traces                            |
| Security               | External Secrets Operator | Secrets from cloud secret manager        |
| Security               | Kyverno                   | Admission-enforced policy                |
| Security               | Cilium                    | CNI + network policy                     |
| Security               | cosign / sigstore         | Image signing                            |
| Security               | SPIFFE/SPIRE              | Cross-cluster workload identity          |

---

## KPIs

| Category           | What to track                                                    |
| ------------------ | ---------------------------------------------------------------- |
| **Adoption**       | BUs onboarded · time-to-first-environment                        |
| **Outcomes**       | DORA: lead time · deploy frequency · change-failure rate · MTTR  |
| **Governance**     | % workloads behind enforced policy + signed supply chain         |
| **Unit economics** | Cost per tenant trending down · spend attributed per cost center |
