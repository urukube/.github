# IDP — Internal Developer Platform

> **For Claude**: This is a living document. Update it at the end of every session whenever a new repo is created, an architectural decision is made, a convention is established, or a bug with design implications is fixed. The user should never have to repeat context.



## The Goal

We are building a **greenfield, organisation-level Internal Developer Platform (IDP)** for the **urukube** organisation.

The platform gives every Business Unit (BU) a self-service, opinionated way to get production-grade Kubernetes environments — without doing raw infrastructure work. A BU asks for an environment; the platform provisions it. Everything in between is abstracted behind **golden paths** (paved roads that make the right way the easy way) and **guardrails** (policy enforcement that makes the wrong way impossible).

The platform is operated by a **central platform team**, centrally funded. BUs are billed only for their own workload clusters (dev + prod). The orchestrator cluster is platform cost.

---

## The Five CNCF Platform Planes

| Plane | Key Tools |
|---|---|
| **Developer Control** | Backstage (self-service portal, golden-path templates, service catalog) |
| **Integration & Delivery** | GitHub (VCS/CI) · Crossplane (CRD-driven infra) · JFrog Artifactory · ArgoCD · Kargo · Argo Rollouts |
| **Resource** | Orchestrator Cluster (PROD + DR) · per-BU dev cluster · per-BU prod cluster |
| **Observability** *(cross-cutting)* | Prometheus · Thanos · Grafana · OpenTelemetry · Loki · Tempo |
| **Security** *(cross-cutting)* | ESO · OIDC · SPIFFE/SPIRE · Kyverno · OPA/Gatekeeper · Cilium · cosign · SBOM |

---

## Cluster Topology

**One Orchestrator Cluster (PROD + DR)** runs Crossplane, ArgoCD, and Backstage. It holds no BU workloads — its sole job is to provision and manage infrastructure for every tenant on demand via CRDs.

- Crossplane reconciles CRDs → cloud infra (VPCs, clusters, IAM, node pools)
- ArgoCD detects newly provisioned clusters and deploys app-of-apps
- Backstage is the self-service portal where BU engineers submit environment requests

Each BU gets 2 clusters: dev + prod. Provisioned entirely by Crossplane on the Orchestrator.

**DR Strategy**: Orchestrator runs active/standby across two AWS regions. Git is the source of truth — the DR cluster re-derives everything from Git and cloud provider state on promotion. RTO < 15 min, RPO < 5 min.

Full details: `.github/.github/CLUSTER-TOPOLOGY.md` and `.github/.github/GITOPS-PROMOTION.md`

---

## Phased Roadmap

| Phase | Focus | Status |
|---|---|---|
| **Phase 0** | Landing zone · network hub-spoke · identity/SSO · Git & IaC layout | Foundation |
| **Phase 1** | Orchestrator cluster · Crossplane + ArgoCD + Backstage · pilot BU dev+prod · reference app | **TODAY** |
| **Phase 2** | Spacelift + OPA · ApplicationSets · Kargo promotion · Argo Rollouts | Next |
| **Phase 3** | Org-wide observability · ESO · Kyverno · cosign/SBOM · Cilium network policy | Later |
| **Phase 4** | Full self-service Backstage golden path (stable after 2-3 BUs) | Later |

---

## What Has Been Built

### GitHub Organisation: `urukube`

#### `.github` repo (org-level documentation)
- `CLUSTER-TOPOLOGY.md` — orchestrator topology, DR strategy, failover runbook
- `GITOPS-PROMOTION.md` — GitOps promotion flow (dev → prod)
- `README.md` — full IDP overview: five planes, economic story, phased roadmap, toolchain, KPIs

#### `release-workflow` repo → `https://github.com/urukube/release-workflow`
Composite GitHub Action + reusable workflow for semantic versioning across all org repos.
- `action.yml` — core composite action (semantic-release, PR title validation, version outputs)
- `.github/workflows/version.yml` — reusable `workflow_call` wrapper; other repos call this via `uses: urukube/release-workflow/.github/workflows/version.yml@v1`
- `.github/workflows/main.yml` — CI for the release-workflow repo itself (pre-commit, self-test, check-workflow-sync)
- `semantic-version.sh` — bash orchestration script (must stay `chmod +x` / mode `100755` in git)

