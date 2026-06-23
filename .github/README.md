# Internal Developer Platform (IDP)

## What We Are Building

A **greenfield Internal Developer Platform** that gives every Business Unit (BU) a self-service, opinionated way to get production-grade Kubernetes environments — without doing raw infrastructure work.

A BU asks for an environment. The platform provisions it. Everything in between is abstracted behind **golden paths** (paved roads that make the right way the easy way) and **guardrails** (policy enforcement that makes the wrong way impossible).

This is operated by a **central platform team**, centrally funded, with BUs billed only for their own workload clusters (dev + prod). The orchestrator cluster is platform cost.

---

## How It Works

![urukube IDP — Provisioning and Delivery Flows](assets/idp-flow.svg)

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

![Cluster Topology](assets/cluster-topology.svg)

> Full topology diagram, component breakdown, and DR strategy: [CLUSTER-TOPOLOGY.md](CLUSTER-TOPOLOGY.md)

---

## GitOps Promotion Flow

![GitOps Promotion Flow](assets/gitops-promotion.svg)

> Full flow diagram, stage breakdown, and key principles: [GITOPS-PROMOTION.md](GITOPS-PROMOTION.md)

---

## Repository Architecture

![urukube Repository Calling Graph](assets/repo-calling-diagram.svg)

The platform is split across **eight GitHub repositories** in the `urukube` organisation, organised into four distinct layers:

### Layer 1 — Entry-Point Repos (calling repos)

**`orchestrator`** is the only environment-specific repo deployed today. It contains no Terraform code — only `orchestrator.tfvars` (all environment inputs) and two workflow entry points:

- `ci.yml` — `workflow_dispatch` that triggers a full provision (plan → approve → apply)
- `destroy.yml` — `workflow_dispatch` with a typed `destroy` confirmation gate; lets you tear down individual components or everything at once

The repo calls `orchestrator-plane-setup` via `workflow_call`, passing the tfvars path, S3 state bucket name, and approver list as inputs. It never touches Terraform directly.

### Layer 2 — Reusable Workflow Repo

**`orchestrator-plane-setup`** is checked out at runtime by the orchestrator workflow. It owns all Terraform logic in three subdirectories, executed in strict order:

| Stage | Directory | What it provisions | Calls |
|---|---|---|---|
| ① | `eks-infra/` | VPC + EKS cluster | `terraform-module-networking @v1.1.2` + `terraform-module-eks @v1.2.0` |
| ② | `eks-essential-addons/` | Cluster-critical add-ons | `terraform-module-essential-addons @v1.0.1` |
| ③ | `orchestrator-custom-addons/` | Platform tools (Istio, ArgoCD, Crossplane…) | `orchestrator-custom-addons @v1.1.12` |

Stages ② and ③ each read `data.terraform_remote_state.eks_infra` from S3 to obtain the cluster endpoint, OIDC provider ARN, VPC ID, and certificate authority data that the EKS modules emit. This makes the stages loosely coupled — they share state through S3, not through Terraform module composition.

Each stage runs through a `terraform-plan-apply` composite action that presents the plan for human approval before applying. The `destroy.yml` counterpart reverses the order: ③ → ② → ①.

### Layer 3 — Leaf Terraform Module Repos

These repos are pure, reusable Terraform modules. They are referenced via `git::https://github.com/urukube/<repo>.git?ref=vX.Y.Z` source blocks and are never called directly by CI — only consumed by `orchestrator-plane-setup`.

| Repo | What it builds | Key design choices |
|---|---|---|
| `terraform-module-networking` | AWS VPC · 3-tier subnets · NAT GW · VPC endpoints · SGs | `intra_subnets` for resource tier (no NAT route) · SG-to-SG rules for EKS→RDS · gateway VPC endpoints in all tiers |
| `terraform-module-eks` | EKS cluster · self-managed node group · launch template | IMDSv2 enforced · EBS encryption · bpffs mount for Cilium · `AL2023` nodeadm bootstrap |
| `terraform-module-essential-addons` | CoreDNS · VPC CNI · EBS CSI · AWS LB Controller · Cluster Autoscaler · Metrics Server · Pod Identity Agent | Essential cluster-critical add-ons only — no platform tooling |
| `orchestrator-custom-addons` | Istio · ArgoCD · Crossplane · ESO · Prometheus · Kiali · ECR pull role | Each add-on is toggle-gated (`enable_*` variable) · IRSA roles in a separate `-role.tf` file · mock-provider `terraform test` CI |

### Layer 4 — Shared Release Workflow

**`release-workflow`** is consumed by every other repo in the organisation. It provides semantic versioning via Conventional Commits — a `workflow_run` trigger fires after CI completes, and a failed CI blocks the release. All repos call it as:

```yaml
uses: urukube/release-workflow/.github/workflows/version.yml@v1
```

The repo contains three artefacts: `action.yml` (composite semantic-release action), `.github/workflows/version.yml` (the `workflow_call` entry point), and `semantic-version.sh` (the bash orchestration script, stored with mode `100755`).

### Calling Relationships at a Glance

```
orchestrator  ──workflow_call──▶  orchestrator-plane-setup
                                      │
                              ┌───────┼───────────────────────┐
                              ▼       ▼                       ▼
                         eks-infra  eks-essential-addons  orchestrator-custom-addons
                              │          │                     │
                    ┌─────────┤          │                     │
                    ▼         ▼          ▼                     ▼
             tf-module-  tf-module-  tf-module-        orchestrator-
             networking    eks      essential-addons   custom-addons

All repos  ──workflow_run──▶  release-workflow  (semantic versioning)
```

---

## The Economic Story

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

---

## Five CNCF Platform Planes

![Five CNCF Platform Planes](assets/cncf-planes.svg)

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
