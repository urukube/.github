# Secret Management

![ESO Secret Provisioning and Sync Flow](assets/secret-management-flow.svg)

Platform secrets follow a two-phase lifecycle: **Terraform provisions** them into AWS Secrets Manager once at cluster setup, and **ESO continuously syncs** them into Kubernetes secrets on a one-hour refresh cycle.

---

## Phase 1 — Provisioning (Terraform, runs once)

The `orchestrator-secret-management` stage (stage ① in `orchestrator-plane-setup`) creates the secrets in AWS Secrets Manager. Secret **paths and descriptions** are committed in `secrets.auto.tfvars.json` (safe to commit — no values). Secret **values** are injected at CI runtime from GitHub org secrets as `TF_VAR_secret_values`, scoped to that workflow step only so they never leak to other Terraform stages.

| Logical key | SM path | Value source | Purpose |
|---|---|---|---|
| `argocd_admin` | `platform/argocd/admin-password` | `ARGO_ADMIN_SECRET` org secret | ArgoCD admin login — stored as a **bcrypt hash** (not plaintext) |
| `github_token` | `platform/github/github-token` | `PLATFORM_GITHUB_PAT` org secret | GitHub PAT for ArgoCD SCM provider — needs `repo` (read) + `read:org` scopes |

**Key Terraform behaviour:** `ignore_changes = [secret_string]` is set on every `aws_secretsmanager_secret_version`. This means Terraform creates the initial secret version but never overwrites the value on subsequent runs. Secret rotation is done outside Terraform.

The module is called at `ref=v1.2.1` from `orchestrator-plane-setup/orchestrator-secret-management/`.

---

## Phase 2 — Sync (ESO, ongoing every 1 hour)

Once the orchestrator cluster is running and the ESO Helm chart has been installed (stage ④), the `eso-configuration` stage (stage ⑤) creates the ESO resources that wire Secrets Manager to Kubernetes. ESO then syncs secret values automatically on a 1-hour refresh cycle.

### IRSA Authentication Chain

ESO does not use static AWS credentials. Instead it uses IRSA (IAM Roles for Service Accounts):

1. The ESO pod runs under the `eso-service-account` service account in the `external-secrets` namespace
2. The EKS OIDC provider issues a signed JWT for that service account
3. ESO calls `STS:AssumeRoleWithWebIdentity` with the JWT to obtain temporary AWS credentials
4. STS validates the JWT against the cluster's OIDC provider and returns credentials scoped to the IAM role

The IAM role (`{cluster-name}-eso`) is created by `orchestrator-custom-addons/eso-role.tf` with the following permissions:

| Permission set | Actions |
|---|---|
| Secrets Manager | `GetSecretValue` · `DescribeSecret` · `ListSecrets` |
| SSM Parameter Store | `GetParameter` · `GetParameters` · `GetParametersByPath` · `DescribeParameters` |
| KMS | `Decrypt` · `DescribeKey` |

### ClusterSecretStore

`orchestrator-eso-config` creates a `ClusterSecretStore` named `aws-secrets-manager` — a cluster-wide ESO resource that holds the AWS connection config (region, auth method). All `ExternalSecret` resources reference this store by name rather than embedding AWS config themselves.

The `ClusterSecretStore` uses `external-secrets.io/v1` API (ESO >= 0.10 promoted from `v1beta1`).

### ExternalSecrets → Kubernetes Secrets

Two `ExternalSecret` resources are created by `orchestrator-eso-config`, both with `refreshInterval: 1h`:

| ExternalSecret | Namespace | SM path pulled | Target Kubernetes secret | Key | creationPolicy |
|---|---|---|---|---|---|
| `argocd-github-token` | `argocd` | `platform/github/github-token` | `argocd-github-token` | `token` | `Owner` — ESO creates and owns this secret |
| `argocd-admin-password` | `argocd` | `platform/argocd/admin-password` | `argocd-secret` | `admin.password` | `Merge` — ESO patches this key into the existing secret ArgoCD creates |

**`creationPolicy: Owner`** — ESO is the sole owner of `argocd-github-token`. If the ExternalSecret is deleted, ESO garbage-collects the Kubernetes secret too.

**`creationPolicy: Merge`** — ArgoCD creates `argocd-secret` itself during startup. ESO patches only the `admin.password` key into it, leaving ArgoCD's other keys (session signing key, etc.) untouched.

---

## Secret Consumers

| Kubernetes secret | Consumer | How it is used |
|---|---|---|
| `argocd-github-token` (key: `token`) | ArgoCD ApplicationSet SCM Provider | Authenticates to the GitHub API to discover repos tagged `platform-custom-xrds` |
| `argocd-secret` (key: `admin.password`) | ArgoCD server | Admin login — ArgoCD expects this as a bcrypt hash, not plaintext |

---

## Pipeline Ordering Constraint

Stage ⑤ (`eso-configuration`) must run **after** stage ④ (`orchestrator-custom-addons`) which installs the ESO Helm chart. The kubectl provider caches Kubernetes API discovery at plan time — if the `ClusterSecretStore` and `ExternalSecret` CRDs are not yet registered, Terraform cannot plan those resources. The pipeline enforces this ordering automatically.

---

## Adding a New Platform Secret

1. Add an entry to `orchestrator-plane-setup/orchestrator-secret-management/secrets.auto.tfvars.json` with the SM path and description (force-add with `git add -f` since `*.tfvars.json` is in `.gitignore`)
2. Add the secret value to the appropriate GitHub org secret and wire it into the `TF_VAR_secret_values` step `env:` in `orchestrator-plane-setup/.github/workflows/main.yml` — scope it to the `orchestrator-secret-management` step only, not to `GITHUB_ENV`
3. Run the `orchestrator-secret-management` CI stage to provision the SM secret
4. Add an entry to `var.external_secrets` in `orchestrator-plane-setup/eso-configuration/` pointing at the new SM path and the target Kubernetes secret
5. Release `orchestrator-eso-config` — ESO will begin syncing the new secret within one refresh cycle
