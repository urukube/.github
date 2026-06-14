# Internal Developer Platform (IDP)

## What we are building

A **greenfield Internal Developer Platform** that gives every business unit (BU) a self-service, opinionated way to get production-grade Kubernetes environments — without doing raw infrastructure work.

A BU asks for an environment. The platform provisions it. Everything in between is abstracted behind **golden paths** (paved roads that make the right way the easy way) and **guardrails** (policy enforcement that makes the wrong way impossible).

This is operated by a **central platform team**, centrally funded, with BUs billed for their own hub and workload clusters.

---

## The Economic Story

```
BU #1 onboards → gets its own hub + dev/prod clusters
BU #2 onboards → same pattern, fully isolated cost center
BU #3 onboards → predictable, repeatable per-BU cost
...
Cost per BU is directly attributable and cleanly isolated
```

Each BU owns its hub — blast radius is contained, billing is unambiguous, and no BU ever shares a control plane with another.

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
    H["per-BU Hub Cluster"]
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

## Cluster Topology: Hub-per-BU

Each BU gets its own dedicated hub (ArgoCD fleet control) plus dev and prod spokes — fully isolated cost center, blast radius, and control plane.

```mermaid
graph TD
    subgraph BUA["BU-A  (own cost center)"]
        A_HUB["🎛️ BU-A Hub\nArgoCD · Addon controllers"]
        A_DEV["dev cluster"]
        A_PROD["prod cluster"]
        A_HUB --> A_DEV
        A_HUB --> A_PROD
    end

    subgraph BUB["BU-B  (own cost center)"]
        B_HUB["🎛️ BU-B Hub\nArgoCD · Addon controllers"]
        B_DEV["dev cluster"]
        B_PROD["prod cluster"]
        B_HUB --> B_DEV
        B_HUB --> B_PROD
    end

    subgraph BUC["BU-C  (onboards later)"]
        C_HUB["🎛️ BU-C Hub\nArgoCD · Addon controllers"]
        C_DEV["dev cluster"]
        C_PROD["prod cluster"]
        C_HUB --> C_DEV
        C_HUB --> C_PROD
    end
```

| What each BU gets | Clusters | Isolation |
| ----------------- | -------- | --------- |
| Hub + dev + prod  | 3        | Full — dedicated control plane, IAM, blast radius |

---

## How a BU Gets an Environment

```mermaid
flowchart TD
    DEV["👩‍💻 BU Engineer\n'I need a Kubernetes environment'"]

    PORTAL["Backstage Portal\nDeveloper Control Plane\n(Phase 4 — manual trigger for now)"]

    SPACELIFT["Spacelift Stack\nRuns Terraform\nEnforces OPA policy\n(cost-center tags · approved regions · no public LBs)"]

    TF["Terraform Module\nnetwork · cluster · node-pools · bootstrap\nProvisions cloud substrate\nInstalls ONE thing inside cluster"]

    ARGOCD["ArgoCD (in cluster)\napp-of-apps\nRegistered with BU hub\nInstalls everything else:\ningress · cert-manager · observability agents\npolicy controllers · secrets operator · apps"]

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
    Phase 1 - Spine · 1 BU    : BU hub cluster (dedicated)
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
                               : request env → Spacelift → hub + clusters + repo scaffold
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
| **Unit economics** | Cost per BU directly attributable · hub + cluster spend tagged per cost center |
