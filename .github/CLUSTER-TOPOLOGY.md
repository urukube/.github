# Cluster Topology

## Orchestrator Cluster

A single **Orchestrator Cluster** (platform team owns, PROD + DR) runs Crossplane, ArgoCD, and Backstage. It holds no BU workloads — its sole job is to provision and manage infrastructure for every customer on demand via Custom Resource Definitions (CRDs).

```mermaid
graph TD
    subgraph ORCH["🎛️ ORCHESTRATOR CLUSTER  (platform team owns)"]
        ORCH_PROD["PROD\nCrossplane · ArgoCD · Backstage\nCustom Resource Definitions"]
        ORCH_DR["DR  (active standby)"]
        ORCH_PROD -.->|failover| ORCH_DR
    end

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

    ORCH_PROD -->|Crossplane provisions| A_DEV
    ORCH_PROD -->|Crossplane provisions| A_PROD
    ORCH_PROD -->|Crossplane provisions| B_DEV
    ORCH_PROD -->|Crossplane provisions| B_PROD
    ORCH_PROD -->|Crossplane provisions| C_DEV
    ORCH_PROD -->|Crossplane provisions| C_PROD
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
flowchart TD
    DEV["👩‍💻 BU Engineer\n'I need a Kubernetes environment'"]

    PORTAL["Backstage Portal\n(hosted on Orchestrator Cluster)\nBU fills in: name · size · region · cost-center"]

    CRD["Environment CRD applied\nto Orchestrator Cluster\nPolicy gates: cost-center tag · approved region · no public LBs"]

    CROSSPLANE["Crossplane\n(on Orchestrator)\nReconciles the CRD\nProvisions cloud infra:\nVPC · cluster · node-pools · IAM"]

    ARGOCD["ArgoCD\n(on Orchestrator)\nDetects new cluster\nDeploys app-of-apps:\ningress · cert-manager · observability agents\npolicy controllers · secrets operator · apps"]

    DEV --> PORTAL --> CRD --> CROSSPLANE --> ARGOCD
```

**Principle: "CRDs declare intent · Crossplane provisions infra · ArgoCD delivers everything else"**

---

## DR Strategy

The Orchestrator Cluster runs in an active/standby configuration:

- **PROD** handles all live traffic — CRD reconciliation, GitOps sync, portal serving.
- **DR** is a hot standby, continuously synced. In the event of a PROD failure, failover is triggered and the DR instance takes over without re-provisioning BU clusters (workloads keep running independently).

> BU workload clusters are independent of the Orchestrator. A failure of the Orchestrator pauses new provisioning and deployments but does not affect running BU workloads.
