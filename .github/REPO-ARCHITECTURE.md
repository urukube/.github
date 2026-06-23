# Repository Architecture

![urukube Repository Calling Graph](assets/repo-calling-diagram.svg)

The platform is split across **eight GitHub repositories** in the `urukube` organisation, organised into four distinct layers:

---

## Layer 1 — Entry-Point Repos (calling repos)

**`orchestrator`** is the only environment-specific repo deployed today. It contains no Terraform code — only `orchestrator.tfvars` (all environment inputs) and two workflow entry points:

- `ci.yml` — `workflow_dispatch` that triggers a full provision (plan → approve → apply)
- `destroy.yml` — `workflow_dispatch` with a typed `destroy` confirmation gate; lets you tear down individual components or everything at once

The repo calls `orchestrator-plane-setup` via `workflow_call`, passing the tfvars path, S3 state bucket name, and approver list as inputs. It never touches Terraform directly.

---

## Layer 2 — Reusable Workflow Repo

**`orchestrator-plane-setup`** is checked out at runtime by the orchestrator workflow. It owns all Terraform logic in three subdirectories, executed in strict order:

| Stage | Directory | What it provisions | Calls |
|---|---|---|---|
| ① | `eks-infra/` | VPC + EKS cluster | `terraform-module-networking @v1.1.2` + `terraform-module-eks @v1.2.0` |
| ② | `eks-essential-addons/` | Cluster-critical add-ons | `terraform-module-essential-addons @v1.0.1` |
| ③ | `orchestrator-custom-addons/` | Platform tools (Istio, ArgoCD, Crossplane…) | `orchestrator-custom-addons @v1.1.12` |

Stages ② and ③ each read `data.terraform_remote_state.eks_infra` from S3 to obtain the cluster endpoint, OIDC provider ARN, VPC ID, and certificate authority data that the EKS modules emit. This makes the stages loosely coupled — they share state through S3, not through Terraform module composition.

Each stage runs through a `terraform-plan-apply` composite action that presents the plan for human approval before applying. The `destroy.yml` counterpart reverses the order: ③ → ② → ①.

---

## Layer 3 — Leaf Terraform Module Repos

These repos are pure, reusable Terraform modules. They are referenced via `git::https://github.com/urukube/<repo>.git?ref=vX.Y.Z` source blocks and are never called directly by CI — only consumed by `orchestrator-plane-setup`.

| Repo | What it builds | Key design choices |
|---|---|---|
| `terraform-module-networking` | AWS VPC · 3-tier subnets · NAT GW · VPC endpoints · SGs | `intra_subnets` for resource tier (no NAT route) · SG-to-SG rules for EKS→RDS · gateway VPC endpoints in all tiers |
| `terraform-module-eks` | EKS cluster · self-managed node group · launch template | IMDSv2 enforced · EBS encryption · bpffs mount for Cilium · `AL2023` nodeadm bootstrap |
| `terraform-module-essential-addons` | CoreDNS · VPC CNI · EBS CSI · AWS LB Controller · Cluster Autoscaler · Metrics Server · Pod Identity Agent | Essential cluster-critical add-ons only — no platform tooling |
| `orchestrator-custom-addons` | Istio · ArgoCD · Crossplane · ESO · Prometheus · Kiali · ECR pull role | Each add-on is toggle-gated (`enable_*` variable) · IRSA roles in a separate `-role.tf` file · mock-provider `terraform test` CI |

---

## Layer 4 — Shared Release Workflow

**`release-workflow`** is consumed by every other repo in the organisation. It provides semantic versioning via Conventional Commits — a `workflow_run` trigger fires after CI completes, and a failed CI blocks the release. All repos call it as:

```yaml
uses: urukube/release-workflow/.github/workflows/version.yml@v1
```

The repo contains three artefacts: `action.yml` (composite semantic-release action), `.github/workflows/version.yml` (the `workflow_call` entry point), and `semantic-version.sh` (the bash orchestration script, stored with mode `100755`).

---

## Calling Relationships at a Glance

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
