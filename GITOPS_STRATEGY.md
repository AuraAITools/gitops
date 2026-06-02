# GitOps + ArgoCD Strategy

**Date:** 2026-05-30
**Method:** Stochastic-consensus synthesis from 4 independent research agents covering (1) repository structure, (2) environment promotion, (3) secrets management, (4) production ecosystem & roadmap.

---

## 1. Convergent Consensus

These points were reached independently by all four research streams and represent the strongest signal for a future-proof design.

| # | Convergent finding |
|---|---|
| 1 | **Folder-per-environment on a single branch.** Branch-per-environment is universally flagged as an anti-pattern (Codefresh, Akuity, Red Hat, CNCF). |
| 2 | **Config repo is separate from app-source repo.** Manifests-in-app-repo creates infinite sync churn and blast-radius problems. |
| 3 | **Monorepo for the config side**, split only when team/tenant scale forces it — never split per environment. |
| 4 | **Bootstrap = App-of-Apps. Fanout = ApplicationSets.** Layered, not either/or. App-of-Apps is not deprecated. |
| 5 | **Kustomize for first-party overlays; Helm via Kustomize (or multi-source Application) for upstream charts.** Pure Helm-for-promotion is awkward; pure raw YAML doesn't scale. |
| 6 | **Secrets never live as plaintext in Git**, and ideally not as ciphertext either. ArgoCD's official docs explicitly recommend cluster-side patterns (ESO/VSO/Sealed/CSI) over render-time secret plugins. |
| 7 | **Argo Rollouts owns the rollout shape (canary/blue-green); ArgoCD owns reconciliation.** Promotion gating reads Rollout state — do not put rollout logic in CI. |
| 8 | **Multi-cluster = ApplicationSet cluster generator + label selectors today; argocd-agent (pull-from-spoke, mTLS) is the 3.x trajectory.** Design with cluster labels so the topology survives the migration. |
| 9 | **Drift defaults to enable:** `automated: { prune, selfHeal }` + `CreateNamespace=true` + Git webhooks (don't rely on 3-min polling). Add `ignoreDifferences` on `/spec/replicas` wherever HPA is in play. |
| 10 | **The ArgoCD 3.x direction is clear:** OCI-distributed manifests, Source Hydrator (DRY → hydrated split), GitOps Promoter (PR-based env promotion), fine-grained RBAC, agent-based multi-cluster. Architect so these can be adopted incrementally. |

---

## 2. Top 3 Promising Solutions

Three end-to-end architectures, ranked by suitability for the most common scenarios. Each is internally coherent — mixing pieces across solutions is fine but requires deliberate trade-off analysis.

---

### Solution 1 — **Modern Default** (recommended starting point)

**Best for:** Cloud-hosted greenfield, 1–5 clusters, <50 services, small platform team. This is the 90% answer.

#### Repository layout (single config repo)

```
aura-gitops/
├── bootstrap/                     # Root App-of-Apps — only thing pointed at manually
│   └── root-app.yaml
├── components/                    # ArgoCD primitives
│   ├── appprojects/
│   └── applicationsets/
│       ├── core-appset.yaml       # cluster generator over /core
│       ├── apps-appset.yaml       # matrix: git-dir(/apps) × cluster
│       └── preview-appset.yaml    # pull-request generator for PR envs
├── core/                          # platform add-ons (cert-manager, ingress, ESO, Reloader, Kyverno)
│   ├── cert-manager/
│   ├── external-secrets/
│   ├── kyverno/
│   └── argo-rollouts/
├── clusters/                      # cluster registry — Secrets with labels env=, region=
│   ├── dev/
│   ├── staging/
│   └── prod/
└── apps/
    └── <service>/
        ├── base/                  # kustomization.yaml + manifests
        └── overlays/
            ├── dev/
            ├── staging/
            └── prod/              # each overlay holds image-tag.yaml + env patches
```

#### Stack

| Layer | Choice |
|---|---|
| Sync engine | **ArgoCD 3.2+** (annotation-based resource tracking; webhook-driven sync) |
| Fanout | **ApplicationSet** — Matrix(git-directory × cluster), Pull Request generator for previews |
| Templating | **Kustomize** for overlays; multi-source Applications for upstream Helm charts |
| Promotion | **PR-bot via GitHub Actions** — bumps `apps/<svc>/overlays/<env>/image-tag.yaml`. CODEOWNERS gate on `overlays/prod/`. |
| Progressive delivery | **Argo Rollouts** with Prometheus AnalysisTemplates; promotion bot waits on `Rollout` health |
| Secrets | **External Secrets Operator + cloud KMS (AWS Secrets Manager / GCP Secret Manager) + Stakater Reloader** |
| Policy | **Kyverno** (Audit in staging, Enforce in prod via overlays) |
| Notifications | **argocd-notifications** → Slack + PagerDuty on `on-sync-failed`, `on-health-degraded` |
| Observability | Prometheus scraping all four metrics endpoints (`8082/8083/8084/applicationset:8080`) |

#### Why it's future-proof
- Every workload is `apps/<name>/{base,overlays/<env>}` — survives the eventual migration to Source Hydrator (which renders DRY → hydrated paths on the same model).
- Cluster labels (`env=`, `region=`) survive a switch from cluster generator (today) to argocd-agent (tomorrow).
- ESO is vendor-neutral (40+ providers) — swap KMS backend without changing CRDs.

#### Failure modes to expect
- Self-heal vs HPA infinite loop → mitigate with `ignoreDifferences` on `/spec/replicas`.
- Promotion bot starts as 30 lines of YAML and grows unmanageable around ~20 services → planned escape hatch is Solution 2 (Kargo).
- Single hub controller scaling cliff around 50–100 clusters.

---

### Solution 2 — **Enterprise Fleet** (future-proofed for scale)

**Best for:** 50+ clusters, 100+ services, multi-region, regulated industries, multiple platform teams. Architect for this if you expect to be here within 18 months — don't retrofit.

#### Repository layout

Same `apps/` + `overlays/` shape as Solution 1, with two additions:

```
aura-gitops/
├── ...
├── infra/                         # Crossplane Compositions for clusters, VPCs, IAM, DBs
│   ├── compositions/
│   └── claims/
└── hydrated/                      # MACHINE-WRITTEN by Source Hydrator (rendered YAML per env)
    ├── dev/
    ├── staging/
    └── prod/
```

The `apps/` tree is the **DRY** source. ArgoCD's **Source Hydrator** renders into `hydrated/<env>/...` and ArgoCD syncs from there. This gives perfect audit-grade diffs and decouples authoring from delivery.

#### Stack additions vs Solution 1

| Layer | Choice |
|---|---|
| Cluster topology | **argocd-agent** — each spoke runs lite-ArgoCD that dials back to the hub principal over mTLS. No inbound to spokes. |
| Cluster lifecycle | **Crossplane** — clusters and surrounding cloud infra (VPC/IAM/RDS) as CRs in `infra/`. ApplicationSet cluster generator picks up new clusters via the Secrets Crossplane creates. |
| Hydration | **Source Hydrator (ArgoCD 3.1+)** committing rendered YAML to `hydrated/` |
| Promotion | **Kargo** — `Warehouse` subscribes to image registry; `Freight` bundles coordinated multi-service releases; `Stage` defines verification + acceptance; `PromotionTemplate` writes to `hydrated/<env>/`. Replaces the PR bot once promotion topology has conditional rules or multi-service coordination. |
| Manifest distribution | **OCI artifacts** (ECR/GAR/GHCR) once ArgoCD 3.x OCI generator stabilises — Git becomes the authoring layer, OCI the delivery layer. |
| Secrets | ESO with multiple backends; consider **VSO** if Vault-native dynamic creds are required for app workloads. |
| Managed control plane | Honestly evaluate **Akuity / Codefresh / Red Hat OpenShift GitOps** above ~10 clusters. Self-hosting ArgoCD at fleet scale is a real headcount cost. |

#### Why it's future-proof
- argocd-agent is the explicit 3.x trajectory for multi-cluster (GA in OpenShift GitOps 1.19; upstream beta → GA expected 3.4/3.5).
- Source Hydrator + GitOps Promoter is the official replacement for overlay-folder sprawl — building on it now avoids a 2027 refactor.
- Kargo is the spiritual successor to ArgoCD Image Updater; production references include Deutsche Telekom (500+ µservices), Cisco ThousandEyes (2,500+ apps).

#### Risks
- argocd-agent upstream GA timing — design assuming it slips, with ApplicationSet cluster generator as fallback.
- Kargo's Enterprise-only features (Terraform freight) are vendor lock-in; stay on OSS core.
- Source Hydrator output paths changed in 3.3 — version-pin your hydrator config.

---

### Solution 3 — **GitOps Purist / Sovereign** (offline / regulated / no-cloud-KMS)

**Best for:** Airgapped environments, sovereign/regulated workloads (gov, fintech with no-cloud constraints), single-cluster homelabs, scenarios where Git really must be the *only* source of truth.

#### Repository layout

Same `bootstrap/` + `apps/` + `overlays/` shape as Solution 1, with one change: secrets travel **encrypted-in-Git**.

```
aura-gitops/
├── ...
├── apps/<service>/overlays/<env>/
│   ├── kustomization.yaml
│   ├── image-tag.yaml
│   └── secrets.enc.yaml          # SOPS-encrypted, age recipients per cluster
└── .sops.yaml                     # creation rules: which files get encrypted with which key
```

#### Stack

| Layer | Choice |
|---|---|
| Sync engine | **ArgoCD 3.2+** with **sidecar Config Management Plugin** running **KSOPS** (Kustomize) or `helm-secrets` (Helm). NB: legacy `argocd-cm.configManagementPlugins` was removed — sidecar CMP is mandatory. |
| Secrets | **SOPS + age** — age replaces PGP, small modern keys; one age recipient per cluster mounted as Secret into the repo-server sidecar. |
| Templating | **Kustomize** only — minimises external dependencies. |
| Fanout | **ApplicationSet** with List generator (small fixed cluster set) or Git directory generator. |
| Promotion | **PR-based, manual or scripted** — `cp -r overlays/staging overlays/prod` style diffs. No Kargo dependency. |
| Progressive delivery | **Argo Rollouts** (optional — many regulated envs require manual cutovers anyway). |
| Policy | **Kyverno** (YAML-native policies travel well in airgapped Git). |

#### Why this configuration
- **Zero external runtime dependencies.** No KMS API to call, no cloud Secrets Manager, no Vault cluster required. The cluster + Git is the whole system.
- **Disaster-recoverable from cold storage.** Encrypted manifests in Git + age private key in a sealed envelope = full reconstitution. Sealed Secrets cannot match this (loss of controller key = permanent loss of all sealed material).
- **Audit-friendly.** Every secret change is a Git commit; every decryption can be wired through audit logging on the sidecar.

#### Risks
- The repo-server sidecar holds the decryption key → harden that pod (NetworkPolicy, separate node pool, restricted RBAC).
- SOPS maintainer pool is smaller than ESO's — healthy but lower cadence.
- Encrypted-in-Git means forensic permanence: if the cipher is broken in 10 years, today's ciphertext leaks.

---

## 3. Decision Matrix

| If your situation is… | Pick |
|---|---|
| New product, AWS/GCP/Azure, single platform team, <50 services | **Solution 1** |
| Multi-region fleet, 50+ clusters, multi-team org, expecting growth | **Solution 2** |
| Airgapped, sovereign, regulated-no-cloud, or strict Git-only | **Solution 3** |
| Heavy HashiCorp Vault investment | Solution 1 or 2 with VSO swapped in for ESO |
| Already on Flux | Stay on Flux unless you specifically need ArgoCD's UI/ApplicationSet/managed offerings |

---

## 4. Anti-Patterns (Do Not Adopt)

Reached as consensus across all four research streams:

1. **Branch-per-environment** (long-lived `dev`/`staging`/`prod` branches).
2. **Manifests in app source repo.**
3. **ArgoCD Image Updater with `argocd` (in-cluster) write-back** — non-persistent, violates GitOps.
4. **Manual `kubectl apply` or `argocd app set -p image=…`** for promotion.
5. **`argocd-vault-plugin`** — explicitly discouraged by ArgoCD maintainers; plaintext lands in Redis cache.
6. **Sealed Secrets for new multi-cluster builds** — single controller-key fragility, ciphertext is per-cluster, DR is manual.
7. **Helm `--set` from CI** for env-specific values — mutates outside Git, drift is silent.
8. **`targetRevision: <tag>` per-env** to differentiate environments — use one branch, differ via folders.

---

## 5. Roadmap Items to Design Around (next 18 months)

1. **argocd-agent → GA** (likely 3.4/3.5). Pull-from-spoke topology will become default for fleets. Use cluster labels today so generators survive.
2. **Source Hydrator + GitOps Promoter** maturing into the canonical promotion pattern. Adopt the DRY/hydrated mental model now.
3. **Native OCI sources** (3.1 GA, ApplicationSet OCI generator in progress). Plan to push manifest bundles to your container registry alongside images.
4. **Fine-grained RBAC** continues tightening across 3.x minors. Write explicit `update/*`, `delete/*`, `logs/get` policies — never wildcards.
5. **Kargo** continuing to consolidate as the promotion layer. ArgoCD Image Updater is maintenance-mode for new projects.
6. **Helm 4** breaking changes (KubeCon NA 2025 reveal) — Kustomize-centric designs are safer through this transition.
7. **Kyverno** consolidating over Gatekeeper for new platform builds.
8. **Managed ArgoCD** (Akuity, Codefresh) becoming the rational default past ~10 clusters — budget headcount or budget vendor.

---

## 6. Recommendation for `aura/gitops`

Start with **Solution 1** (Modern Default). It is the lowest-risk, highest-leverage starting point for a greenfield repo, and every architectural decision in it composes cleanly into Solution 2 when scale demands. Specifically:

- Initialise the directory tree shown in Solution 1.
- Stand up ArgoCD 3.2+ with App-of-Apps bootstrap.
- Use ApplicationSet Matrix(git-dir × cluster) from day one — do not write per-app `Application` YAML by hand.
- Adopt ESO + Reloader + Kyverno from day one — retrofitting any of them is painful.
- Defer Kargo and Source Hydrator until you feel actual pain from the PR-bot or overlay folders. They are the planned upgrade path, not the starting point.

---

## 7. Key Source References

**Repository structure**
- ArgoCD official best practices: <https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/>
- Red Hat: How to set your GitOps directory structure: <https://developers.redhat.com/articles/2022/09/07/how-set-your-gitops-directory-structure>
- Akuity GitOps best-practices whitepaper: <https://akuity.io/blog/gitops-best-practices-whitepaper>
- CNCF: App-of-Apps in ArgoCD: <https://www.cncf.io/blog/2025/10/07/managing-kubernetes-workloads-using-the-app-of-apps-pattern-in-argocd-2/>

**Promotion**
- Codefresh/Octopus: Stop using branches for GitOps environments: <https://octopus.com/blog/stop-using-branches-deploying-different-gitops-environments>
- Codefresh: 30 ArgoCD anti-patterns: <https://octopus.com/blog/30-argo-cd-antipatterns-for-gitops>
- Kargo patterns: <https://docs.kargo.io/user-guide/patterns>
- Akuity: Kargo — the missing GitOps promotion layer: <https://akuity.io/blog/kargo-gitops-promotion-layer>

**Secrets**
- ArgoCD secret management: <https://argo-cd.readthedocs.io/en/stable/operator-manual/secret-management/>
- External Secrets Operator: <https://external-secrets.io/latest/>
- ESO on CNCF: <https://www.cncf.io/projects/external-secrets/>
- SOPS: <https://github.com/getsops/sops>
- Stakater Reloader: <https://github.com/stakater/Reloader>

**Ecosystem & roadmap**
- argocd-agent: <https://github.com/argoproj-labs/argocd-agent>
- Source Hydrator: <https://argo-cd.readthedocs.io/en/latest/user-guide/source-hydrator/>
- GitOps Promoter: <https://argo-gitops-promoter.readthedocs.io/en/latest/>
- ArgoCD 2.14 → 3.0 upgrade guide: <https://argo-cd.readthedocs.io/en/stable/operator-manual/upgrading/2.14-3.0/>
- ArgoCD native OCI proposal: <https://argo-cd.readthedocs.io/en/stable/proposals/native-oci-support/>
- Crossplane + ArgoCD guide: <https://docs.crossplane.io/latest/guides/crossplane-with-argo-cd/>
