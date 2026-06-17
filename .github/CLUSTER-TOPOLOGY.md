# Cluster Topology

## Orchestrator Cluster

A single **Orchestrator Cluster** (platform team owns, PROD + DR) runs Crossplane, ArgoCD, and Backstage. It holds no BU workloads — its sole job is to provision and manage infrastructure for every customer on demand via Custom Resource Definitions (CRDs).

```mermaid
graph LR
    subgraph ORCH["🎛️ ORCHESTRATOR CLUSTER  —  platform team owns"]
        direction TB
        ORCH_PROD["PROD\nCrossplane · ArgoCD · Backstage\nCustom Resource Definitions"]
        ORCH_DR["DR\nactive standby"]
        ORCH_PROD -.->|failover| ORCH_DR
    end

    subgraph WORKLOADS["BU WORKLOAD CLUSTERS"]
        direction TB
        subgraph BUA["BU-A  (own cost center)"]
            direction TB
            A_DEV["dev cluster"]
            A_PROD["prod cluster"]
        end
        subgraph BUB["BU-B  (own cost center)"]
            direction TB
            B_DEV["dev cluster"]
            B_PROD["prod cluster"]
        end
        subgraph BUC["BU-C  (onboards later)"]
            direction TB
            C_DEV["dev cluster"]
            C_PROD["prod cluster"]
        end
    end

    ORCH_PROD -->|Crossplane provisions| A_DEV
    ORCH_PROD -->|Crossplane provisions| A_PROD
    ORCH_PROD -->|Crossplane provisions| B_DEV
    ORCH_PROD -->|Crossplane provisions| B_PROD
    ORCH_PROD -->|Crossplane provisions| C_DEV
    ORCH_PROD -->|Crossplane provisions| C_PROD

    classDef orchProd    fill:#4F46E5,stroke:#3730A3,color:#FFFFFF,font-weight:bold
    classDef orchDR      fill:#818CF8,stroke:#4F46E5,color:#FFFFFF
    classDef devCluster  fill:#0D9488,stroke:#0F766E,color:#FFFFFF
    classDef prodCluster fill:#D97706,stroke:#B45309,color:#FFFFFF

    class ORCH_PROD orchProd
    class ORCH_DR orchDR
    class A_DEV,B_DEV,C_DEV devCluster
    class A_PROD,B_PROD,C_PROD prodCluster
```

| What the platform runs | Clusters | Cost model |
| ---------------------- | -------- | ---------- |
| Orchestrator (PROD + DR) | 2 | Flat platform cost — does not scale with BU count |
| Per-BU workloads | 2 per BU (dev + prod) | BU cost center |

---

## Orchestrator Components

| Component | Role |
| --------- | ---- |
| **Crossplane** | Reconciles CRDs into cloud infrastructure (VPCs, clusters, IAM, node pools) |
| **ArgoCD** | Detects newly provisioned clusters and deploys app-of-apps (ingress, cert-manager, observability agents, policy controllers, secrets operator, apps) |
| **Backstage** | Self-service portal — BU engineers submit environment requests that generate CRDs |

---

## How a BU Gets an Environment

```mermaid
flowchart LR
    DEV["👩‍💻 BU Engineer\n━━━━━━━━━━━━━━\n'I need a Kubernetes\nenvironment'"]

    PORTAL["Backstage Portal\n━━━━━━━━━━━━━━\nHosted on Orchestrator\nname · size · region\ncost-center"]

    CRD["Environment CRD\n━━━━━━━━━━━━━━\nApplied to Orchestrator\ncost-center tag\napproved region\nno public LBs"]

    CROSSPLANE["Crossplane\n━━━━━━━━━━━━━━\nReconciles the CRD\nProvisions cloud infra\nVPC · cluster\nnode-pools · IAM"]

    ARGOCD["ArgoCD\n━━━━━━━━━━━━━━\nDetects new cluster\nDeploys app-of-apps\ningress · cert-manager\nobservability · policy\nsecrets operator · apps"]

    DEV --> PORTAL --> CRD --> CROSSPLANE --> ARGOCD

    style DEV        fill:#1D4ED8,stroke:#1E3A8A,color:#FFFFFF,font-weight:bold
    style PORTAL     fill:#7C3AED,stroke:#5B21B6,color:#FFFFFF
    style CRD        fill:#4F46E5,stroke:#3730A3,color:#FFFFFF
    style CROSSPLANE fill:#0D9488,stroke:#0F766E,color:#FFFFFF
    style ARGOCD     fill:#D97706,stroke:#B45309,color:#FFFFFF
```

**Principle: "CRDs declare intent · Crossplane provisions infra · ArgoCD delivers everything else"**

---

## DR Strategy

The Orchestrator Cluster runs in an active/standby configuration:

- **PROD** handles all live traffic — CRD reconciliation, GitOps sync, portal serving.
- **DR** is a hot standby, continuously synced. In the event of a PROD failure, failover is triggered and the DR instance takes over without re-provisioning BU clusters (workloads keep running independently).

> BU workload clusters are independent of the Orchestrator. A failure of the Orchestrator pauses new provisioning and deployments but does not affect running BU workloads.
