# gitops

GitOps config repo for AuraAITools. ArgoCD reconciles every cluster from this repo. See [`GITOPS_STRATEGY.md`](./GITOPS_STRATEGY.md) for the architecture rationale.

## Layout

```
bootstrap/
├── root-app.yaml             # the only thing applied by hand — App-of-Apps root
└── apps/                     # every other Application lives here
    ├── argocd.yaml           # ArgoCD self-management
    ├── istio-base.yaml       # CRDs                (sync-wave 0)
    ├── istiod.yaml           # control plane      (sync-wave 1)
    └── istio-gateway.yaml    # ingress gateway    (sync-wave 2)
core/                         # values / overlays for platform components
├── argocd/values.yaml
└── istio/{istiod,gateway}-values.yaml
```

## Pinned versions

| Component         | Version  | Notes                  |
| :---------------- | :------- | :--------------------- |
| ArgoCD Helm chart | `9.5.17` | ships ArgoCD `v3.4.3`  |
| Istio             | `1.30.0` | sidecar mode           |

Bump deliberately, never track `latest`.

## Concepts

### Sync waves

ArgoCD reconciles in **waves**, not all-at-once. The annotation `argocd.argoproj.io/sync-wave: "<int>"` assigns a resource (or, in an App-of-Apps, a child Application) to a wave. ArgoCD then:

1. Sorts everything in the sync by wave number, ascending (negatives first).
2. Applies every resource in the lowest wave in parallel.
3. **Waits for that wave to report `Healthy`** before starting the next one.
4. Default wave is `0` if the annotation is missing.

This is how you express "X must exist and be ready before Y is created." It's the GitOps replacement for `helm install --wait && kubectl wait && helm install` chains.

Waves in this repo:

| Wave | App             | Why it has to go in this order                                                                  |
| :--- | :-------------- | :---------------------------------------------------------------------------------------------- |
| `-10`| `argocd`        | Self-management has to take over the in-place Helm release before any other App starts syncing. |
| `0`  | `istio-base`    | Installs the Istio CRDs. Everything below references them — without these, manifests fail.      |
| `1`  | `istiod`        | The control plane; admission webhook for sidecar injection. Gateways can't come up without it.  |
| `2`  | `istio-gateway` | Gateway pods need sidecars injected by `istiod`, so this can't race ahead.                      |

Two things to remember:

- Waves order **the sync**, not the runtime. Once everything is Healthy, waves are irrelevant — the controller just watches drift on every resource equally.
- Waves work on a single Application (resources inside one Application sync) **and** at the App-of-Apps level (Application objects themselves sync in wave order). We use both: the four Applications above are wave-ordered, and individual resources inside an Application can carry their own sync-wave annotation if needed.

## One-time bootstrap

Run on the cluster's `kubectl` context. The Helm install is throwaway — once the `argocd` Application reconciles, it adopts the release in-place and Git becomes the source of truth.

```bash
# 1. Clone the repo onto the host running kubectl
git clone https://github.com/AuraAITools/gitops.git ~/gitops
cd ~/gitops

# 2. Add the argo helm repo
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# 3. Create the namespace
kubectl create namespace argocd

# 4. Install ArgoCD (single line — zsh line-continuations need a trailing-whitespace-free `\`)
helm install argocd argo/argo-cd --namespace argocd --version 9.5.17 --values ~/gitops/core/argocd/values.yaml --wait

# 5. Register the private-repo credential (skip if the repo is public).
#    Chicken-and-egg: ArgoCD can't fetch the repo until this Secret exists,
#    so it has to be created imperatively here. Move it under SOPS later.
read -s -p "PAT: " PAT && echo
kubectl -n argocd create secret generic aura-gitops-repo \
  --from-literal=type=git \
  --from-literal=url=https://github.com/AuraAITools/gitops.git \
  --from-literal=username=<gh-username> \
  --from-literal=password="$PAT"
unset PAT
kubectl -n argocd label secret aura-gitops-repo argocd.argoproj.io/secret-type=repository

# 6. Hand over to GitOps — root-app creates every other Application
kubectl apply -f bootstrap/root-app.yaml
```

**Do not** `helm uninstall` after Step 6 — that would delete the live ArgoCD. The `argocd` Application adopts the existing release because the release name (`argocd`) and namespace match.

### Creating the PAT (Step 5)

Fine-grained PAT at <https://github.com/settings/personal-access-tokens/new>:
- **Resource owner:** `AuraAITools`
- **Repository access:** Only select repositories → `AuraAITools/gitops`
- **Permissions → Contents:** `Read-only` (bump to read-write later when Source Hydrator lands)
- **Expiration:** ≤90 days

The label `argocd.argoproj.io/secret-type=repository` is what makes ArgoCD pick the Secret up — without it the Secret is invisible. ArgoCD matches it to any Application whose `repoURL` equals the Secret's `url` field.

### If you applied `root-app` before the credential existed

The Application gets stuck on `Unknown / Healthy` waiting on Git auth. After creating the Secret, force an immediate reconcile instead of waiting on the 3-min poll:

```bash
kubectl -n argocd annotate application root \
  argocd.argoproj.io/refresh=hard --overwrite
```

## Verify

```bash
# All Applications should be Synced + Healthy within a few minutes
kubectl -n argocd get applications

# Expect: root, argocd, istio-base, istiod, istio-gateway
```

Smoke-test self-management: change `core/argocd/values.yaml`, commit, push. The `argocd` Application drifts to OutOfSync then self-heals within ~3 min (or instantly with a Git webhook).

## Access the UI

Until Istio routing is wired up (`argocd.lab.lan` via a `Gateway` + `VirtualService`), use port-forward:

```bash
kubectl -n argocd port-forward svc/argocd-server 8080:80
# http://localhost:8080  — user: admin
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
```

Rotate the initial admin password after first login.

## Re-bootstrap (cluster blown away)

The bootstrap is idempotent. Re-run steps 3 → 6 above against a fresh cluster. The `argocd` Application is annotated `sync-wave: -10` so it reconciles before everything else.
