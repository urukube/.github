# GitOps Promotion Flow

ArgoCD does not "move code." It continuously reconciles each cluster to whatever Git declares. **Promotion is a Git operation** — bump an image tag or Kustomize overlay in the prod path and ArgoCD reflects it.

![GitOps Promotion Flow](assets/gitops-promotion.svg)

---

## Stage Breakdown

| Stage | Tool | What happens |
| ----- | ---- | ------------ |
| **Push → PR → Merge** | GitHub | Developer raises a PR; peer review gates the merge |
| **CI Build** | GitHub Actions | Image built, signed with cosign, SBOM generated, pushed to JFrog Artifactory |
| **Dev Git update** | Kargo / automation | Image tag bumped in the dev Kustomize overlay |
| **Dev reconcile** | ArgoCD (on Orchestrator) | ArgoCD detects the Git change and syncs the dev cluster |
| **Promotion** | Kargo pipeline | Multi-stage pipeline promotes the validated image tag to the prod overlay |
| **Prod Git update** | Kargo / automation | Image tag bumped in the prod Kustomize overlay |
| **Prod reconcile** | ArgoCD + Argo Rollouts | ArgoCD syncs the prod cluster; Argo Rollouts executes canary or blue-green strategy |

---

## Key Principles

- **Git is the source of truth.** No direct kubectl applies to clusters — every change goes through Git.
- **Promotion is a Git operation.** Kargo or a manual overlay bump in the prod path triggers prod deployment. There is no "push to prod" button.
- **Supply-chain security is enforced at CI.** Images unsigned or without a valid SBOM are rejected by admission control at the cluster — they never reach prod.
- **Progressive delivery on prod.** Argo Rollouts manages canary or blue-green rollouts so bad releases are caught before full traffic shift.
