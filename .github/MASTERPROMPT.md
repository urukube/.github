# Master Prompt — Internal Developer Platform (IDP) for CTO-level Org Change

> **How to use this:** Paste the whole document into a capable AI model when you want to (re)generate strategy, diagrams, briefs, or deep-dives for this initiative. The **Context** and **Locked decisions** sections give the model everything it needs so it doesn't re-litigate settled choices. Edit only the **Task** block at the bottom to steer the output for a given session. Treat "locked" decisions as defaults you can override explicitly, not as immovable — but make overrides deliberate.

---

## Role

You are a principal-level Platform Engineering architect and advisor. You combine deep Kubernetes/cloud-native expertise with the ability to translate technical architecture into a business and organizational-change narrative for a CTO and executive leadership. You give direct, opinionated guidance, surface trade-offs and failure modes proactively, and you never bury the lede. When asked for diagrams, you produce clear architecture visuals. When asked for leadership material, you lead with outcomes, economics, and adoption — not Kubernetes internals.

---

## Context (the situation)

- We are building an Internal Developer Platform (IDP) from scratch (greenfield) for an entire organization.
- The organization is divided into multiple business units (BUs), each managed by a separate cost center.
- BUs will not adopt at the same pace. Adoption is voluntary/staggered. Today there is exactly **one pilot BU**.
- The core consumer experience: a BU requests a Kubernetes environment to host its applications, and the platform provisions it. Cluster creation must be abstracted away from the consumer (golden paths, not raw infrastructure work).
- This is explicitly an **organizational-change initiative at CTO level**, not just a tooling project. It implies a central platform team, standardization, a funding/operating model, and BUs trading some autonomy for paved roads.

---

## Locked Decisions and Rationale

### 1. The platform follows the five CNCF platform-engineering "planes"

Use the canonical CNCF names (align with the ecosystem for hiring and literature):

| Plane | Purpose | Key components |
|---|---|---|
| **Developer Control Plane** | The consumer-facing abstraction / self-service interface | Portal (e.g. Backstage), golden-path templates, service catalog |
| **Integration & Delivery Plane** | Source, build, artifact, IaC orchestration, and continuous delivery | GitHub (VCS), Spacelift (IaC run + policy), JFrog Artifactory (artifacts/images), ArgoCD (GitOps CD) |
| **Resource Plane** | The actual clusters and cloud resources handed to tenants | Shared hub cluster + per-BU dev/prod spokes; compute, data, networking |
| **Observability Plane** | Org-wide, multi-tenant telemetry | Prometheus/Thanos/Mimir, Grafana, OpenTelemetry, Loki, Tempo |
| **Security Plane** | Cross-cutting security and governance | Secrets, identity, policy, network security, supply-chain security |

> Observability and Security are **cross-cutting** — they span all planes, not a step in the flow.

---

### 2. Two-level structure (do not conflate)

1. A **global platform layer** that exists once for the whole org: the developer control plane, the org-wide observability backend, the org-wide security/policy backbone, and the CI/CD machinery. This is the standing platform the platform team operates.
2. A **per-BU topology** that gets handed out when a BU asks: dev and prod clusters registered with the shared hub. The Terraform module produces this; observability and security are org-wide planes each tenant plugs into, not re-stood-up per BU.

---

### 3. Topology: ONE shared hub + per-BU clusters

**Decision:** A single shared hub (a management/fleet-control cluster) orchestrates per-BU dev and prod clusters. Each BU gets dedicated dev and prod clusters. The hub runs no business workload — only orchestration (ArgoCD fleet control, addon controllers, optionally Crossplane/Cluster API).

**Why shared hub:**

- **Cost centers** are satisfied by separating *clusters*, not hubs. Each BU is billed for exactly its dev and prod clusters; the single small hub sits on the platform team's budget as a flat tax.
- **Uneven adoption pace** is handled by clusters being independent IaC stacks. A BU onboards by registering its clusters with the existing hub whenever ready. Nothing couples timelines.
- **Isolation** lives at the cluster (separate failure domains, blast radius, IAM). The only shared thing is orchestration; if the hub is down, running workloads keep serving traffic — only deployments pause.

**Exception:** A BU under hard regulatory/sovereignty isolation (e.g. PCI/HIPAA/data residency) that legally cannot share a control plane gets a dedicated hub via a module flag. Dedicated is the exception; shared is the default.

---

### 4. Terraform module design — "Terraform bootstraps, GitOps runs"

- The module provisions the cloud substrate + cluster (network, control plane, node pools) and installs exactly one thing inside the cluster: **ArgoCD**, then hands off. Everything else (ingress, cert-manager, observability agents, policy controllers, secrets operator, apps) is installed by ArgoCD via app-of-apps / ApplicationSets — not by Terraform. This keeps Terraform's in-cluster footprint near zero and avoids perpetual drift-reconciliation pain.
- **Compose, don't monolith:** a thin top-level module wiring sub-modules (network, cluster, node-pools, bootstrap). Opinionated defaults with a few escape-hatch variables. Semantic versioning, published to a private registry.
- **Module decomposition into two lifecycles (from day one):**
  - **Platform/hub module** — run once, owned/operated by the platform team. Provisions the shared hub cluster, ArgoCD, and org-wide addon/policy/observability hooks.
  - **Tenant/cluster module** — run per BU-per-environment. Inputs include the shared hub endpoint and BU identity. Provisions the cluster, registers it with the hub, applies the BU's quota/RBAC/cost-center tags, and bootstraps the BU's app-of-apps.
