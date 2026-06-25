# Repository Architecture

![urukube Repository Calling Graph](assets/repo-calling-diagram.svg)

The platform is split across **eleven GitHub repositories** in the `urukube` organisation, organised into four distinct layers:

---

## Layer 1 — Entry-Point Repos (calling repos)

**`orchestrator`** is the only environment-specific repo deployed today. It contains no Terraform code — only `orchestrator.tfvars` (all environment inputs) and two workflow entry points:

- `ci.yml` — `workflow_dispatch` that triggers a full provision (plan → approve → apply)
- `destroy.yml` — `workflow_dispatch` with a typed `destroy` confirmation gate; lets you tear down individual components or everything at once

The repo calls `orchestrator-plane-setup` via `workflow_call`, passing the tfvars path, S3 state bucket name, and approver list as inputs. It never touches Terraform directly.

---

## Layer 2 — Reusable Workflow Repo

**`orchestrator-plane-setup`** is checked out at runtime by the orchestrator workflow. It owns all Terraform logic in six subdirectories, executed in strict sequential order:

| Stage | Directory | What it provisions | Calls |
|---|---|---|---|
| ① | `orchestrator-secret-management/` | AWS Secrets Manager secrets | `orchestrator-secret-management @v1.2.1` |
| ② | `eks-infra/` | VPC + EKS cluster | `terraform-module-networking @v1.1.2` + `terraform-module-eks @v1.2.0` |
| ③ | `eks-essential-addons/` | Cluster-critical add-ons | `terraform-module-essential-addons @v1.0.1` |
| ④ | `orchestrator-custom-addons/` | Platform tools (Istio, ArgoCD, Crossplane, ESO Helm…) | `orchestrator-custom-addons @v1.3.1` |
| ⑤ | `eso-configuration/` | ESO ClusterSecretStore + ExternalSecrets | `orchestrator-eso-config @v1.0.1` |
| ⑥ | `argocd-configuration/` | ArgoCD Apps + ApplicationSets | `argocd-configuration @v1.2.0` |

**Stage ①** is AWS-only and has no remote state dependency — it provisions Secrets Manager secrets before the cluster exists, so their ARNs are available to later stages.

**Stages ③–⑥** each read `data.terraform_remote_state.eks_infra` from S3 to obtain the cluster endpoint, OIDC provider ARN, VPC ID, and certificate authority data that stage ② emits. This makes the stages loosely coupled — they share state through S3, not through Terraform module composition.

**Stage ⑤ must run after stage ④.** The kubectl provider discovers CRDs at plan time. If the ESO Helm chart has not been applied yet (stage ④), the `ClusterSecretStore` and `ExternalSecret` kinds are unknown and the plan fails. The pipeline ordering enforces this constraint.

Each stage runs through the `terraform-plan-apply` composite action (init → plan → approve gate → apply). The `destroy.yml` counterpart reverses the order: ⑥ → ⑤ → ④ → ③ → ② → ①.

`orchestrator-plane-setup` also exposes three granular lower-level actions for use cases where plan and apply need to run in separate CI jobs:

| Action | What it does |
|---|---|
| `.github/actions/plan-tf/` | `terraform init` + `terraform plan` only |
| `.github/actions/apply-tf/` | `terraform init` + `terraform apply --auto-approve` |
| `.github/actions/destroy-tf/` | `terraform init` + `terraform destroy --auto-approve` |

The combined `terraform-plan-apply` and `terraform-plan-destroy` actions remain available for the standard human-gated provision/destroy flow.

---

## Layer 3 — Leaf Terraform Module Repos

These repos are pure, reusable Terraform modules. They are referenced via `git::https://github.com/urukube/<repo>.git?ref=vX.Y.Z` source blocks and are never called directly by CI — only consumed by `orchestrator-plane-setup`.

