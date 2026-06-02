# gitops

GitOps config repo for AuraAITools. ArgoCD reconciles every cluster from this repo. See [`GITOPS_STRATEGY.md`](./GITOPS_STRATEGY.md) for the architecture rationale.

## Dashboards

| Dashboard   | URL (once routing is up) | Port-forward fallback                                                                | What it shows                                                                  |
| :---------- | :----------------------- | :----------------------------------------------------------------------------------- | :----------------------------------------------------------------------------- |
| **ArgoCD**  | http://argocd.lab.lan    | `kubectl -n argocd port-forward svc/argocd-server 8080:80` → http://localhost:8080   | Applications, sync status, sync history, drift, manual sync.                   |
| **Grafana** | http://grafana.lab.lan ⏳ | `kubectl -n monitoring port-forward svc/kube-prometheus-stack-grafana 3000:80` → http://localhost:3000 | Cluster + node metrics out of the box. Istio's canonical dashboards (Mesh / Service / Workload / Performance / Control Plane / Extension) appear under the **Istio** folder — pulled from `istio/istio@release-1.30` at helm-template time. |
| **Kiali**   | http://kiali.lab.lan ⏳   | `kubectl -n kiali port-forward svc/kiali 20001:20001` → http://localhost:20001       | Service mesh topology, traffic graph, per-service request rate / error rate.   |