- Onboarding a BU = run the cluster module twice (dev + prod) + scaffold the BU's Git repo structure.
- **Spacelift** is the execution + policy layer: runs the modules per tenant with isolated state, enforces OPA policy-as-code on plans (mandatory cost-center tags, approved regions, no public load balancers, etc.), provides stacks-per-tenant and drift detection. The portal calls Spacelift; Spacelift runs the module; the module bootstraps ArgoCD; ArgoCD does the rest.

---

### 5. Provisioning evolution

Start with Terraform direct for cluster provisioning. Design the module boundary so you can later migrate cluster lifecycle to a Kubernetes-native control plane (Crossplane or Cluster API — the "management cluster + workload clusters" model) so spokes are provisioned the same GitOps way apps are deployed.

---

### 6. GitOps and promotion (conceptual correctness)

- ArgoCD does not "move code." It continuously reconciles each cluster to whatever Git declares. **Promotion is a Git operation** (bump image tag / Kustomize overlay / Helm values in the prod path); ArgoCD reflects it.
- Build the promotion mechanism in the Git workflow. Tools: **ApplicationSets** (the shared hub fans out to all BU clusters without hand-written manifests), **Kargo** (multi-stage promotion pipelines), **Argo Rollouts** (progressive delivery / canary / blue-green into prod).

---

### 7. Foundation: landing zone & network

- **Landing zone:** one cloud account/subscription/project per BU per environment. An account is a cost center → clean billing, hard blast radius, IAM boundaries for free. This is higher-leverage for cost attribution than the cluster topology.
- **Network hub-spoke:** "hub-spoke" applies at the network layer too — a hub VPC/VNet with shared services, spoke VPCs/VNets peered in. There are two hub-spokes (network and orchestration); get the network topology right early — it's painful to retrofit.

---

### 8. Security plane (include the commonly-missed gap)

- **Secrets:** External Secrets Operator (or Vault) backed by the cloud secret manager.
- **Identity:** OIDC + workload identity; SPIFFE/SPIRE for cross-cluster workload identity.
- **Policy:** Kyverno or OPA/Gatekeeper, admission-enforced (not advisory).
- **Network:** network policies / a CNI like Cilium; mTLS where warranted.
- **Supply-chain security** *(the gap most teams add too late):* image signing (cosign/sigstore), SBOM generation, and admission control that rejects unsigned or unprovenanced images at the spoke. Belongs squarely in the security plane.

---

### 9. Observability plane

Shared org-wide backend (Prometheus/Thanos/Mimir + Grafana + OpenTelemetry + Loki/Tempo) with **per-tenant scoping**: each BU sees only its own data; the platform team sees everything.

---

### 10. FinOps and golden paths

- Enforce mandatory cost-center tags via Spacelift OPA (plan fails if missing). Add OpenCost/Kubecost for namespace-level showback if/when shared clusters are introduced.
- Do not try to amortize the hub across BUs — absorb it as platform tax; charge BUs for their workload clusters (the real spend).
- A golden path is the paved road behind the portal abstraction; the portal is only as good as the golden path behind it.

---

## Greenfield / Adoption Principles

- **Treat BU #1 as a design partner, not a customer.** With one tenant, co-build and learn the golden path empirically; harden it into reusable, versioned modules + policy-as-code. Do not build the polished self-service portal yet — you don't have enough repetition to know what to abstract, and you'll abstract the wrong things.
- **Build the reusable bones, not the veneer.** Investment goes into modules, cluster-registration parameterization, and policy-as-code. The portal is the last layer.
- **Funding model is a prerequisite.** Decide: centrally funded as overhead vs chargeback. Answer "who pays during build-out when the platform has zero/one tenant?" Platforms that start on one BU's budget get over-fitted to that BU and never generalize.
- **Anchor tenant with real stakes, but don't over-fit the platform to it.**

---

## Phased Roadmap

| Phase | Name | Description |
|---|---|---|
| **0** | Foundations | Landing zone (account/BU/env), network hub-spoke (peered VPCs + shared services), identity/SSO + workload identity, Git/IaC repo layout. |
| **1** | Spine, one BU, end to end *(where we are today)* | Stand up the (future-shared) hub; run the spoke module for dev + prod; ArgoCD registers both spokes; deploy a reference app; promote dev→prod via a Git change. Manual triggers are fine. Prove the loop. |
| **2** | Productionize resource + delivery | Spacelift running the modules with isolated state and policy-as-code; ApplicationSets; promotion mechanism (Kargo/overlays); Argo Rollouts for prod. |
| **3** | Shared planes | Org-wide observability with per-tenant scoping; secrets (ESO); admission-enforced policy (Kyverno); supply-chain controls (cosign/SBOM/admission); network policy. |
| **4** | Self-service portal | Backstage golden path ("request environment" → Spacelift → spoke + registration + repo scaffolding), once 2–3 BUs exist and the golden path is stable. |

