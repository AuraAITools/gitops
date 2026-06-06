# gitops

GitOps config repo for AuraAITools. ArgoCD reconciles every cluster from this repo. See [`GITOPS_STRATEGY.md`](./GITOPS_STRATEGY.md) for the architecture rationale.

## Dashboards

| Dashboard   | URL (once routing is up) | Port-forward fallback                                                                | What it shows                                                                  |
| :---------- | :----------------------- | :----------------------------------------------------------------------------------- | :----------------------------------------------------------------------------- |
| **ArgoCD**  | http://argocd.lab.lan    | `kubectl -n argocd port-forward svc/argocd-server 8080:80` → http://localhost:8080   | Applications, sync status, sync history, drift, manual sync.                   |
| **Grafana** | http://grafana.lab.lan ⏳ | `kubectl -n monitoring port-forward svc/kube-prometheus-stack-grafana 3000:80` → http://localhost:3000 | Cluster + node metrics out of the box. Istio's canonical dashboards (Mesh / Service / Workload / Performance / Control Plane / Extension) appear under the **Istio** folder — pulled from `istio/istio@release-1.30` at helm-template time. |
| **Kiali**   | http://kiali.lab.lan ⏳   | `kubectl -n kiali port-forward svc/kiali 20001:20001` → http://localhost:20001       | Service mesh topology, traffic graph, per-service request rate / error rate.   |
| **Vault**   | http://vault.lab.lan      | `kubectl -n vault port-forward svc/vault 8200:8200` → http://localhost:8200          | KV secret store; auth methods; policies. See **Managing secrets**.             |

