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

> **Owned and operated exclusively by the Platform team.** BU teams have no involvement in DR management — they are notified of status and experience no disruption to running workloads.

---

### Multi-Region Architecture

The Orchestrator Cluster runs as an **active/standby pair across two separate cloud regions**. This ensures that a full regional failure (AZ outages, cloud provider incidents, network partitions) cannot take down the platform control plane.

```mermaid
graph LR
    subgraph PRIMARY["🟢  PRIMARY REGION  (e.g. us-east-1)"]
        direction TB
        ORCH_PROD["ORCHESTRATOR — PROD\nCrossplane · ArgoCD · Backstage\nCRDs · Live traffic"]
    end

    subgraph DR_REGION["🟡  DR REGION  (e.g. us-west-2)"]
        direction TB
        ORCH_DR["ORCHESTRATOR — DR\nCrossplane · ArgoCD · Backstage\nWarm standby · No live traffic"]
    end

    subgraph BU_CLUSTERS["BU WORKLOAD CLUSTERS  (any region)"]
        direction TB
        BU1["BU-A  dev + prod"]
        BU2["BU-B  dev + prod"]
        BU3["BU-C  dev + prod"]
    end

    ORCH_PROD -->|manages| BU_CLUSTERS
    ORCH_PROD -.->|continuous state sync| ORCH_DR
    ORCH_DR -.->|ready to take over| BU_CLUSTERS

    style ORCH_PROD  fill:#4F46E5,stroke:#3730A3,color:#FFFFFF,font-weight:bold
    style ORCH_DR    fill:#818CF8,stroke:#4F46E5,color:#FFFFFF
    style BU1        fill:#0D9488,stroke:#0F766E,color:#FFFFFF
    style BU2        fill:#0D9488,stroke:#0F766E,color:#FFFFFF
    style BU3        fill:#0D9488,stroke:#0F766E,color:#FFFFFF
```

---

### What Gets Synced (Continuous Replication)

| Component | What is replicated | Mechanism |
| --------- | ------------------ | --------- |
| **Crossplane** | All Composite Resource Claims, provider configs, managed resource state | etcd backup + restore to DR; Crossplane re-adopts existing cloud resources on startup |
| **ArgoCD** | All Application and AppProject manifests, cluster registrations, RBAC | Git is the source of truth — DR ArgoCD re-syncs from the same Git repos on startup |
| **Backstage** | Service catalog, software templates, BU onboarding records | Database replication to DR region (read replica promoted on failover) |
| **Secrets** | All secrets remain in the cloud secret manager (e.g. AWS Secrets Manager) | Not stored on the cluster — DR reads from the same secret store |
| **CRD definitions** | All platform CRDs and composite resource definitions | Stored in Git — applied to DR cluster by ArgoCD bootstrap on startup |

> **Key principle:** Git is the ultimate source of truth. The DR cluster does not need to replicate in-memory state — it re-derives everything from Git and the cloud provider's own resource state.

---

### Failure Detection

The Platform team is responsible for monitoring Orchestrator health. Alerts fire on:

- Orchestrator API server unresponsive for > 2 minutes
- Crossplane controller not reconciling for > 5 minutes
- ArgoCD sync failures across all applications
- Backstage portal returning 5xx errors
- Cross-region health check (DR pings PROD every 30 seconds) failing

All alerts route to the **Platform team on-call** via PagerDuty (or equivalent). BU teams are never in the alert path for Orchestrator health.

---

### Failover Procedure (Platform Team Runbook)

```mermaid
flowchart TD
    ALERT["🚨 Alert fires\nPlatform on-call paged"]
    ASSESS["Platform team assesses\nIs PROD recoverable quickly?"]
    MINOR["Minor incident\nRestart pods / rollback\nNo failover needed"]
    DECLARE["Declare DR event\nEscalate to Platform lead\nOpen incident channel"]
    DNS["Update DNS / load balancer\nRoute traffic to DR region\nTarget: < 5 min"]
    PROMOTE["Promote DR cluster\nScale up Crossplane + ArgoCD\nEnable Backstage write mode"]
    VALIDATE["Platform team validates\nCrossplane reconciling\nArgoCD syncing from Git\nBackstage portal serving"]
    NOTIFY["Notify BU teams\n'Platform in DR mode — existing workloads unaffected.\nNew provisioning available shortly.'"]
    MONITOR["Monitor DR cluster\nContinue incident response\non PROD region"]
    RESTORE["Restore PROD region\nReprovision Orchestrator\nSync state from DR"]
    FAILBACK["Failback to PROD\nReverse DNS\nScale down DR"]

    ALERT --> ASSESS
    ASSESS -->|recoverable| MINOR
    ASSESS -->|not recoverable / SLA breach| DECLARE
    DECLARE --> DNS
    DNS --> PROMOTE
    PROMOTE --> VALIDATE
    VALIDATE --> NOTIFY
    NOTIFY --> MONITOR
    MONITOR --> RESTORE
    RESTORE --> FAILBACK

    style ALERT    fill:#DC2626,stroke:#991B1B,color:#FFFFFF,font-weight:bold
    style ASSESS   fill:#D97706,stroke:#B45309,color:#FFFFFF
    style MINOR    fill:#0D9488,stroke:#0F766E,color:#FFFFFF
    style DECLARE  fill:#DC2626,stroke:#991B1B,color:#FFFFFF
    style DNS      fill:#4F46E5,stroke:#3730A3,color:#FFFFFF
    style PROMOTE  fill:#4F46E5,stroke:#3730A3,color:#FFFFFF
    style VALIDATE fill:#4F46E5,stroke:#3730A3,color:#FFFFFF
    style NOTIFY   fill:#7C3AED,stroke:#5B21B6,color:#FFFFFF
    style MONITOR  fill:#D97706,stroke:#B45309,color:#FFFFFF
    style RESTORE  fill:#0D9488,stroke:#0F766E,color:#FFFFFF
    style FAILBACK fill:#0D9488,stroke:#0F766E,color:#FFFFFF
```

---

### Recovery Targets

| Metric | Target | Notes |
| ------ | ------ | ----- |
| **RTO** (Recovery Time Objective) | < 15 minutes | Time from alert to DR cluster serving traffic |
| **RPO** (Recovery Point Objective) | < 5 minutes | Maximum data loss (last Git commit + secret manager state) |
| **Failback RTO** | < 30 minutes | Time to restore PROD and cut back from DR |

---

### Impact During DR Mode

| What | Impact |
| ---- | ------ |
| **Running BU workloads** | None — BU dev/prod clusters run independently. Outage of the Orchestrator does not affect live application traffic. |
| **New environment provisioning** | Paused during failover window (< 15 min), then restored on DR cluster |
| **GitOps deployments** | Brief pause during failover, then ArgoCD on DR re-syncs from Git automatically |
| **Backstage portal** | Unavailable during failover window, then restored on DR cluster |

---

### Platform Team Responsibilities

| Responsibility | Owner |
| -------------- | ----- |
| Monitor Orchestrator health 24/7 | Platform on-call rotation |
| Execute failover runbook | Platform team lead + on-call engineer |
| Communicate status to BU teams | Platform team lead |
| Maintain and test DR cluster monthly | Platform SRE |
| Conduct quarterly failover drills | Platform team |
| Restore PROD and execute failback | Platform SRE |
| Post-incident review | Platform team + management |