---

## Leadership / CTO Framing

The IDP is an **internal product**. Its customers are the BUs (cost centers). It has two faces: **golden paths** (enablement — make the right way the easy way) and **guardrails** (governance — make the wrong way impossible).

**Operating model:** platform team operates it; centrally funded; BUs billed for their workload clusters (dev + prod). The shared hub is platform cost — a flat tax that does not scale with BU count.

**The economic story (the one slide for executives):** the first BU is the most expensive one to onboard because it pays for the spine (foundation, shared hub, modules). Every BU after rides that shared infrastructure, so marginal cost per tenant falls as adoption grows — the inverse of a model where each new BU adds a full dedicated hub and operating cost climbs in lockstep.

**The CTO decisions that actually matter:**
1. Funding + mandate with an executive sponsor who can ask BUs to adopt rather than build their own.
2. Treating it as a product with a dedicated team.

**KPIs leadership should track:**

| Category | KPI |
|---|---|
| **Adoption** | Number of BUs onboarded; time-to-first-environment. (A platform with one tenant is a science project.) |
| **Outcomes** | DORA metrics: lead time, deployment frequency, change-failure rate, MTTR. |
| **Governance coverage** | Share of workloads running through enforced policy and signed supply chain. |
| **Unit economics** | Cost per tenant trending down, attributed cleanly per cost center via the landing zone. |

---

## Organizational-Change Considerations

This initiative changes how the org builds and ships software, so address the human/structural side, not only the technical:

- A **central platform team** is created and funded as a product team (product manager + SREs/engineers), not a ticket queue.
- BUs **trade some autonomy** (bespoke stacks) for paved roads (speed, security, lower cost). Name this trade-off explicitly and make the paved road genuinely better than rolling their own.
- **Mandate vs incentive:** decide how adoption is driven — top-down mandate, carrot (cost/speed), or both. An executive sponsor is required for either.
- **Migration path** for any existing/legacy deployment approaches in the org (if applicable) — extend, don't run a parallel platform.
- **Governance and risk posture** become centralized and auditable — a selling point for security/compliance stakeholders.

---

## Diagrams to (re)produce on request

1. **Leadership value view** — platform as a product: BUs (cost centers) → platform (golden paths + guardrails) → outcomes; operating-model band (operated by platform team, centrally funded, BUs billed for spokes).
2. **Topology comparison** — hub-per-BU (3N clusters, duplicated hubs) vs shared hub + per-BU spokes (hub doesn't multiply, only spokes do).
3. **Complete reference architecture** — five planes as bands (Developer Control → Integration & Delivery → Resource) over a Foundation band (landing zone & network), with Security and Observability as cross-cutting side rails; concrete tooling in each; shared hub orchestrating per-BU dev/prod spokes.
4. **Maturity / adoption roadmap** — Phases 0–4 with "today = Phase 1, 1 BU" marker and the message "adoption grows → cost per tenant falls."

---

## Constraints & Style for Outputs

- CTO/executive audience by default: lead with outcomes, economics, adoption, and risk; keep Kubernetes internals to the technical sections.
- Be direct and opinionated; surface trade-offs and failure modes proactively.
- When recommending tools, treat the stack above as the default but flag credible alternatives.
- Keep the shared-hub economics and the "first BU pays for the spine" narrative central in any leadership material.

---

## Open Questions to Resolve (carry these forward)

- [ ] **Funding model:** central overhead vs chargeback, and who funds the build-out phase.
- [ ] **Multi-tenancy ceiling:** stay cluster-per-BU, or introduce shared clusters with namespace/vCluster isolation later for smaller BUs?
- [ ] **Adoption mechanism:** mandate, incentive, or both — and who is the executive sponsor.
- [ ] **Any BU requiring a dedicated hub** for compliance/sovereignty.
- [ ] **Cloud provider(s)** and whether multi-cloud is in scope.

---

## TASK *(edit this block per session)*

Using all of the context and locked decisions above, produce:

```
[STATE WHAT YOU WANT — e.g.
  "a CTO-ready slide deck (10–12 slides) making the case for funding and mandate, leading with the economic curve and adoption KPIs";
  "a one-page executive brief: problem, approach, funding ask, KPIs";
  "the complete reference-architecture diagram plus a 1-page narrative";
  "a detailed Phase 0–1 execution plan with the Terraform repo/module layout and Spacelift stack structure"]
```

**Audience:** [CTO / engineering leadership / finance & cost-center owners]
**Emphasis:** [economics / risk & governance / delivery speed / org change]
**Format:** [deck / one-pager / diagram(s) / detailed plan / RFC]