⏳ = `VirtualService` not yet committed; see [Backlog](#backlog).

**Initial credentials** (rotate after first login):
- ArgoCD: `admin` / `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d`
- Grafana: `admin` / `admin` (set in `core/monitoring/values.yaml`)
- Kiali: anonymous (homelab only — change `auth.strategy` before any shared env)

## Layout

```
bootstrap/
├── root-app.yaml                    # the only thing applied by hand — App-of-Apps root
└── apps/                            # every other Application lives here
    ├── argocd.yaml                  # ArgoCD self-management        (sync-wave -10)
    ├── istio-base.yaml              # CRDs                          (sync-wave 0)
    ├── istiod.yaml                  # control plane                 (sync-wave 1)
    ├── istio-gateway.yaml           # ingress gateway               (sync-wave 2)
    ├── argocd-routing.yaml          # expose argocd UI              (sync-wave 3)
    ├── kube-prometheus-stack.yaml   # Prom + Grafana + Alertmgr     (sync-wave 4)
    └── kiali.yaml                   # mesh topology dashboard       (sync-wave 5)
core/                                # values / overlays for platform components
├── argocd/
│   ├── values.yaml
│   └── routing/                     # Gateway + VirtualService for argocd.lab.lan
├── istio/{istiod,gateway}-values.yaml
├── monitoring/values.yaml           # kube-prometheus-stack
└── kiali/values.yaml
```

## Pinned versions

| Component               | Version  | Notes                                   |
| :---------------------- | :------- | :-------------------------------------- |
| ArgoCD Helm chart       | `9.5.17` | ships ArgoCD `v3.4.3`                   |
| Istio                   | `1.30.0` | sidecar mode                            |
| kube-prometheus-stack   | `86.1.0` | Prometheus Operator `v0.91.0`           |
| Kiali                   | `2.27.0` | Helm chart `kiali/kiali-server`         |

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

| Wave | App              | Why it has to go in this order                                                                  |
| :--- | :--------------- | :---------------------------------------------------------------------------------------------- |
| `-10`| `argocd`         | Self-management has to take over the in-place Helm release before any other App starts syncing. |
| `0`  | `istio-base`     | Installs the Istio CRDs. Everything below references them — without these, manifests fail.      |
| `1`  | `istiod`         | The control plane; admission webhook for sidecar injection. Gateways can't come up without it.  |
| `2`  | `istio-gateway`  | Gateway pods need sidecars injected by `istiod`, so this can't race ahead.                      |
| `3`  | `argocd-routing` | `Gateway` + `VirtualService` for `argocd.lab.lan` — needs istiod (CRD validation) + gateway up. |
| `4`  | `kube-prometheus-stack` | Prometheus + Grafana + Alertmanager + node-exporter + kube-state-metrics. No hard dep on Istio at sync time, but scrapes envoy sidecars and istiod once they're up. |
| `5`  | `kiali`          | Service mesh dashboard. Reads from the Prometheus that wave 4 just installed. |

Two things to remember:

- Waves order **the sync**, not the runtime. Once everything is Healthy, waves are irrelevant — the controller just watches drift on every resource equally.
- Waves work on a single Application (resources inside one Application sync) **and** at the App-of-Apps level (Application objects themselves sync in wave order). We use both: the four Applications above are wave-ordered, and individual resources inside an Application can carry their own sync-wave annotation if needed.

### Initial admin password lifecycle

The installer auto-creates `Secret/argocd-initial-admin-secret` in the `argocd` namespace with a random password for the built-in `admin` user. It is **not** the authoritative store — the real admin password hash lives in `Secret/argocd-secret` under `admin.password` (bcrypt). On first startup, if `argocd-secret` has no `admin.password` set, `argocd-server` seeds it from `argocd-initial-admin-secret`. That's the only purpose of the initial secret — bootstrapping the seed.

Lifecycle:

1. Read the password: `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d`.
2. Log in to the UI (or `argocd login`).
3. `argocd account update-password` — writes a new bcrypt hash into `argocd-secret`.
4. **Only then** `kubectl -n argocd delete secret argocd-initial-admin-secret`. It's now redundant.

Why not delete it immediately: if you drop it before step 3, and `argocd-server` later restarts, the controller's seeding check finds no admin password to seed from and no plaintext to recover — you lock yourself out. The initial secret is a one-time recovery lifeline; only drop it once you've rotated.

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

# Expect: root, argocd, istio-base, istiod, istio-gateway, argocd-routing,
#         kube-prometheus-stack, kiali
```

Smoke-test self-management: change `core/argocd/values.yaml`, commit, push. The `argocd` Application drifts to OutOfSync then self-heals within ~3 min (or instantly with a Git webhook).

## Access the UI

### Steady-state — via Istio at `http://argocd.lab.lan`

The `argocd-routing` Application installs a `Gateway` (in `istio-ingress` ns, listening on :80) and a `VirtualService` (in `argocd` ns, routing to `argocd-server:80`). One-time DNS:

```bash
# Find the address the gateway is reachable on
kubectl -n istio-ingress get svc istio-ingressgateway

# Then on whichever machine you'll browse from, add to /etc/hosts:
#   <gateway-ip>   argocd.lab.lan
# For k3d on the Mac mini itself: 127.0.0.1   argocd.lab.lan
```

Verify end-to-end:

```bash
curl -v http://argocd.lab.lan/    # expect 200 + ArgoCD HTML
```

If `curl` 503s or "no healthy upstream," the Gateway selector isn't matching the gateway pods — `kubectl -n istio-ingress get pods --show-labels` and confirm `istio=ingressgateway` is among the labels.

### Fallback — port-forward (when routing isn't up yet, e.g. fresh bootstrap)

```bash
kubectl -n argocd port-forward svc/argocd-server 8080:80
# http://localhost:8080
```

### First login

User `admin`; initial password from the auto-generated Secret:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
```

Rotate the admin password (`argocd account update-password`) or wire SSO, **then** delete `argocd-initial-admin-secret`. Deleting it before changing the password locks you out on the next `argocd-server` restart.

## Observability

Three things make up the stack:

| Piece                      | Source                                                                 |
| :------------------------- | :--------------------------------------------------------------------- |
| **Prometheus**             | from `kube-prometheus-stack` — scrapes envoy sidecars (`:15090 /stats/prometheus`) and istiod via `additionalPodMonitors` / `additionalServiceMonitors` in `core/monitoring/values.yaml`. |
| **Grafana**                | also from `kube-prometheus-stack`; sidecar auto-loads any `ConfigMap` labeled `grafana_dashboard=1`. Istio's canonical dashboards are wired in via `grafana.dashboards.istio.*` in `core/monitoring/values.yaml` — the grafana subchart fetches the JSON at helm-template time and generates labeled ConfigMaps automatically, grouped under an **Istio** folder. |
| **Kiali**                  | service mesh topology + dep graph; reads metrics from the same Prometheus. Auth set to `anonymous` for homelab — change before any shared env. |

What's disabled and why (in `core/monitoring/values.yaml`):

- `kubeEtcd`, `kubeProxy`, `kubeControllerManager`, `kubeScheduler` — k3d/k3s replaces or hides these. Leaving them on just produces DOWN targets and noisy alerts.
- `tracing.enabled: false` in Kiali — no Tempo / Jaeger installed yet.

### Access (until per-service routing is added)

```bash
# Grafana
kubectl -n monitoring port-forward svc/kube-prometheus-stack-grafana 3000:80
# http://localhost:3000  — admin / admin (change immediately)

# Kiali
kubectl -n kiali port-forward svc/kiali 20001:20001
# http://localhost:20001
```

Adding `grafana.lab.lan` and `kiali.lab.lan` `VirtualService`s is a follow-up — see Backlog.

## Re-bootstrap (cluster blown away)

The bootstrap is idempotent. Re-run steps 3 → 6 above against a fresh cluster. The `argocd` Application is annotated `sync-wave: -10` so it reconciles before everything else.

## Backlog

- [ ] **Secrets management — SOPS + age.** Per the strategy doc, Sealed Secrets is an anti-pattern for this setup (single controller-key fragility, per-cluster ciphertext, manual DR). The chosen direction is SOPS + age with a KSOPS sidecar in `argocd-repo-server`. Removes the imperative PAT bootstrap and lets app secrets live encrypted in Git.
- [ ] **cert-manager** for TLS at `*.lab.lan`. Options: self-signed CA via the cert-manager bootstrap Issuer (zero external deps, browser warnings) or Let's Encrypt via DNS-01 if the lab domain ever becomes routable. Currently all routing is plain HTTP, which is OK on the LAN only.
- [ ] **Istio routing for Grafana + Kiali** — `grafana.lab.lan` / `kiali.lab.lan` VirtualServices. Will also be the point to refactor `argocd-gateway` → a shared `public-gateway` with `*.lab.lan`.
- [ ] **Tracing** — Tempo (or Jaeger), wired into Kiali's `external_services.tracing`.
- [ ] **Source Hydrator enablement** — once an `apps/<svc>/overlays/<env>/` tree exists. Strategy memo and project memory both have this as day-one for app workloads.