⏳ = `VirtualService` not yet committed; see [Backlog](#backlog).

**Initial credentials** (rotate after first login):
- ArgoCD: `admin` / `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d`
- Grafana: `admin` / `admin` (set in `core/monitoring/values.yaml`)
- Kiali: anonymous (homelab only — change `auth.strategy` before any shared env)
- Vault: **root token** from first-time `vault operator init` — see **Managing secrets** below for the one-time unseal/init runbook

## Application endpoints

External URLs follow the homelab pattern `<svc>[-<env>].lab.lan`. All public-facing services attach to a single shared `*.lab.lan` Istio `Gateway` (`istio-ingress/public-gateway`, installed by the `routing` Application at sync-wave 3); each app's own `VirtualService` adds its concrete host.

Hosts shown with ⏳ are waiting on either the per-service `VirtualService` to land in this repo **or** the hostname being added to your DNS (see [DNS setup](#dns-setup) below). Hosts shown with ❌ are for services not yet deployed.

### Per-environment hosts

| Service              | Dev                              | Staging                              | Prod                          | Status |
| :------------------- | :------------------------------- | :----------------------------------- | :---------------------------- | :----- |
| **BFF** (aura-report-website) | `app-dev.lab.lan` ✅      | `app-staging.lab.lan` ❌             | `app.lab.lan` ✅              | dev + prod deployed (needs `ecr-pull` + `bff-secrets` imperative bootstrap, see below); staging not deployed |
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
    ├── vault.yaml                   # HashiCorp Vault (KV store)    (sync-wave 5)
    ├── vault-routing.yaml           # vault.lab.lan VirtualService  (sync-wave 5)
    ├── apps-project.yaml            # AppProject + apps-{dev,staging,prod} ns (wave 6)
    ├── cloudnative-pg.yaml          # CNPG operator                 (sync-wave 6)
    ├── ecr-rotator.yaml             # ECR pull-cred CronJob         (sync-wave 6)
    ├── postgres-<db>-<env>.yaml     # 3 databases × 3 envs = 9 Apps (sync-wave 7)
    │                                # db ∈ {aura, keycloak, spicedb}
    │                                # env ∈ {dev, staging, prod}
    ├── keycloak-{dev,prod}.yaml     # IdP                           (sync-wave 8)
    ├── report-ms-{dev,prod}.yaml    # Java backend                  (sync-wave 9)
    └── aura-report-website-{dev,prod}.yaml  # Next.js BFF           (sync-wave 10)
core/                                # values / overlays for platform components
├── argocd/
│   ├── values.yaml
│   └── routing/                     # VirtualService → argocd.lab.lan (attaches to public-gateway)
├── routing/
│   └── public-gateway.yaml          # shared Gateway in istio-ingress; hosts: ["*.lab.lan"]
├── istio/{istiod,gateway}-values.yaml
├── monitoring/values.yaml           # kube-prometheus-stack
├── kiali/values.yaml
├── vault/                           # Helm values + VirtualService for vault.lab.lan
├── apps-project/                    # AppProject `apps` + 3 env namespaces
├── cloudnative-pg/values.yaml
└── ecr-rotator/                     # CronJob + RBAC; refreshes ecr-pull Secret in apps-* every 8h
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
├── report-ms/                       # Java backend (Spring Boot)
│   ├── base/                        # Deployment + Service + ConfigMap (constants)
│   └── overlays/{dev,prod}/         # per-env S3 bucket + VirtualService
└── aura-report-website/             # Next.js BFF
    ├── base/                        # Deployment + Service + ConfigMap
    └── overlays/{dev,prod}/         # per-env NEXTAUTH_URL + KEYCLOAK_ISSUER + VirtualService
```

## Pinned versions

| Component               | Version  | Notes                                              |
| :---------------------- | :------- | :------------------------------------------------- |
| ArgoCD Helm chart       | `9.5.17` | ships ArgoCD `v3.4.3`; Source Hydrator (alpha) on  |
| Istio                   | `1.30.0` | sidecar mode                                       |
| kube-prometheus-stack   | `86.1.0` | Prometheus Operator `v0.91.0`                      |
| Kiali                   | `2.27.0` | Helm chart `kiali/kiali-server`                    |
| CloudNativePG operator  | `0.28.2` | operator `v1.29.1` — manages all 3 env Postgres    |
| HashiCorp Vault         | `1.18.0` | chart `0.29.1`; single-node Raft + Shamir seal     |

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
| `5`  | `kiali`, `vault`, `vault-routing` | Service mesh dashboard; HashiCorp Vault (single-node Raft, manual Shamir unseal); Vault `VirtualService`. No deps between them — all install in parallel. |
| `6`  | `apps-project`, `cloudnative-pg`, `ecr-rotator` | AppProject `apps` + 3 env namespaces, CNPG operator, and the ECR pull-cred rotator. All three install in parallel. Workloads in wave 7+ depend on namespaces + CNPG + `ecr-pull` Secret. |
| `7`  | `postgres-{aura,keycloak,spicedb}-{dev,staging,prod}` | Nine `Cluster` CRs — 3 databases × 3 envs. Each materialized via Source Hydrator from `apps/postgres-<db>/overlays/<env>` → `environments/<env>` branches. |
| `8`  | `keycloak-{dev,prod}` | IdP — needs `postgres-keycloak` (wave 7). |
| `9`  | `report-ms-{dev,prod}` | Java backend — needs `postgres-aura` (wave 7) for JDBC + Keycloak (wave 8) for OIDC. Reaches SpiceDB at `spicedb:50051` when that lands. |
| `10` | `aura-report-website-{dev,prod}` | Next.js BFF — proxies report-ms and redirects browsers to Keycloak's public hostname. |

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
printf 'PAT: '; read -s PAT; echo
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
printf 'WRITE PAT: '; read -s PAT; echo
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

### ECR pull-credential rotator (one-time bootstrap)

The `ecr-rotator` Application installs a `CronJob` in the `ecr-rotator` namespace that runs every 8 hours, calls `aws ecr get-login-password`, and overwrites the `ecr-pull` Secret in `apps-{dev,staging,prod}`. ECR tokens expire after 12h, so the 8h schedule gives a 4h grace window.

One imperative step — the AWS credentials Secret. Everything else is GitOps.

**1. Create a minimal-permissions IAM user for the rotator.** In the AWS console (or via CLI), create a new IAM user `aura-ecr-rotator` with an inline policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "ecr:GetAuthorizationToken",
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecr:BatchGetImage",
        "ecr:GetDownloadUrlForLayer"
      ],
      "Resource": "arn:aws:ecr:ap-southeast-1:696085047789:repository/aura"
    }
  ]
}
```

Create an access key for this user (Security credentials → Create access key → Application running outside AWS). Copy the access key + secret.

**2. Drop the access key into the `aws-credentials` Secret** (one-shot, lives only on the cluster — not in git):

```bash
printf 'AWS_ACCESS_KEY_ID: '; read AWS_AK
printf 'AWS_SECRET_ACCESS_KEY: '; read -s AWS_SK; echo
kubectl -n ecr-rotator create secret generic aws-credentials \
  --from-literal=AWS_ACCESS_KEY_ID="$AWS_AK" \
  --from-literal=AWS_SECRET_ACCESS_KEY="$AWS_SK" \
  --dry-run=client -o yaml | kubectl apply -f -
unset AWS_AK AWS_SK
```

(The `ecr-rotator` namespace is created by the Application at sync-wave 6; if you're running this before that synced, `kubectl create ns ecr-rotator` first.)

**3. Kick off the first run** so you don't wait up to 8h for the first scheduled execution:

```bash
kubectl -n ecr-rotator create job --from=cronjob/ecr-pull-rotator ecr-pull-bootstrap
kubectl -n ecr-rotator logs -f job/ecr-pull-bootstrap
```

You should see `token-bytes=<some number>` then three `secret/ecr-pull configured` lines (one per env namespace). Confirm:

```bash
kubectl get secret ecr-pull -n apps-dev -n apps-prod -n apps-staging --ignore-not-found
```

After that, any `ImagePullBackOff` pods will pick up the Secret on the next pull retry — or force it with `kubectl rollout restart deploy/<name> -n <ns>`.

### Imperative Secrets for `report-ms` (per env, until SOPS+age lands)

`report-ms-secrets` — sensitive env vars consumed via `envFrom: secretRef`. Replace the placeholder values with real ones before running:

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

### Imperative Secrets for `aura-report-website` (BFF, per env)

Two sensitive env vars: NextAuth session secret + the Keycloak client secret (same `aura-application-client` as report-ms, so reuse the value).

```bash
# AUTH_SECRET: generate fresh per env. 32 random bytes, base64-encoded.
AUTH_SECRET_DEV="$(openssl rand -base64 32)"
AUTH_SECRET_PROD="$(openssl rand -base64 32)"

# Keycloak client secret — same `aura-application-client` as report-ms.
# Grab it from Keycloak admin: Clients → aura-application-client → Credentials.
printf 'KEYCLOAK_CLIENT_SECRET: '; read -s KC_SECRET; echo

for ns in apps-dev apps-prod; do
  AUTH="$( [ "$ns" = apps-dev ] && echo "$AUTH_SECRET_DEV" || echo "$AUTH_SECRET_PROD" )"
  kubectl -n "$ns" create secret generic bff-secrets \
    --from-literal=AUTH_SECRET="$AUTH" \
    --from-literal=KEYCLOAK_CLIENT_SECRET="$KC_SECRET" \
    --dry-run=client -o yaml | kubectl apply -f -
done
unset KC_SECRET AUTH_SECRET_DEV AUTH_SECRET_PROD AUTH
```

`AUTH_SECRET` must be unique per env (rotating it invalidates all existing NextAuth sessions in that env). `KEYCLOAK_CLIENT_SECRET` is shared with `report-ms`, so use the same value you put in `report-ms-secrets`.

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

# Expect (29 Applications total):
#   root
#   argocd, istio-base, istiod, istio-gateway, routing, argocd-routing
#   kube-prometheus-stack, kiali
#   vault, vault-routing
#   apps-project, cloudnative-pg, ecr-rotator
#   postgres-aura-{dev,staging,prod}
#   postgres-keycloak-{dev,staging,prod}
#   postgres-spicedb-{dev,staging,prod}
#   keycloak-{dev,prod}
#   report-ms-{dev,prod}
#   aura-report-website-{dev,prod}
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

## Managing secrets

Application secrets live in **HashiCorp Vault**, self-hosted in-cluster (`core/vault/`). The Vault Secrets Operator (VSO) — coming in a follow-up commit — watches `VaultStaticSecret` CRDs in each `apps-<env>` namespace and materializes normal `Secret` resources, which pods consume via `envFrom`. Until VSO + per-app migration land, the existing imperative `kubectl create secret` blocks still apply.

```
You:    vault kv put aura/dev/report-ms KEYCLOAK_CLIENT_SECRET=... MAIL_PASSWORD=...
        # via UI at http://vault.lab.lan, or `vault` CLI port-forwarded

Vault:  encrypted at rest on /vault/data (Raft storage), sealed/unsealed via
        Shamir's secret sharing (5 keys, 3 needed to unseal)

VSO:    watches a VaultStaticSecret CRD in apps-<env>; reads Vault as the
        per-app ServiceAccount via the Kubernetes auth method; writes a normal
        Secret into the namespace; re-syncs every 60s so `vault kv put` updates
        propagate without a redeploy

App:    consumes the Secret via envFrom — identical Deployment YAML to today
```

### Why Vault (and not SOPS / sealed-secrets / ESO)

- **Learning**: industry-standard tool; same patterns at scale apply at this scale.
- **No external deps**: self-hosted, no cloud KMS, no Docker Hub for the secret store itself.
- **Rotation is `vault kv put`** — no commit, no PR, no deploy. SOPS would need a re-encrypt + commit per rotation.
- **Dynamic secrets later**: PG dynamic credentials, short-lived AWS STS tokens — Vault's headline feature, available when we want it. Backlog.

What's lost vs. SOPS: Vault is one more service to operate. The unseal step is real (see below). For 3–5 secrets, SOPS would be lower-effort; we picked Vault as the explicit long-term move.

### Architecture choices

| Decision | Pick | Why |
| :--- | :--- | :--- |
| Storage backend | Raft (integrated) | No external KV; HashiCorp-recommended since 1.4 |
| Replicas | 1 | Single-node sufficient for homelab. Bump to 3 for HA when needed |
| Seal type | Shamir (default, manual unseal) | No external KMS dep. Trade-off: every Vault pod restart needs manual unseal |
| K8s integration | Vault Secrets Operator (VSO) | Official, modern. Produces normal `Secret` resources |
| Auth method | Kubernetes auth | Pods present their SA token; no per-app bootstrap secret |

### One-time bootstrap (after Vault Application syncs Healthy)

The Vault pod starts **sealed**. You need to initialize and unseal it once.

```bash
# 1. Initialize Vault — generates 5 unseal keys + 1 root token.
#    Save the entire output to your password manager. Losing all 5 unseal
#    keys means losing every secret Vault holds.
kubectl -n vault exec -it vault-0 -- vault operator init

# Output looks like:
#   Unseal Key 1: <base64>
#   Unseal Key 2: ...
#   Unseal Key 3: ...
#   Unseal Key 4: ...
#   Unseal Key 5: ...
#   Initial Root Token: hvs.<long-string>

# 2. Unseal — provide 3 of the 5 keys.
kubectl -n vault exec -it vault-0 -- vault operator unseal   # paste key 1
kubectl -n vault exec -it vault-0 -- vault operator unseal   # paste key 2
kubectl -n vault exec -it vault-0 -- vault operator unseal   # paste key 3
# Status flips to "Sealed: false"

# 3. Verify
kubectl -n vault exec -it vault-0 -- vault status
```

After this you can log in to the UI at `http://vault.lab.lan` (DNS entry required — see DNS setup above) with the root token. **Rotate the root token to a short-lived token via the UI's "Generate Root" flow once you've configured proper auth methods.** The bootstrap root token grants everything; only use it for initial setup.

### Pod restart unseal (the cost of no auto-unseal)

Every time `vault-0` restarts (cluster reboot, image bump, eviction), Vault re-seals automatically. You bring it back with the same 3-keys unseal flow — typically ~20 seconds of work, but you'll know about it because anything depending on Vault stops getting fresh secrets.

For a homelab on a Mac mini, this is acceptable. If it becomes painful, the fallback is **storing the unseal keys in a K8s Secret + an init container that auto-unseals on startup** — defeats most of the seal guarantee but reasonable on a single-node cluster with physical security. We'll add that pattern if needed; not yet.

### Bootstrap that stays imperative

- The 5 unseal keys + root token (kept in your password manager — by design, can't be in git).
- The per-environment Vault auth setup (Kubernetes auth method, policies, roles) — done with `vault` CLI or Terraform once. Comes in commit 2 of the Vault rollout.

Everything else lands via GitOps.

### Where things will live (once VSO is wired in)

```
core/vault/                            # Helm install + UI VirtualService
core/vault-secrets-operator/           # VSO Helm install + cluster-level config (commit 2)
apps/<svc>/overlays/<env>/
  secrets.vault.yaml                   # VaultStaticSecret CRD; references aura/<env>/<svc> path
```

Today: 3 imperative Secrets (`report-ms-secrets`, `bff-secrets`, `keycloak-admin-secret`).
After full Vault rollout: 0 imperative Secrets in the apps-* namespaces. Two remain: the `aws-credentials` Secret for the ECR rotator, and the Vault unseal keys (in your password manager, not the cluster).

## Re-bootstrap (cluster blown away)

The bootstrap is idempotent. Re-run steps 3 → 6 above against a fresh cluster. The `argocd` Application is annotated `sync-wave: -10` so it reconciles before everything else.

## Backlog

- [x] ~~**ECR pull-credential rotator**~~ — landed at `core/ecr-rotator/`. Single CronJob in `ecr-rotator` ns, ServiceAccount with per-namespace `Role`+`RoleBinding` in each `apps-<env>`, runs every 8h. Bootstrap = one imperative `aws-credentials` Secret + one `kubectl create job --from=cronjob/...`.
- [ ] **Vault Secrets Operator (VSO) + per-app migration** — commit 2 of the Vault rollout: install VSO at `core/vault-secrets-operator/`, configure Kubernetes auth method, then migrate `report-ms-secrets` / `bff-secrets` / `keycloak-admin-secret` to `VaultStaticSecret` CRDs.
- [ ] **Vault auto-unseal (pragmatic homelab pattern)** — only if manual unseal becomes painful. Likely shape: init container reads unseal keys from a K8s Secret on startup and calls `vault operator unseal`. Weakens the seal guarantee but acceptable on a single-node cluster.
- [ ] **Dynamic Vault secrets (Postgres + AWS STS)** — Vault's headline feature. Use the `database` engine to issue short-lived PG credentials per app, and the `aws` engine to mint STS tokens for the ECR rotator (replacing the static AWS access key). Far enough out that we don't need to plan it now.
- [x] ~~**Secrets management strategy**~~ — chose Vault over SOPS+age for the long-term path. See [Managing secrets](#managing-secrets). Vault server lands at sync-wave 5; VSO + per-app migration are follow-up commits.
- [ ] **cert-manager** for TLS at `*.lab.lan`. Options: self-signed CA via the cert-manager bootstrap Issuer (zero external deps, browser warnings) or Let's Encrypt via DNS-01 if the lab domain ever becomes routable. Currently all routing is plain HTTP, which is OK on the LAN only.
- [ ] **Istio routing for Grafana + Kiali** — `grafana.lab.lan` / `kiali.lab.lan` VirtualServices attaching to the shared `public-gateway`.
- [x] ~~**`public-gateway` refactor**~~ — landed. One Gateway in `istio-ingress` listens on `*.lab.lan`; per-service `VirtualService` lives with the app. Adding a new public host is now a single-resource change.
- [ ] **Tracing** — Tempo (or Jaeger), wired into Kiali's `external_services.tracing`.
- [x] ~~**Source Hydrator enablement**~~ — landed with Postgres (`apps/postgres/overlays/<env>` → `environments/<env>` branches). Spec is `spec.sourceHydrator` on every app Application; see Concepts.
- [ ] **CNPG backups** — `Cluster.spec.backup` to S3/MinIO with PITR. Required before any real prod use. Currently marked `# TODO` in `apps/postgres/overlays/prod/kustomization.yaml`.
- [ ] **Postgres connection from app pods** — each cluster auto-generates `postgres-<db>-app` Secret per env (so `postgres-aura-app`, `postgres-keycloak-app`, `postgres-spicedb-app` in each `apps-<env>` namespace). App workloads consume it via env vars or projected files.
- [ ] **ApplicationSet refactor** — at 9 Postgres Applications, the duplication in `bootstrap/apps/postgres-*.yaml` is real. A matrix generator over `(db, env)` would collapse all 9 into a single ApplicationSet. Defer until Source Hydrator's behavior is settled (don't compound two alpha-adjacent features).
