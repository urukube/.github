# Internal Developer Platform (IDP)

## What we are building

A **greenfield Internal Developer Platform** that gives every business unit (BU) a self-service, opinionated way to get production-grade Kubernetes environments — without doing raw infrastructure work.

A BU asks for an environment. The platform provisions it. Everything in between is abstracted behind **golden paths** (paved roads that make the right way the easy way) and **guardrails** (policy enforcement that makes the wrong way impossible).

This is operated by a **central platform team**, centrally funded, with BUs billed only for their own workload clusters (dev + prod). The shared hub is platform cost.

---

## The Economic Story

```
BU #1 onboards → pays for the spine (foundation + shared hub + modules)
BU #2 onboards → rides the shared hub, only pays for its own clusters
BU #3 onboards → marginal cost falls further
...
Cost per tenant trends DOWN as adoption grows
```

The shared hub is a flat tax on the platform team's budget. Every BU after the first rides existing infrastructure — the inverse of a model where each new tenant adds a full dedicated hub.

---

## Five CNCF Platform Planes

```mermaid
block-beta
  columns 1

  block:observability["🔭 OBSERVABILITY PLANE — cross-cutting"]
    OBS1["Prometheus · Thanos · Mimir"]
    OBS2["Grafana Dashboards"]
    OBS3["OpenTelemetry Collector"]
    OBS4["Loki · Tempo"]
  end

  space

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
    I["per-BU Dev Cluster"]
    J["per-BU Prod Cluster"]
  end

  space

  block:foundation["🏗️ FOUNDATION"]
    K["Landing Zone (1 account/BU/env)"]
    L["Network Hub-Spoke (VPC peering)"]
    M["Identity / SSO · Git & IaC layout"]
  end

  space

  block:security["🔒 SECURITY PLANE — cross-cutting"]
    SEC1["ESO · OIDC · SPIFFE/SPIRE"]
    SEC2["Kyverno · OPA/Gatekeeper"]
    SEC3["Cilium · cosign · SBOM"]
  end

  developer --> delivery
  delivery --> resource
  resource --> foundation

  classDef obsClass   fill:#EDE9FE,stroke:#7C3AED,color:#3730A3
  classDef devClass   fill:#DBEAFE,stroke:#2563EB,color:#1E3A8A
  classDef delivClass fill:#D1FAE5,stroke:#059669,color:#064E3B
  classDef resClass   fill:#FEF9C3,stroke:#CA8A04,color:#713F12
  classDef foundClass fill:#F1F5F9,stroke:#475569,color:#1E293B
  classDef secClass   fill:#FFE4E6,stroke:#E11D48,color:#881337

  class OBS1,OBS2,OBS3,OBS4 obsClass
  class A,B,C devClass
  class D,E,F,G delivClass
  class H,I,J resClass
  class K,L,M foundClass
  class SEC1,SEC2,SEC3 secClass

  style observability fill:#EDE9FE,stroke:#7C3AED,color:#3730A3
  style developer     fill:#DBEAFE,stroke:#2563EB,color:#1E3A8A
  style delivery      fill:#D1FAE5,stroke:#059669,color:#064E3B
  style resource      fill:#FEF9C3,stroke:#CA8A04,color:#713F12
  style foundation    fill:#F1F5F9,stroke:#475569,color:#1E293B
  style security      fill:#FFE4E6,stroke:#E11D48,color:#881337
```

> Observability and Security are **cross-cutting** — shown as top/bottom bands to reflect that they span all three core planes.

---

## Cluster Topology: Shared Hub ✅

One shared hub (platform team owns) orchestrates per-BU dev and prod clusters. The hub runs no business workload — only ArgoCD fleet control and addon controllers.

```mermaid
graph TD
    HUB["🎛️ SHARED HUB\n(platform team owns)\nArgoCD fleet control\nAddon controllers\nNo BU workloads"]

    subgraph BUA["BU-A  (own cost center)"]
        A_DEV["dev cluster"]
        A_PROD["prod cluster"]
    end

    subgraph BUB["BU-B  (own cost center)"]
        B_DEV["dev cluster"]
        B_PROD["prod cluster"]
    end

    subgraph BUC["BU-C  (onboards later)"]
        C_DEV["dev cluster"]
        C_PROD["prod cluster"]
    end

    HUB --> A_DEV
    HUB --> A_PROD
    HUB --> B_DEV
    HUB --> B_PROD
    HUB --> C_DEV
    HUB --> C_PROD
```

| Model | 3 BUs = ? clusters | Cost trend |
| ----- | ------------------ | ---------- |
| **Shared hub** | **7 (1 hub + 6 clusters)** | **Hub is a flat tax — cost per BU falls** |
| Hub-per-BU | 9 (3 hubs + 6 clusters) | Grows linearly with adoption |

---

## How a BU Gets an Environment

```mermaid
flowchart TD
    DEV["👩‍💻 BU Engineer\n'I need a Kubernetes environment'"]

    PORTAL["Backstage Portal\nDeveloper Control Plane\n(Phase 4 — manual trigger for now)"]

    SPACELIFT["Spacelift Stack\nRuns Terraform\nEnforces OPA policy\n(cost-center tags · approved regions · no public LBs)"]

    TF["Terraform Module\nnetwork · cluster · node-pools · bootstrap\nProvisions cloud substrate\nInstalls ONE thing inside cluster"]

    ARGOCD["ArgoCD (in cluster)\napp-of-apps\nRegistered with shared hub\nInstalls everything else:\ningress · cert-manager · observability agents\npolicy controllers · secrets operator · apps"]

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
    DEV_SYNC["ArgoCD reconciles\ndev cluster ✅"]
    PROMOTE["Promotion = Git operation\n(Kargo pipeline or manual overlay bump)"]
    PROD_GIT["Bump image tag in Git\n(prod overlay)"]
    PROD_SYNC["ArgoCD reconciles\nprod cluster\nArgo Rollouts: canary / blue-green ✅"]

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
    Phase 1 - Spine · 1 BU    : Shared hub cluster
    (TODAY)                    : Dev + prod clusters for pilot BU
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
                               : request env → Spacelift → clusters + repo scaffold
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