| Repo | What it builds | Key design choices |
|---|---|---|
| `orchestrator-secret-management` | AWS Secrets Manager secrets (bulk) | `secretlist` map for paths/descriptions (safe to commit); `secret_values` injected at CI runtime via `TF_VAR_secret_values`; `ignore_changes` on secret versions after initial creation |
| `terraform-module-networking` | AWS VPC · 3-tier subnets · NAT GW · VPC endpoints · SGs | `intra_subnets` for resource tier (no NAT route) · SG-to-SG rules for EKS→RDS · gateway VPC endpoints in all tiers |
| `terraform-module-eks` | EKS cluster · self-managed node group · launch template | K8s 1.36 · IMDSv2 enforced · EBS encryption · bpffs mount for Cilium · AL2023 nodeadm bootstrap · port 15017 open when `enable_istio=true` |
| `terraform-module-essential-addons` | CoreDNS · VPC CNI · EBS CSI · AWS LB Controller · Cluster Autoscaler · Metrics Server · Pod Identity Agent | Essential cluster-critical add-ons only — no platform tooling |
| `orchestrator-custom-addons` | Istio · ArgoCD · Crossplane · ESO Helm · Prometheus · Kiali · ECR pull role · Image Updater · ECR CronJob | Each add-on is toggle-gated (`enable_*` variable) · IRSA roles in a separate `-role.tf` file · NLB cross-zone + backend SG management · mock-provider `terraform test` CI |
| `orchestrator-eso-config` | ESO `ClusterSecretStore` + `ExternalSecret` resources | Uses `external-secrets.io/v1` API (not `v1beta1`) · `server_side_apply = true` · must run after ESO Helm install (stage ④) |
| `argocd-configuration` | ArgoCD `Application: platform-tenant-registry` + `ApplicationSet: platform-custom-xrds` | Two resources: (1) fixed Application pointing at `platform-tenant-registry` repo for Crossplane EnvironmentConfigs; (2) SCM Provider ApplicationSet that auto-discovers repos tagged `platform-custom-xrds` and creates one Application per repo · both sync to `crossplane-system` · `kubectl` provider only |

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
              ┌───────┬──────────┬────┴──────────────┬──────────────┬────────────────┐
              ▼       ▼          ▼                   ▼              ▼                ▼
         ①secret   ②eks-    ③eks-essential-   ④orchestrator-  ⑤eso-        ⑥argocd-
          -mgmt    infra       addons           custom-addons  configuration configuration
              │    ╱  ╲          │                   │              │                │
              ▼   ▼    ▼         ▼                   ▼              ▼                ▼
         secret- net-  eks-  essential-         custom-          eso-           argocd-
         mgmt   working module  addons            addons         config         config
         module        module   module            module         module         module

All repos  ──workflow_run──▶  release-workflow  (semantic versioning)
```

### `eso-configuration` — Why it is a Separate Stage

Once ESO is running on the orchestrator cluster (Helm chart installed by stage ④), stage ⑤ creates the `ClusterSecretStore` and `ExternalSecret` resources that wire ESO to AWS Secrets Manager. The split exists because the kubectl provider caches API discovery at plan time — if `ClusterSecretStore` and `ExternalSecret` CRDs are not yet registered, Terraform cannot plan the resources. Running them in a separate stage after the Helm install resolves this chicken-and-egg dependency.

### `argocd-configuration` — What it Does

Once ArgoCD is running on the orchestrator cluster (installed by stage ④), stage ⑥ creates **two** ArgoCD resources:

**① `Application: platform-tenant-registry`**

A fixed Application pointing at the `platform-tenant-registry` repo. This repo holds Crossplane `EnvironmentConfig` resources (org-wide defaults + per-BU metadata). ArgoCD syncs it continuously to `crossplane-system` with `recurse: true` so new tenant directories are picked up automatically.

- Do **not** add the `platform-custom-xrds` GitHub topic to this repo — it is managed via a fixed Application, not the SCM generator.
- Adding a new BU = add one file under `tenants/<bu-name>/environmentconfig.yaml` with the required `bu-id` label.

**② `ApplicationSet: platform-custom-xrds`**

An ApplicationSet that turns the whole GitHub org into a self-registering XRD registry:

1. The SCM Provider generator authenticates to GitHub using the `argocd-github-token` Kubernetes secret (synced into the `argocd` namespace by stage ⑤ via ESO)
2. It scans every repo in the `urukube` org for the GitHub topic `platform-custom-xrds`
3. For each matching repo it creates one ArgoCD `Application` named `xrd-<repo-name>`
4. Each Application syncs from `main` branch, path `.`, into the `crossplane-system` namespace with `prune: true` and `selfHeal: true`

Adding new Crossplane XRDs or Compositions requires only: create a repo, add the `platform-custom-xrds` topic, push the manifests — ArgoCD picks it up automatically with no manual wiring.