**Convention**: All repos in the org should use `urukube/release-workflow/.github/workflows/version.yml@v1` for releases — not `cycjimmy/semantic-release-action` or any other third-party action.

#### `terraform-module-networking` repo → `https://github.com/urukube/terraform-module-networking`
Terraform module for the AWS networking layer that hosts the **Orchestrator Cluster**.

**Subnet architecture — 3-tier design across 3 AZs:**

| Tier | Purpose | CIDR (AZ-A / AZ-B / AZ-C) | Routing |
|---|---|---|---|
| **EKS Node** (private) | EKS worker nodes and pods | `10.0.0.0/19` · `10.0.32.0/19` · `10.0.64.0/19` | → NAT GW |
| **Resource** (private/intra) | RDS, ElastiCache, MSK, etc. | `10.0.96.0/19` · `10.0.128.0/19` · `10.0.160.0/19` | VPC-local only |
| **Public** | NAT Gateways + LB ENIs | `10.0.192.0/24` · `10.0.193.0/24` · `10.0.194.0/24` | → IGW |

Key design decisions:
- Resource subnets use `intra_subnets` in the VPC module (no NAT GW route — managed services don't need outbound internet)
- EKS nodes can reach resource tier by default via a dedicated `aws_resources` security group that allows all inbound from the `eks_nodes` SG
- Gateway VPC endpoints (S3, DynamoDB) have routes injected into all three subnet tiers including resource subnets
- ALB and NLB are **single cross-AZ resources** — one LB spans all AZs via ENIs placed in each public subnet. NAT Gateways are the only truly per-AZ resource.
- Public subnet CIDRs start at `.192` to avoid overlap with the two `/19` private tiers (which cover `.0.0` to `.191.255`)
- Module uses `terraform-aws-modules/vpc ~> 5.5.0` with AWS provider `>= 6.42.0`

Outputs consumed by the EKS module: `eks_node_subnet_ids`, `resource_subnet_ids`, `node_security_group_id`, `aws_resources_security_group_id`, `vpc_id`

- Lock file (`.terraform.lock.hcl`) is **committed** in this module (pinned to `6.52.0`)
- All 11 terraform tests pass (alb, nlb, security_groups, vpc, vpc_endpoints)

#### `terraform-module-eks` repo → `https://github.com/urukube/terraform-module-eks`
Terraform module for provisioning EKS clusters for the Orchestrator and BU workload clusters.

- Uses `terraform-aws-modules/eks ~> 21.0` with AWS provider `>= 6.42.0`
- **Self-managed node groups** are the current default (`is_eks_managed_node_group = false`)
- EKS managed node group path (`is_eks_managed_node_group = true`) is stubbed with a TODO — not yet implemented
- Lock file (`.terraform.lock.hcl`) is **committed** in this module (pinned to `6.52.0`)

Key design decisions:
- A dedicated `aws_launch_template` (`lt.tf`) is wired into the self-managed node group — it owns the EC2-level config: EBS encryption, IMDSv2 (`http_tokens = required`), CloudWatch monitoring, no public IP, security groups. The node group block owns bootstrap/userdata and ASG sizing only.
- `aws_vpc_security_group_ingress_rule` with `for_each` over subnet CIDRs — one rule per CIDR (modern split resource, not `aws_security_group_rule`)
- Cluster name pattern: `{friendly_name}-eks`
- Self-managed node group name pattern: `{friendly_name}-{bu_id}-{app_id}-sm`
- `ami_type` drives bootstrap method: `AL2023_x86_64_STANDARD` uses nodeadm/cloudinit; `AL2_x86_64` uses legacy bootstrap args
- `cluster_asg_names` output derived from `module.eks.self_managed_node_groups` map (returns `[]` automatically when managed node groups are used)
- `pre-bootstrap-user-data.sh` mounts bpffs for Cilium/eBPF tools

#### `terraform-module-essential-addons` repo → `https://github.com/urukube/terraform-module-essential-addons`
Terraform module for the cluster-critical (essential) add-ons that every EKS cluster needs regardless of workload type. Consumed by `orchestrator-plane-setup/eks-essential-addons/` at `ref=v1.0.1`.

**Add-ons installed:**
- CoreDNS — cluster DNS
- VPC CNI — pod networking
- EBS CSI Driver — persistent volume support
- AWS Load Balancer Controller — ALB/NLB provisioning
- Cluster Autoscaler — node autoscaling
- Metrics Server — resource metrics API
- Pod Identity Agent — EKS Pod Identity (alternative to IRSA)

Key design decisions:
- All add-ons are optional via toggle variables (`enable_*`) — the module is composable
- Cluster name, endpoint, OIDC ARN, VPC ID are passed in as inputs (consumed from `eks_infra` remote state by the calling root module)
- Uses `terraform test` with mock providers for CI (no real AWS credentials needed)

#### `orchestrator-plane-setup` repo → `https://github.com/urukube/orchestrator-plane-setup`
Reusable Terraform root configuration that wires together the networking and EKS modules to provision a full cluster environment. Formerly named `terraform-module-eks-setup` (renamed on GitHub). Called via `workflow_call` by environment repos such as `orchestrator`. **Latest release: `v1.8.5`.**

Contains six subdirectory root modules executed in strict sequential order:
1. `orchestrator-secret-management/` → AWS-only; calls `orchestrator-secret-management @v1.2.1`; no remote state dependency. Secret names from `secrets.auto.tfvars.json`, secret values from `TF_VAR_secret_values` injected via step-level `env:` in the workflow (from `ARGO_ADMIN_SECRET` and `PLATFORM_GITHUB_PAT` org secrets)
2. `eks-infra/` → calls `terraform-module-networking @v1.1.2` + `terraform-module-eks @v1.2.0`; writes outputs to S3 remote state
3. `eks-essential-addons/` → reads `eks-infra` S3 state; calls `terraform-module-essential-addons @v1.0.1`
4. `orchestrator-custom-addons/` → reads `eks-infra` S3 state; calls `orchestrator-custom-addons @v1.3.2`; installs ESO Helm chart (CRDs registered here)
5. `eso-configuration/` → reads `eks-infra` S3 state; calls `orchestrator-eso-config @v1.0.1`; creates ClusterSecretStore + ExternalSecrets (must run AFTER orchestrator-custom-addons so ESO CRDs are registered)
6. `argocd-configuration/` → reads `eks-infra` S3 state; calls `argocd-configuration @v1.3.0`; ApplicationSet + platform-tenant-registry Application

- Calls `urukube/terraform-module-networking` at `ref=v1.1.2`, `urukube/terraform-module-eks` at `ref=v1.2.0`, `urukube/terraform-module-essential-addons` at `ref=v1.0.1`, `urukube/orchestrator-custom-addons` at `ref=v1.3.2`, `urukube/orchestrator-eso-config` at `ref=v1.0.1`, `urukube/argocd-configuration` at `ref=v1.3.0`, and `urukube/orchestrator-secret-management` at `ref=v1.2.1` — update these pins when consuming new releases
- `CKV_TF_1` and `CKV_TF_2` checkov skips added on all module blocks — using version tags rather than commit hashes
- `eks-essential-addons` is built and wired into both `main.yml` and `destroy.yml`; `eks-custom-addons` is not yet built
- `orchestrator-secret-management` has no `data.tf` or `get_env.sh` — it is AWS-only with no Kubernetes or remote state dependency. Secret values are scoped to the step via `env:` (not job-level `GITHUB_ENV`) to avoid leaking `TF_VAR_secret_values` to subsequent Terraform steps
- **ESO CRD chicken-and-egg**: The kubectl provider caches API discovery at plan time. If ESO CRDs are not registered when a component initialises, it fails. Fix: put ClusterSecretStore/ExternalSecret resources in a SEPARATE pipeline component (`eso-configuration/`) that runs after the ESO Helm install step (`orchestrator-custom-addons/`).
- Resource subnets are NOT passed in — networking module auto-calculates them from VPC CIDR and exposes via `resource_subnet_ids` / `resource_subnet_cidrs` outputs
- Variable name: `eks_node_subnets` (not `private_subnets`) — matches networking module's variable name
- Networking module outputs consumed: `eks_node_subnet_ids`, `eks_node_subnet_cidrs`, `vpc_id`, `node_security_group_id`, `control_plane_security_group_id`
- Release workflow gates on `pre-commit` workflow (no dedicated CI workflow in this repo — `main.yml` and `destroy.yml` are `workflow_call` reusable workflows, not CI)
- Lock file committed at root and in `eks-infra/` subdirectory
- terraform-docs pre-commit arg must be `../README.md` (not `README.md`) because docs are generated from the `eks-infra/` subdirectory, not the repo root

#### `orchestrator` repo → `https://github.com/urukube/orchestrator`
Environment-specific calling repo for the Orchestrator Cluster. Contains no Terraform code — only environment inputs and workflow entry points.

- `orchestrator.tfvars` — all inputs for the Orchestrator cluster (region, VPC CIDR, AZs, node sizing, endpoint access CIDRs, etc.)
- `ci.yml` — `workflow_dispatch` that calls `urukube/orchestrator-plane-setup/.github/workflows/main.yml@v1` to provision the cluster
- `destroy.yml` — `workflow_dispatch` with confirmation gate (`destroy` typed by user) that calls the destroy workflow; component choices are `argocd_configuration`, `eso_configuration`, `orchestrator_secret_management`, `orchestrator_custom_addons`, `eks_essential_addons`, `eks_infra`, and `all`
- State bucket: `urukube-orchestrator-tfstate`, S3 key prefix: `orchestrator`
- Pattern: **calling repos contain only tfvars + workflow entry points**; all Terraform logic lives in the reusable setup repo

**Valid `vpc_endpoints` keys** (from `terraform-module-networking`): `ssm`, `ssmmessages`, `ec2messages`, `kms`, `ecr_api`, `ecr_dkr`, `ec2`, `sts`, `logs`, `s3`, `dynamodb`. The key `eks` is NOT valid — do not use it.

#### `orchestrator-secret-management` repo → `https://github.com/urukube/orchestrator-secret-management`
Terraform module that bulk-provisions AWS Secrets Manager secrets. Consumed by `orchestrator-plane-setup/orchestrator-secret-management/` at `ref=v1.2.1`.

- **`secretlist`** variable: `map(object({ name = string, description = optional(string, "") }))` — key is the stable Terraform logical identifier, value holds the SM path and description. Committed in `secrets.auto.tfvars.json` in the setup component (auto-loaded by Terraform)
- **`secret_values`** variable: `map(string)`, sensitive — injected at CI runtime as `TF_VAR_secret_values` from GitHub org secrets; never stored in tfvars files. Creates `aws_secretsmanager_secret_version` for each key present in both `secretlist` and `secret_values`
- Outputs: `secret_arns` and `secret_names` (both maps of logical key → value)
- Only provider needed: `hashicorp/aws >= 6.42.0`
- Platform secrets provisioned: `platform/argocd/admin-password` (from `ARGO_ADMIN_SECRET`) and `platform/github/github-token` (from `PLATFORM_GITHUB_PAT` org secret — must be a long-lived PAT with `repo` read + `read:org` scopes; `GITHUB_TOKEN` is reserved by GitHub for the ephemeral workflow token and cannot be used here)
- `secrets.auto.tfvars.json` only contains SM paths and descriptions (no values) — safe to commit even though `.gitignore` blocks `*.tfvars.json`; force-add with `git add -f`
- CI uses `mock_provider "aws"` — no AWS credentials needed
- **`ignore_changes = [secret_string]`** is set on `aws_secretsmanager_secret_version` — Terraform only sets the value on first creation. Subsequent value changes (e.g. rotating the ArgoCD password) must be pushed to Secrets Manager directly via `aws secretsmanager put-secret-value`, then force-resync ESO with `kubectl annotate externalsecret <name> -n <ns> force-sync=$(date +%s) --overwrite`
- **`ARGO_ADMIN_SECRET` must be stored as a bcrypt hash**, not plaintext — ArgoCD reads `admin.password` as a bcrypt hash and will reject logins if it is plaintext. Generate with: `kubectl exec -n argocd deploy/argocd-server -- argocd account bcrypt --password <password>`
- **`PLATFORM_GITHUB_PAT` is stored as plaintext** — ArgoCD uses it directly as a Bearer token; no transformation needed

#### `orchestrator-eso-config` repo → `https://github.com/urukube/orchestrator-eso-config`
Terraform module that configures ESO on the orchestrator cluster. Consumed by `orchestrator-plane-setup/eso-configuration/` at `ref=v1.0.1`. **Public repo.**

- Creates one `ClusterSecretStore` (`aws-secrets-manager`) pointing to AWS Secrets Manager, using ESO's JWT-based IRSA auth via the ESO service account
- Creates one `ExternalSecret` per entry in `var.external_secrets` — each maps SM paths → Kubernetes secret keys
- **Must run in a separate pipeline step AFTER the ESO Helm install** — kubectl provider discovers CRDs at plan time; if ESO Helm hasn't been applied yet, `ClusterSecretStore` and `ExternalSecret` kinds are unknown and the plan fails
- All resources use `server_side_apply = true` to bypass client-side schema validation of CRD-backed types
- ESO CRDs are served at `external-secrets.io/v1` (not `v1beta1`) — ESO >= 0.10 promoted the API; never use `v1beta1`
- Platform ExternalSecrets defined in `orchestrator.tfvars`: `argocd-github-token` (→ `argocd-github-token` K8s secret, `Owner` policy) and `argocd-admin-password` (→ `argocd-secret` K8s secret, `Merge` policy — ArgoCD reads `admin.password` as bcrypt hash)
- Variable: `external_secrets` is a `map(object({namespace, target_secret_name, creation_policy, mappings}))` — `creation_policy="Merge"` for existing secrets, `"Owner"` for new ones
- Only provider needed: `gavinbunney/kubectl >= 1.14.0`
- CI uses `mock_provider "kubectl"` — no real credentials needed. Test assertions must not rely on resource attribute values (mock returns hash placeholders) — assert on fixed booleans like `server_side_apply == true` instead
- Destroy order: `eso_configuration` is destroyed before `orchestrator_custom_addons` (ESO Helm) in the reverse pipeline

#### `argocd-configuration` repo → `https://github.com/urukube/argocd-configuration`
Terraform module that creates ArgoCD ApplicationSets on the orchestrator cluster. Consumed by `orchestrator-plane-setup/argocd-configuration/` at `ref=v1.3.0`.

- **ApplicationSet `platform-custom-xrds`**: uses the ArgoCD SCM Provider generator (GitHub) to discover all repos in the org with the GitHub topic `platform-custom-xrds`, creates one ArgoCD Application per repo, syncing from `main` branch path `.` into `crossplane-system` namespace
- GitHub token Kubernetes secret is now **synced by ESO** (`eso-configuration` step) — not manually created. Secret name `argocd-github-token` is created by ESO from SM path `platform/github/github-token`
- Application names are prefixed `xrd-{{repository}}` to avoid collisions
- `syncPolicy.automated` with `prune: true` and `selfHeal: true`
- Only provider needed: `gavinbunney/kubectl >= 1.14.0`
- CI uses `mock_provider "kubectl"` — no AWS credentials needed

#### `orchestrator-custom-addons` repo → `https://github.com/urukube/orchestrator-custom-addons`
Terraform module that installs all add-ons onto the orchestrator EKS cluster. Each addon follows a `<addon>.tf` + `<addon>-role.tf` split: the `.tf` file owns Kubernetes resources (namespace, Helm release, CRD manifests); the `-role.tf` file owns the IRSA role and IAM policies.

**File layout:**

| File(s) | What it installs |
|---|---|
| `argocd.tf` / `argocd-role.tf` | ArgoCD Helm release + IRSA role with cross-account `sts:AssumeRole`, EKS describe, and ECR read |
| `argocd-ecr-updater.tf` | CronJob (every 6h) that refreshes the ECR token secret in the `argocd` namespace; IRSA role + RBAC scoped to patch that one secret |
| `argocd-image-updater.tf` | ArgoCD Image Updater Helm release; polls ECR every 1m for new image tags |
| `crossplane.tf` / `crossplane-role.tf` | Crossplane Helm + Provider + ProviderConfig install chain; IRSA role with EC2, EKS, IAM, KMS, S3, cross-account STS |
| `eso.tf` | ESO Helm release + IRSA role with Secrets Manager, SSM Parameter Store, KMS decrypt |
| `ecr.tf` | Cross-account ECR pull role (`{cluster-name}-ecr-pull`) for BU clusters in other accounts to assume |
| `istio.tf` / `istio-gateway.tf` / `istio-ingress-class.tf` / `istio-virtualservices.tf` | Istio base → istiod → ingress gateway (NLB); path-based VirtualServices for ArgoCD, Kiali, Prometheus |
| `kiali.tf` | Kiali server Helm release in `istio-system` |
| `prometheus.tf` | Prometheus community Helm release in `monitoring` |

**Key design decisions:**
- No hub/spoke model — this module is the **orchestrator** only. BU cluster (spoke-side) roles are created on-the-fly when Crossplane provisions each BU cluster, not here.
- `ecr.tf` is gated on `var.enable_ecr` (not `enable_argocd`) — ECR is independent infrastructure
- ESO on the orchestrator pulls platform secrets from Secrets Manager/SSM. ECR pull-secret distribution to BU namespaces is a spoke-cluster concern and lives in the BU cluster module (not yet built)
- Crossplane install has two `time_sleep` fences: 60s after Helm (for core CRDs: `Provider`, `DeploymentRuntimeConfig`) and 120s after the `Provider` resource (for the provider package to download and register the `ProviderConfig` CRD)
- Crossplane IRSA trust uses `StringLike` on `system:serviceaccount:crossplane-system:provider-aws-*` and `upbound-provider-aws-*` to cover both the family provider and sub-providers without enumerating each SA name
- `redis`, `redisSecretInit`, and `dex` pods in ArgoCD have `sidecar.istio.io/inject: "false"` — these components make outbound connections during init; the Envoy sidecar intercepts traffic before it is ready, causing restart loops
- All Kubernetes resources use versioned resource types (`kubernetes_namespace_v1`, `kubernetes_role_v1`, `kubernetes_role_binding_v1`) — unversioned resources are deprecated

**Providers:** `hashicorp/aws >= 6.42.0`, `hashicorp/helm ~> 3.0`, `hashicorp/kubernetes >= 2.35.0`, `gavinbunney/kubectl >= 1.14.0`, `hashicorp/time >= 0.9.0`

**Testing:** `terraform test` uses `mock_provider` for all five providers (`aws`, `helm`, `kubernetes`, `kubectl`, `time`) — no real AWS credentials needed in CI. AWS data sources (`caller_identity`, `region`, `partition`) return mock values via `mock_data` blocks. CI workflow has no `Configure AWS Credentials` step.

**Toggle variables:** `enable_istio`, `enable_argocd`, `enable_prometheus`, `enable_kiali`, `enable_eso`, `enable_ecr`, `enable_crossplane` — all default `false`.

#### `platform-xp-s3` repo → `https://github.com/urukube/platform-xp-s3`
Crossplane XRD package that provides a self-service S3 bucket golden path. Auto-discovered by ArgoCD via the `platform-custom-xrds` GitHub topic and deployed to `crossplane-system`.

- Claim kind: `US3Bucket` — composite kind: `XUS3Bucket` — group: `storage.platform.urukube.io/v1alpha1`
- XRD/Composition name: `xus3buckets.storage.platform.urukube.io`
- Required parameters: `awsAccountId` (12-digit, regex-validated), `region`, `buId` (pattern `^BU[0-9]{3}$`)
- Optional parameters: `versioning` (bool, default `false`), `tags` (map)
- Composed resources: `Bucket`, `BucketServerSideEncryptionConfiguration` (AES256, always on), `BucketVersioning`, `BucketPublicAccessBlock` (all four flags, always blocked)
- Cross-account: Composition dynamically creates one `aws.upbound.io/v1beta1 ProviderConfig` per claim, named after the composite, that chains `orchestrator-eks-crossplane` IRSA → `arn:aws:iam::<awsAccountId>:role/urukube-crossplane-role`
- **IAM permissions for `urukube-crossplane-role`**: must be `s3:*` — the Upbound S3 provider reads many bucket attributes on every reconcile pass (ACL, CORS, website, logging, replication, etc.); a minimal action list will fail with AccessDenied during observe
- **EnvironmentConfig-driven tagging**: Composition selects `org-defaults` (static) + BU-specific EnvironmentConfig (by label `bu-id: <buId>`) and patches merged data as AWS tags (`BuId`, `CostCenter`, `OwnerEmail`, `OrgName`, `ComplianceScope`, `ManagedBy`) onto the bucket

#### `platform-tenant-registry` repo → `https://github.com/urukube/platform-tenant-registry`
Registry of Crossplane `EnvironmentConfig` resources for org-wide and per-BU metadata. Synced to the orchestrator cluster by a dedicated ArgoCD Application (not the SCM generator ApplicationSet — do NOT add `platform-custom-xrds` GitHub topic to this repo).

- `org/environmentconfig.yaml` — `org-defaults` EnvironmentConfig: shared by all BUs (`orgName`, `complianceScope`, `dataClassification`, `regulatoryRegion`, `managedBy`)
- `tenants/<bu-name>/environmentconfig.yaml` — one per BU; holds `buId`, `buName`, `costCenter`, `ownerEmail`, `oncallContact`
- Each BU EnvironmentConfig **must** have label `bu-id: <BU-ID>` (e.g. `bu-id: BU001`) — this is what the Composition's label selector matches against `spec.parameters.buId` from the claim
- Onboarding a new BU = add one file under `tenants/` with the right label; XRDs and Compositions need no changes
- Two-layer merge: Compositions always include `org-defaults` (Reference) + BU config (Selector by `bu-id` label); later entry wins on conflict
- ArgoCD Application defined in `argocd-configuration` module — syncs from `main`, path `.`, destination `crossplane-system`
- Pilot BU: `tenants/ads-platform/` → `buId: BU001`, `buName: ads_platform`

---

## Naming Conventions

All resources follow: `{friendly_name}-{bu_id}-{app_id}-{resource-type}`

Example: `allhub-BU001-APP001-vpc`, `allhub-BU001-APP001-eks-nodes-sg`

- `friendly_name` — human-readable prefix (e.g. `allhub`, `devspoke`)
- `bu_id` — Business Unit identifier
- `app_id` — Application identifier
- `env` — one of `dev`, `staging`, `prod`, `test`

---

## Key Conventions

- All Terraform modules use `pre-commit` with checkov, yamllint, terraform-docs, and standard file checks
- `terraform-docs` auto-generates the `<!-- BEGIN_TF_DOCS -->` section in README.md on every commit
- Semantic versioning is driven by Conventional Commits across all repos
- The org name is **urukube** — never BakeFoundry or OrbitCluster
- Security group rules use the modern separate resources (`aws_vpc_security_group_ingress_rule` / `aws_vpc_security_group_egress_rule`) not inline rules
- AWS provider constraint across all modules: `>= 6.42.0` (minimum driven by `terraform-aws-modules/eks ~> 21.0`). Never add an upper bound — it causes irreconcilable conflicts when modules are composed in a root module
- Use `data.aws_region.current.region` — both `.id` and `.name` are deprecated in AWS provider >= 6.42.0
- GitHub Actions workflow files must quote `"on":` (not bare `on:`) to satisfy yamllint truthy rule
- Release workflow pattern: trigger on `workflow_run` (CI completion) not directly on `push`. The `if` condition gates on `conclusion == 'success'` so a failed CI blocks release. Never use `cycjimmy/semantic-release-action` — always use `urukube/release-workflow/.github/workflows/version.yml@v1`
- Lock file (`.terraform.lock.hcl`) is **committed** in all modules — ensures CI reproducibility and auditability of provider upgrades
- terraform-docs `--output-file` path is relative to the module directory being processed — use `../README.md` when TF files are in a subdirectory and the README is at the repo root
- Calling repos (e.g. `orchestrator`) have no TF files at root — comment out terraform-docs in `.pre-commit-config.yaml` for these repos
- Never include `eks_custom_addons` in destroy workflow component options until that component is actually built (`eks_essential_addons` is now built and included)
- `.releaserc.json` should not exist in any repo — the org release workflow handles semantic-release config. Delete it when found.
- `terraform test` files use `mock_provider` for all providers (including `aws`) so tests run without real credentials. AWS data sources (`aws_caller_identity`, `aws_region`, `aws_partition`) must be covered with `mock_data` blocks. CI workflows for addon/add-on modules do NOT need a `Configure AWS Credentials` step.
- Addon module file convention: `<addon>.tf` owns Kubernetes resources (namespace, Helm release, CRD manifests, `time_sleep`); `<addon>-role.tf` owns the IRSA role and IAM policies. Locals shared between the two files (e.g. namespace name) stay in the `.tf` file; locals used only by IAM (e.g. role name) go in the `-role.tf` file.
- Istio ingress gateway NLB requires two annotations beyond `aws-load-balancer-scheme`: `service.beta.kubernetes.io/aws-load-balancer-manage-backend-security-group-rules: "true"` (lets LBC add node SG inbound rules for NLB traffic — without this health checks fail) and `service.beta.kubernetes.io/aws-load-balancer-attributes: load_balancing.cross_zone.enabled=true` (without this, only the AZ where the Istio pod runs can receive traffic)
- Kiali caches Prometheus "disabled" state if Prometheus is not ready when Kiali starts. Fix: `depends_on = [helm_release.prometheus]` in `kiali.tf`. Live fix: `kubectl rollout restart deployment/kiali -n istio-system`
- Platform services are exposed at `platform.urukube.io` via path-based Istio VirtualServices: `/argocd` (ArgoCD), `/kiali` (Kiali), `/prometheus` (Prometheus). Add `<NLB-IP> platform.urukube.io` to hosts file for local access. NLB DNS: `k8s-istiosys-istioing-d95495ea26-5383e433dca28455.elb.us-east-1.amazonaws.com`
- `secrets.auto.tfvars.json` in a component directory contains only SM paths and descriptions (no secret values) — safe to commit despite `.gitignore` blocking `*.tfvars.json`. Use `git add -f` to force-add it. Secret values are injected at CI runtime via `TF_VAR_*` env vars scoped to the specific workflow step (`env:` on the `uses:` step, not `GITHUB_ENV`), preventing leakage to other Terraform steps.
- AWS-only components (e.g. `orchestrator-secret-management`) do not need `data.tf` or `get_env.sh` — those are only required when a component needs to read remote state from a previous component (e.g. to get cluster endpoint for kubectl/kubernetes providers).
