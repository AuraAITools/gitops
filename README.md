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

## Application endpoints

External URLs follow the homelab pattern `<svc>[-<env>].lab.lan`. All public-facing services attach to a single shared `*.lab.lan` Istio `Gateway` (`istio-ingress/public-gateway`, installed by the `routing` Application at sync-wave 3); each app's own `VirtualService` adds its concrete host.

Hosts shown with ⏳ are waiting on either the per-service `VirtualService` to land in this repo **or** the hostname being added to your DNS (see [DNS setup](#dns-setup) below). Hosts shown with ❌ are for services not yet deployed.

### Per-environment hosts

| Service              | Dev                              | Staging                              | Prod                          | Status |
| :------------------- | :------------------------------- | :----------------------------------- | :---------------------------- | :----- |
| **BFF** (aura-report-website) | `app-dev.lab.lan` ❌      | `app-staging.lab.lan` ❌             | `app.lab.lan` ❌              | not deployed |
| **report-ms** (Java backend)  | `api-dev.lab.lan` ✅      | `api-staging.lab.lan` ❌             | `api.lab.lan` ✅              | dev + prod deployed (needs `ecr-pull` + `report-ms-secrets` imperative bootstrap, see below); staging not deployed |
| **Keycloak** (public OIDC)    | `accounts-dev.lab.lan` ✅ | `accounts-staging.lab.lan` ❌        | `accounts.lab.lan` ✅         | dev + prod deployed (VirtualService wired); staging not deployed |
| **Keycloak** (admin console)  | `admin-accounts-dev.lab.lan` ✅ | `admin-accounts-staging.lab.lan` ❌ | `admin-accounts.lab.lan` ✅   | dev + prod deployed (VirtualService wired); staging not deployed |
| **SpiceDB** (gRPC, internal)  | n/a                              | n/a                                  | n/a                           | not deployed |

### DNS setup

The shared Gateway listens on `*.lab.lan`, but `/etc/hosts` doesn't support wildcards — each hostname has to resolve to the gateway's external IP individually. Two options:

```bash
# Find the gateway's external IP (with k3d on the Mac mini, this is usually 127.0.0.1)
kubectl -n istio-ingress get svc istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}'; echo
```

**Option A — `/etc/hosts` (simplest, one line per hostname):**

```
127.0.0.1   argocd.lab.lan
127.0.0.1   accounts-dev.lab.lan
127.0.0.1   admin-accounts-dev.lab.lan
127.0.0.1   accounts.lab.lan
127.0.0.1   admin-accounts.lab.lan
```

**Option B — `dnsmasq` (true wildcard, one line covers all current + future hosts):**

```
# /opt/homebrew/etc/dnsmasq.conf  (on the Mac mini)
address=/lab.lan/127.0.0.1
```

Then `sudo brew services restart dnsmasq` and point your laptop's resolver at the Mac mini's IP for the `lab.lan` zone. Option B is what to land before adding many more `*.lab.lan` services.

### In-cluster DNS (always works once the Service exists)

App-to-app calls inside the mesh use cluster DNS — short form within the same namespace, FQDN across.

| Service              | In-cluster URL (same-ns short form)               | Cross-ns FQDN                                     |
| :------------------- | :------------------------------------------------ | :------------------------------------------------ |
| BFF                  | `http://aura-report-website:3000`                 | `http://aura-report-website.apps-<env>.svc:3000`  |
| report-ms (HTTP/GraphQL) | `http://report-ms:8080`                       | `http://report-ms.apps-<env>.svc:8080`            |
| Keycloak             | `http://keycloak:8080`                            | `http://keycloak.apps-<env>.svc:8080`             |
| SpiceDB (gRPC)       | `spicedb:50051`                                   | `spicedb.apps-<env>.svc:50051`                    |
| postgres-aura (rw)   | `postgres-aura-rw:5432`                           | `postgres-aura-rw.apps-<env>.svc:5432`            |
| postgres-keycloak (rw) | `postgres-keycloak-rw:5432`                     | `postgres-keycloak-rw.apps-<env>.svc:5432`        |
| postgres-spicedb (rw) | `postgres-spicedb-rw:5432`                       | `postgres-spicedb-rw.apps-<env>.svc:5432`         |

### Port-forward (poking from your laptop until routing is up)

```bash
# Keycloak — dev (substitute apps-prod for prod)
kubectl -n apps-dev port-forward svc/keycloak 8081:8080
# http://localhost:8081  — admin / admin (rotate immediately)

# report-ms — once deployed
kubectl -n apps-dev port-forward svc/report-ms 8082:8080
# GraphQL at http://localhost:8082/graphql, actuator at /actuator/health

# BFF — once deployed
kubectl -n apps-dev port-forward svc/aura-report-website 3000:3000
# http://localhost:3000

# Postgres (psql via the cnpg plugin or raw)
kubectl -n apps-dev port-forward svc/postgres-aura-rw 5432:5432
# psql "postgresql://aura:$(kubectl -n apps-dev get secret postgres-aura-app -o jsonpath='{.data.password}' | base64 -d)@localhost:5432/aura"
```

### Initial credentials

| Service                     | User    | Password source                                                                                                       |
| :-------------------------- | :------ | :-------------------------------------------------------------------------------------------------------------------- |
| Keycloak admin console (all envs) | `admin` | base-encoded `admin` in `apps/keycloak/base/admin-secret.yaml` — **rotate via the Keycloak admin console after first login** |
| Postgres app users (all DBs / envs) | `<dbname>` (e.g. `aura`, `keycloak`, `spicedb`) | CNPG-generated `postgres-<db>-app` Secret in `apps-<env>`. Key: `password`. |

## Layout

```
bootstrap/
├── root-app.yaml                    # the only thing applied by hand — App-of-Apps root
└── apps/                            # every other Application lives here
    ├── argocd.yaml                  # ArgoCD self-management        (sync-wave -10)
    ├── istio-base.yaml              # CRDs                          (sync-wave 0)
    ├── istiod.yaml                  # control plane                 (sync-wave 1)
    ├── istio-gateway.yaml           # ingress gateway               (sync-wave 2)
    ├── routing.yaml                 # shared *.lab.lan Gateway      (sync-wave 3)
    ├── argocd-routing.yaml          # argocd.lab.lan VirtualService (sync-wave 3)
    ├── kube-prometheus-stack.yaml   # Prom + Grafana + Alertmgr     (sync-wave 4)
    ├── kiali.yaml                   # mesh topology dashboard       (sync-wave 5)
    ├── apps-project.yaml            # AppProject + apps-{dev,staging,prod} ns (wave 6)
    ├── cloudnative-pg.yaml          # CNPG operator                 (sync-wave 6)
    ├── postgres-<db>-<env>.yaml     # 3 databases × 3 envs = 9 Apps (sync-wave 7)
    │                                # db ∈ {aura, keycloak, spicedb}
    │                                # env ∈ {dev, staging, prod}
    ├── keycloak-{dev,prod}.yaml     # IdP                           (sync-wave 8)
    └── report-ms-{dev,prod}.yaml    # Java backend                  (sync-wave 9)
core/                                # values / overlays for platform components
├── argocd/
│   ├── values.yaml
│   └── routing/                     # VirtualService → argocd.lab.lan (attaches to public-gateway)
├── routing/
│   └── public-gateway.yaml          # shared Gateway in istio-ingress; hosts: ["*.lab.lan"]
├── istio/{istiod,gateway}-values.yaml
├── monitoring/values.yaml           # kube-prometheus-stack
├── kiali/values.yaml
├── apps-project/                    # AppProject `apps` + 3 env namespaces
└── cloudnative-pg/values.yaml
apps/                                # application workloads (DRY tree, hydrated by Source Hydrator)
├── postgres-aura/                   # main application DB
│   ├── base/                        # CNPG Cluster CR — name: postgres-aura, db: aura
│   └── overlays/{dev,staging,prod}/ # per-env storage / replica patches
├── postgres-keycloak/               # identity provider DB
│   ├── base/                        # name: postgres-keycloak, db: keycloak
│   └── overlays/{dev,staging,prod}/
├── postgres-spicedb/                # authorization (Zanzibar) DB
│   ├── base/                        # name: postgres-spicedb, db: spicedb
│   └── overlays/{dev,staging,prod}/
├── keycloak/                        # IdP
│   ├── base/                        # Deployment + Service + admin Secret + realm.json
│   └── overlays/{dev,prod}/         # per-env KC_HOSTNAME + VirtualService
└── report-ms/                       # Java backend (Spring Boot)
    ├── base/                        # Deployment + Service + ConfigMap (constants)
    └── overlays/{dev,prod}/         # per-env S3 bucket + VirtualService
```

## Pinned versions

| Component               | Version  | Notes                                              |
| :---------------------- | :------- | :------------------------------------------------- |
| ArgoCD Helm chart       | `9.5.17` | ships ArgoCD `v3.4.3`; Source Hydrator (alpha) on  |
| Istio                   | `1.30.0` | sidecar mode                                       |
| kube-prometheus-stack   | `86.1.0` | Prometheus Operator `v0.91.0`                      |
| Kiali                   | `2.27.0` | Helm chart `kiali/kiali-server`                    |
| CloudNativePG operator  | `0.28.2` | operator `v1.29.1` — manages all 3 env Postgres    |

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
| `3`  | `routing`, `argocd-routing` | Shared `public-gateway` (hosts `*.lab.lan`) + argocd's `VirtualService`. Every other service attaches a VirtualService later — no per-app Gateway needed. |
| `4`  | `kube-prometheus-stack` | Prometheus + Grafana + Alertmanager + node-exporter + kube-state-metrics. No hard dep on Istio at sync time, but scrapes envoy sidecars and istiod once they're up. |
| `5`  | `kiali`          | Service mesh dashboard. Reads from the Prometheus that wave 4 just installed. |
| `6`  | `apps-project`, `cloudnative-pg` | AppProject `apps` + 3 env namespaces (`apps-{dev,staging,prod}`) and the CNPG operator install in parallel. App workloads in wave 7 need both. |
| `7`  | `postgres-{aura,keycloak,spicedb}-{dev,staging,prod}` | Nine `Cluster` CRs — 3 databases × 3 envs. Each materialized via Source Hydrator from `apps/postgres-<db>/overlays/<env>` → `environments/<env>` branches. |
| `8`  | `keycloak-{dev,prod}` | IdP — needs `postgres-keycloak` (wave 7). |
| `9`  | `report-ms-{dev,prod}` | Java backend — needs `postgres-aura` (wave 7) for JDBC + Keycloak (wave 8) for OIDC. Reaches SpiceDB at `spicedb:50051` when that lands. |

Two things to remember:

- Waves order **the sync**, not the runtime. Once everything is Healthy, waves are irrelevant — the controller just watches drift on every resource equally.
- Waves work on a single Application (resources inside one Application sync) **and** at the App-of-Apps level (Application objects themselves sync in wave order). We use both: the four Applications above are wave-ordered, and individual resources inside an Application can carry their own sync-wave annotation if needed.

### Source Hydrator (alpha)

ArgoCD 3.4's "rendered manifest pattern." The Application's `spec.source` is replaced by `spec.sourceHydrator`:

- **`drySource`** — where humans author. Helm chart, Kustomize overlay, or raw YAML on `main`. Path: `apps/<svc>/overlays/<env>`.
- **`syncSource`** — where ArgoCD actually syncs from. ArgoCD's **commit server** renders `drySource` and pushes the rendered YAML to this branch. We use one branch per env: `environments/dev`, `environments/staging`, `environments/prod`.
- **`hydrateTo`** (optional, not used yet) — a review branch the hydrator writes *first*; a promotion process then fast-forwards `syncSource`. Equivalent to PR-style env promotion.

What this buys us vs. plain `spec.source`:

1. **Rendered diff in review** — pull requests against `environments/<env>` show the actual YAML delta a deploy will produce, not just "helm values changed." Promotion review surface is auditable.
2. **DRY authoring on one branch** — overlays live on `main`. Branch-per-env (the canonical anti-pattern) is avoided because humans never edit env branches; only the hydrator writes them.
3. **Per-env hydration isolation** — a render failure for `prod` doesn't block `dev`'s sync.

What it costs:

- A write-scoped repo Secret (`argocd.argoproj.io/secret-type=repository-write`). See "Enabling Source Hydrator" below.
- The `commitServer` component runs in `argocd` namespace (added to `core/argocd/values.yaml` with `commitServer.enabled: true`).
- Still alpha in 3.4.3 — spec/path semantics have shifted across 3.1 → 3.2 → 3.3; pin the chart version and read the upgrade notes before bumping.

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

### Enabling Source Hydrator (one-time, after initial bootstrap)

The argocd values commit turns on `commitServer.enabled: true` and `hydrator.enabled: true`. Two more pieces have to land out-of-band before any Application with a `sourceHydrator` field can sync:

```bash
# 1. Create a separate WRITE-scoped GitHub PAT:
#    Permissions → Contents: Read AND write
#    (Same repo selection as the existing read PAT.)

# 2. Register it as a write secret (distinct from the existing read secret).
read -s -p "WRITE PAT: " PAT && echo
kubectl -n argocd create secret generic aura-gitops-repo-write \
  --from-literal=type=git \
  --from-literal=url=https://github.com/AuraAITools/gitops.git \
  --from-literal=username=<gh-username> \
  --from-literal=password="$PAT"
unset PAT
kubectl -n argocd label secret aura-gitops-repo-write \
  argocd.argoproj.io/secret-type=repository-write

# 3. Create the env branches as empty orphans. The hydrator can also create
#    these on first run, but pre-creating avoids a chicken-and-egg if it
#    doesn't auto-create on your version.
for env in dev staging prod; do
  git checkout --orphan "environments/$env"
  git rm -rf . >/dev/null 2>&1 || true
  git commit --allow-empty -m "init environments/$env"
  git push origin "environments/$env"
done
git checkout main
```

The pull secret (`aura-gitops-repo`, label `repository`) and the push secret (`aura-gitops-repo-write`, label `repository-write`) are deliberately separate — ArgoCD treats them as two different credentials so a leaked read PAT can't write.

### Imperative Secrets for `report-ms` (per env, until SOPS+age lands)

`report-ms` pulls from private ECR and consumes a bag of credentials. Until SOPS ships, both Secrets are created imperatively per env. The Deployment references them by name (`ecr-pull`, `report-ms-secrets`) — without them, pods sit in `ImagePullBackOff` or `CreateContainerConfigError`.

**1. ECR pull secret** (`ecr-pull`) — needs to land in every `apps-<env>` namespace. ECR tokens expire after 12 hours, so production wants the CronJob rotator from `AURA_STACK_DEPLOYMENT.md` §6.1; for first-light dev work, the one-shot below is enough:

```bash
AWS_REGION=ap-southeast-1
AWS_ACCT=696085047789
for ns in apps-dev apps-prod; do
  TOKEN="$(aws ecr get-login-password --region "$AWS_REGION")"
  kubectl -n "$ns" create secret docker-registry ecr-pull \
    --docker-server="${AWS_ACCT}.dkr.ecr.${AWS_REGION}.amazonaws.com" \
    --docker-username=AWS \
    --docker-password="$TOKEN" \
    --dry-run=client -o yaml | kubectl apply -f -
done
unset TOKEN
```

Re-run before the 12h ECR token expires, or set up the CronJob rotator. (Backlog item.)

**2. `report-ms-secrets`** — sensitive env vars consumed via `envFrom: secretRef`. Replace the placeholder values with real ones before running:

```bash
for ns in apps-dev apps-prod; do
  kubectl -n "$ns" create secret generic report-ms-secrets \
    --from-literal=KEYCLOAK_CLIENT_SECRET='super-secret-shit' \
    --from-literal=KEYCLOAK_CLIENT_UUID='d388a85a-26ed-48e1-a9a4-57b87c77dc61' \
    --from-literal=SPICEDB_PRESHARED_KEY='dev-secret-key' \
    --from-literal=AWS_ACCESS_KEY='test' \
    --from-literal=AWS_SECRET_ACCESS_KEY='REPLACE_ME' \
    --from-literal=MAIL_PASSWORD='REPLACE_ME_GMAIL_APP_PASSWORD' \
    --dry-run=client -o yaml | kubectl apply -f -
done
```

Keys must match `base/configmap.yaml`'s `envFrom` contract:
- `KEYCLOAK_CLIENT_SECRET`, `KEYCLOAK_CLIENT_UUID` — get from Keycloak's `aura-application-client` (Credentials tab).
- `SPICEDB_PRESHARED_KEY` — same value that SpiceDB is configured with (`--grpc-preshared-key`); the SpiceDB Application will inject it on its side.
- `AWS_ACCESS_KEY` / `AWS_SECRET_ACCESS_KEY` — IAM user with S3 access to the per-env bucket (`aura-dev`, `aura-prod`).
- `MAIL_PASSWORD` — Gmail app-specific password.

Different values per env? Run the `for` loop with two separate calls. The bucket name itself is **not** here — it lives in `apps/report-ms/overlays/<env>/kustomization.yaml` as `AWS_S3_BUCKET_NAME`.

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

# Expect (19 Applications total):
#   root
#   argocd, istio-base, istiod, istio-gateway, argocd-routing
#   kube-prometheus-stack, kiali
#   apps-project, cloudnative-pg
#   postgres-aura-{dev,staging,prod}
#   postgres-keycloak-{dev,staging,prod}
#   postgres-spicedb-{dev,staging,prod}
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

- [ ] **ECR pull-credential rotator** — CronJob in each `apps-<env>` ns that re-runs `aws ecr get-login-password` every 8h and overwrites the `ecr-pull` Secret. Until this lands, the one-shot `kubectl create secret docker-registry` above has a 12-hour expiry window. See `AURA_STACK_DEPLOYMENT.md` §6.1.
- [ ] **Secrets management — SOPS + age.** Per the strategy doc, Sealed Secrets is an anti-pattern for this setup (single controller-key fragility, per-cluster ciphertext, manual DR). The chosen direction is SOPS + age with a KSOPS sidecar in `argocd-repo-server`. Removes the imperative PAT bootstrap and lets app secrets live encrypted in Git.
- [ ] **cert-manager** for TLS at `*.lab.lan`. Options: self-signed CA via the cert-manager bootstrap Issuer (zero external deps, browser warnings) or Let's Encrypt via DNS-01 if the lab domain ever becomes routable. Currently all routing is plain HTTP, which is OK on the LAN only.
- [ ] **Istio routing for Grafana + Kiali** — `grafana.lab.lan` / `kiali.lab.lan` VirtualServices attaching to the shared `public-gateway`.
- [x] ~~**`public-gateway` refactor**~~ — landed. One Gateway in `istio-ingress` listens on `*.lab.lan`; per-service `VirtualService` lives with the app. Adding a new public host is now a single-resource change.
- [ ] **Tracing** — Tempo (or Jaeger), wired into Kiali's `external_services.tracing`.
- [x] ~~**Source Hydrator enablement**~~ — landed with Postgres (`apps/postgres/overlays/<env>` → `environments/<env>` branches). Spec is `spec.sourceHydrator` on every app Application; see Concepts.
- [ ] **CNPG backups** — `Cluster.spec.backup` to S3/MinIO with PITR. Required before any real prod use. Currently marked `# TODO` in `apps/postgres/overlays/prod/kustomization.yaml`.
- [ ] **Postgres connection from app pods** — each cluster auto-generates `postgres-<db>-app` Secret per env (so `postgres-aura-app`, `postgres-keycloak-app`, `postgres-spicedb-app` in each `apps-<env>` namespace). App workloads consume it via env vars or projected files.
- [ ] **ApplicationSet refactor** — at 9 Postgres Applications, the duplication in `bootstrap/apps/postgres-*.yaml` is real. A matrix generator over `(db, env)` would collapse all 9 into a single ApplicationSet. Defer until Source Hydrator's behavior is settled (don't compound two alpha-adjacent features).
