# Aura Stack — Kubernetes Deployment Architecture

**Drafted:** 2026-06-03
**Updated:** 2026-06-07
**Status:** Mostly implemented — see §2 for per-layer status. SpiceDB is the only major design item still pending — current plan is `spicedb-operator` (CR-driven) rather than a hand-written Deployment.
**Scope:** How `aura-report-website`, `report-ms`, Keycloak, and SpiceDB land on the gitops-managed cluster across `dev` / `staging` / `prod`.

This is a companion to [`GITOPS_STRATEGY.md`](./GITOPS_STRATEGY.md) (the *how we deploy*) and [`README.md`](./README.md) (the *what's currently deployed*). The README is the live-truth operational doc; this doc captures the design decisions and trade-offs we made along the way.

### Deviations from the original draft
- **Domain**: `auraenterprise.solutions` everywhere — `auratest.dev` was deprecated, `lab.lan` was never wired up.
- **Public ingress**: Cloudflare Tunnel (in-cluster `cloudflared` Deployment) terminates TLS at Cloudflare's edge and tunnels to the Istio gateway. No public LoadBalancer or DNS infrastructure on the Mac mini.
- **Secrets**: HashiCorp Vault (self-hosted, single-node Raft, Shamir seal) + Vault Secrets Operator (VSO) — chose this over the original SOPS+age plan for the learning value + dynamic-secrets ceiling.
- **Cluster**: `kind` on a single Mac mini (not k3d as originally assumed).
- **SpiceDB**: will land via the official [`spicedb-operator`](https://authzed.com/docs/spicedb/ops/operator) (CR-driven) rather than a hand-written Deployment + init-migrate Job. The standalone `postgres-spicedb` CNPG cluster was torn down (2026-06-07) — the operator stack will pick its own datastore strategy.

---

## 1. Overview

```
   ┌──────────────────┐  HTTPS  ┌────────────────────────┐  HTTP  ┌──────────────────────┐
   │   Browser/User   │ ◀─────▶ │ Cloudflare edge (TLS)  │ ◀────▶ │ cloudflared (in-cl.) │
   └──────────────────┘         │ *.auraenterprise.solut │        │ → istio gateway      │
                                └────────────────────────┘        └─────────┬────────────┘
                                                                            │
              ┌─────────────────────────────────────────────────────────────┼─────────────┐
              ▼                                                             ▼             ▼
      ┌──────────────────┐       ┌────────────────────┐         ┌────────────────────┐
      │ aura-report-     │       │ keycloak           │         │ report-ms          │
      │   website (BFF)  │       │ (accounts.*)       │         │ (api.*)            │
      │ Next.js, :3000   │       │ quay.io, :8080     │         │ Spring Boot, :8080 │
      └────────┬─────────┘       └──────────┬─────────┘         └──┬─────────────┬───┘
               │                            ▲                      │             │
               │ OIDC redirect (browser)    │ admin API + JWK       │             │
               │  + JWT validate via JWK    │ (server-to-server     │             │
               ├───────────────────────────►│  from report-ms)      │             │
               │                            │◄──────────────────────┤             │
               │ proxy /graphql, /api/v1                            │ gRPC :50051 │
               └────────────────────────────────────────────────────┘             │
                                                                                 ▼
                                                                       ┌─────────────────────┐
                                                                       │ spicedb             │
                                                                       │ authzed, :50051gRPC │
                                                                       │   + :8443 HTTP      │
                                                                       └─────────┬───────────┘
                                                                                 │
       ┌─────────────────────────┬─────────────────────────────────────┐
       ▼                         ▼                                     ▼
┌──────────────────┐    ┌──────────────────┐         ┌────────────────────────────────┐
│ postgres-aura    │    │ postgres-keycloak│         │ SpiceDB datastore              │
│ CNPG, PG 18.0    │    │ CNPG, PG 16.4    │         │ (operator-pending; engine TBD: │
│ db: aura         │    │ db: keycloak     │         │  memory for dev, CNPG for prod)│
│ public + audit   │    │ self-init        │         └────────────────────────────────┘
└──────────────────┘    └──────────────────┘
```

Per environment (`apps-dev`, `apps-staging`, `apps-prod`):
- **2 Postgres clusters** scaffolded (`postgres-aura`, `postgres-keycloak`). SpiceDB's datastore is left to the operator stack — see §3.4.
- **3 stateful services** — Keycloak, SpiceDB (via operator), report-ms.
- **1 edge service** — aura-report-website.
- **1 Istio Gateway + multiple VirtualServices** — to expose BFF, Keycloak public/admin, and report-ms API.

---

## 2. What's already in place vs. what's new

| Layer | Status | Notes |
| :--- | :--- | :--- |
| Cluster, ArgoCD, Istio, observability | ✅ Done | Per `README.md`. |
| CNPG operator | ✅ Done | At sync-wave 6. |
| Postgres clusters (aura / keycloak) × 3 envs | ✅ Done | At sync-wave 7. Version-pinned per §4. The original `postgres-spicedb` CNPG cluster was torn down (2026-06-07); the operator stack chooses its own datastore. |
| Public ingress (Cloudflare Tunnel + shared Istio `public-gateway` on `*.auraenterprise.solutions`) | ✅ Done | `core/cloudflared/` (in-cluster) + `core/routing/public-gateway.yaml`. Cloudflare wildcard tunnel rule routes everything; Istio VirtualServices select the right Service by Host. |
| ECR image pull credentials | ✅ Done | `core/ecr-rotator/` CronJob refreshes `ecr-pull` Secret in each `apps-<env>` every 8h. Per §6.1. |
| Keycloak Deployment + routing (dev + prod) | ✅ Done | Per `apps/keycloak/`. Image is still the upstream `quay.io/keycloak/keycloak:25.0`; custom theme image is pending (§6.2). |
| report-ms Deployment + routing (dev + prod) | ✅ Done | Per `apps/report-ms/`. Secrets are now Vault-managed via VSO. |
| BFF (aura-report-website) Deployment + routing (dev + prod) | ✅ Done | Per `apps/aura-report-website/`. Wired to `app.auraenterprise.solutions` / `app-dev.auraenterprise.solutions`. |
| Secrets management (HashiCorp Vault + VSO) | ✅ Done | Vault server + VSO + per-app `VaultStaticSecret` for report-ms. `bff-secrets` and `keycloak-admin-secret` migration pending. |
| Liquibase migrations (report-ms) | ✅ Built into app | Spring Boot auto-runs on startup; no Job needed. |
| SpiceDB via [`spicedb-operator`](https://authzed.com/docs/spicedb/ops/operator) | ❌ Not yet | Operator install (cluster-scoped) + per-env `SpiceDBCluster` CR + schema-apply Job. Replaces the original hand-written Deployment + init-migrate plan. |
| Keycloak custom-theme image | ❌ Not yet | Themes baked into a custom image (§6.2); realm-import.json already mounted as ConfigMap with the upstream image. |

---

## 3. Service inventory (the load-bearing facts)

### 3.1 aura-report-website (Next.js BFF)
- **Image:** `696085047789.dkr.ecr.ap-southeast-1.amazonaws.com/aura:aura-report-website`
- **Listens on:** `:3000`
- **Runtime:** Node.js 24.15.0-alpine, Next.js standalone build.
- **Memory:** ~300 MiB (docker-compose baseline).
- **Required env vars:**
  - `AUTH_SECRET` (NextAuth session secret) — Secret
  - `NEXTAUTH_URL` — `https://<env-host>` (env-specific)
  - `KEYCLOAK_CLIENT_ID` — `aura-application-client`
  - `KEYCLOAK_CLIENT_SECRET` — Secret
  - `KEYCLOAK_ISSUER` — `https://accounts.<env-host>/realms/aura`
  - `NEXT_PUBLIC_REPORT_SERVICE_URL` — `https://api.<env-host>`
  - `NODE_ENV=production`
- **Reaches:** Keycloak (issuer, browser-side redirect); report-ms (proxies `/graphql` + `/api/v1/*` server-side).
- **Health:** No documented endpoint; rely on TCP ready on `:3000`.

### 3.2 report-ms (Java Spring Boot 4.0.2 / Java 25)
- **Image:** `696085047789.dkr.ecr.ap-southeast-1.amazonaws.com/aura:report-ms`
- **Listens on:** `:8080` — GraphQL at `/graphql`, REST at `/api/v1/*`, Actuator at `/actuator/*`.
- **Memory:** ~700 MiB.
- **Required env vars:**
  - `DB_URL` — `jdbc:postgresql://postgres-aura-rw.apps-<env>.svc:5432/aura`
  - `DB_USERNAME` / `DB_PASSWORD` — from CNPG's auto-generated `postgres-aura-app` Secret
  - `KEYCLOAK_CLIENT_ID`, `KEYCLOAK_CLIENT_SECRET`, `KEYCLOAK_CLIENT_SERVER_URL`, `KEYCLOAK_CLIENT_UUID`, `JWK_SET_URI` — OIDC + admin API
  - `SPRING_PROFILES_ACTIVE=cloud`
  - `AWS_REGION=ap-southeast-1`, `AWS_S3_BUCKET_NAME=<env-specific>`, AWS credentials
  - `SPICEDB_TARGET=spicedb.apps-<env>.svc:50051`
  - `SPICEDB_PRESHARED_KEY` — Secret
  - `SPICEDB_PLAINTEXT=true` (homelab) / `false` (prod with mTLS via Istio)
  - Email config (`AURA_EMAIL_*`) — optional
- **Health:** `/actuator/health/live`, `/actuator/health/ready`.
- **Reaches:** postgres-aura (JDBC), Keycloak (admin API + JWK), SpiceDB (gRPC), AWS S3.
- **Migrations:** Liquibase, runs on startup (changelog at `database-runner/.../changelog-master.yaml`). No separate Job needed.

### 3.3 Keycloak
- **Image:** `quay.io/keycloak/keycloak:25.0`
- **Listens on:** `:8080`
- **DB:** `postgres-keycloak` (PG 16.4 — stays inside Keycloak 25's official support matrix of PG 13–16).
- **Required config (start args + env):**
  - `--import-realm` startup arg
  - `KC_DB=postgres`, `KC_DB_URL=jdbc:postgresql://postgres-keycloak-rw.apps-<env>.svc:5432/keycloak`
  - `KC_DB_USERNAME` / `KC_DB_PASSWORD` — from CNPG's `postgres-keycloak-app` Secret
  - `KC_HOSTNAME=https://accounts.<env-host>`
  - `KC_PROXY_HEADERS=xforwarded`
  - `KC_FEATURES=hostname:v2`
  - `KEYCLOAK_ADMIN` / `KEYCLOAK_ADMIN_PASSWORD` — bootstrap admin
- **Volumes mounted:**
  - `/opt/keycloak/data/import/realm-import.json` — from ConfigMap (or initContainer pulling from auth-server repo)
  - `/opt/keycloak/themes/aura` and `/opt/keycloak/themes/aura-admin` — **baked into a custom image** (see §6)
- **Reaches:** postgres-keycloak only.
- **Realm:** `aura` (id `095811b5-918c-458e-8551-bfbb18c0b108`), clients `aura-application-client`, `backend-client`, `mobile-client`.

### 3.4 SpiceDB (via `spicedb-operator`)
The original plan was a hand-written Deployment + init-container `migrate head` + schema-apply Job, backed by a dedicated CNPG `postgres-spicedb` cluster. We pivoted (2026-06-07) to the official operator pattern; the postgres-spicedb tree was deleted and the per-env Applications removed from `bootstrap/apps/`.

- **Install:** cluster-scoped operator from upstream bundle manifest (or chart). Lives at `core/spicedb-operator/`, sync-wave 5 alongside the other operators. CRDs: `SpiceDBCluster`, `AuthzedEnterpriseCluster`.
- **Per env:** one `SpiceDBCluster` CR in `apps-<env>` (e.g. `dev-spicedb`). Operator reconciles → Deployment + Service + (optional) migrations Job.
- **Datastore:** chosen per env via `spec.config.datastoreEngine`:
  - **Dev / staging:** `memory` (no datastore — lossy across restarts, fine for testing). Zero-infra option.
  - **Prod:** `postgres` — reintroduce a CNPG `postgres-spicedb` cluster (PG 18 + `track_commit_timestamp=on`) when prod traffic justifies it.
- **Secret:** `SpiceDBCluster` references a Secret containing `preshared_key` (and `datastore_uri` when not in-memory). Both materialized via Vault Secrets Operator (`apps/spicedb/overlays/<env>/vaultstaticsecret.yaml`).
- **Service name:** `<spicedb-cluster-name>-spicedb` (e.g. `dev-spicedb.apps-dev.svc:50051`). Update `report-ms`'s `SPICEDB_TARGET` to match once we settle on a name convention.
- **Schema bootstrap:** still our concern. 14 `.zed` files in `report-ms/application/src/main/resources/spicedb/`. Apply via a Job (`apps/spicedb/base/schema-apply-job.yaml`) using `zed schema write index.zed`. Idempotent — re-applies every sync.
- **Reaches:** the configured datastore (none / postgres) only.

---

## 4. Postgres adjustments needed (modify the existing CNPG `Cluster` CRs)

The current `apps/postgres-*/base/cluster.yaml` files have generic settings. Two of three clusters move to PG 18.0; postgres-keycloak stays on 16.4 to remain inside Keycloak 25's official support matrix.

| Cluster | Edit | Why |
| :--- | :--- | :--- |
| `postgres-aura` | Pin `spec.imageName` to `ghcr.io/cloudnative-pg/postgresql:18.0`. | report-ms's Liquibase changelog uses standard DDL; PG 18 is a strict superset. Verify on first dev sync. |
| `postgres-keycloak` | Pin `spec.imageName` to `ghcr.io/cloudnative-pg/postgresql:16.4`. | Keycloak 25 officially supports PG 13–16; staying inside that matrix avoids vendor-untested behavior. |
| ~~`postgres-spicedb`~~ | ~~PG 18 + `track_commit_timestamp=on`~~ | Removed 2026-06-07 (operator pivot, §3.4). Will be reintroduced for prod only if/when SpiceDB needs a persistent datastore. |

These are diffable, minimal edits. No restructuring of the postgres tree.

---

## 5. Repo layout — proposed additions

Follow the established pattern: each app gets a DRY tree under `apps/<svc>/{base,overlays/<env>}/`, and 3 Applications under `bootstrap/apps/<svc>-<env>.yaml`, all using `spec.sourceHydrator`.

```
apps/
├── postgres-{aura,keycloak}/           # existing (postgres-spicedb removed 2026-06-07)
├── keycloak/
│   ├── base/                           # Deployment, Service, ConfigMap (realm-import), ServiceAccount
│   └── overlays/{dev,staging,prod}/    # replicas, resource limits, KC_HOSTNAME, image tag
├── spicedb/                            # operator-managed (planned)
│   ├── base/                           # schema-apply Job + .zed ConfigMap (datastore-agnostic)
│   └── overlays/{dev,staging,prod}/    # SpiceDBCluster CR, VaultStaticSecret for preshared_key, datastore choice
├── report-ms/
│   ├── base/                           # Deployment, Service, env/Secret refs, actuator probes
│   └── overlays/{dev,staging,prod}/    # replicas, resource limits, image tag, DB url, AWS_S3_BUCKET_NAME
└── aura-report-website/
    ├── base/                           # Deployment, Service, env/Secret refs
    └── overlays/{dev,staging,prod}/    # replicas, resource limits, image tag, NEXTAUTH_URL, etc.

bootstrap/apps/
├── postgres-*-<env>.yaml               # existing × 6 (2 dbs × 3 envs)
├── spicedb-operator.yaml               # cluster-scoped operator (sync-wave 5; planned)
├── keycloak-<env>.yaml                 # × 3
├── spicedb-<env>.yaml                  # × 3 (SpiceDBCluster CR per env)
├── report-ms-<env>.yaml                # × 3
├── aura-report-website-<env>.yaml      # × 3
└── (eventually one ApplicationSet replaces the per-env app entries; see §11)

core/
├── (existing) ...
└── routing/
    └── public-gateway.yaml             # shared gateway on *.auraenterprise.solutions (landed)
```

Per-env overlays patch what genuinely differs:
- `image:` tag (e.g. `aura:report-ms-<commit-sha>` — promoted via image-tag.yaml file the bot rewrites)
- `replicas`
- `resources.requests/limits`
- Hostnames embedded in env vars (`accounts.<env-host>`, `api.<env-host>`)

---

## 6. Custom image strategy (ECR auth + custom Keycloak)

### 6.1 ECR pull credentials
All four custom images live on private ECR. Two viable patterns:

| Pattern | Pros | Cons | Pick for homelab? |
| :--- | :--- | :--- | :--- |
| **Static `regcred` Secret rotated daily by CronJob** running `aws ecr get-login-password` | Simple, no IAM roles | 12-hour ECR token expiry; CronJob must run at < 12h interval. AWS keys in a Secret. | ✅ |
| **IRSA / pod identity** | No static creds | Requires EKS or IAM Roles for Service Accounts — not available on kind. | ❌ |
| **ECR credential helper as kubelet image-credential-provider** | Per-node, no Secret | Node-level config; kind nodes are Docker containers that get rebuilt frequently. | ❌ |

**Recommendation:** static Secret + CronJob that rotates every 8h. CronJob uses an IAM user with `ecr:GetAuthorizationToken` only.

### 6.2 Custom Keycloak image (for themes)
Themes are static assets that live in `auth-server` repo. Two ways:

| Option | How | Tradeoff |
| :--- | :--- | :--- |
| **A. Bake custom image** — `FROM quay.io/keycloak/keycloak:25.0; COPY themes/ /opt/keycloak/themes/`, push to ECR | Themes pinned to image tag, clean rollback | One more image to build/push |
| **B. ConfigMap volumes** | No new image | ConfigMap is limited to 1 MiB after compression; themes likely larger. Also: CSS/JS as ConfigMap entries is brittle. |
| **C. initContainer `git clone`** of `auth-server` repo at startup | Themes versioned via git | Pod startup depends on git availability; unauditable runtime drift |

**Recommendation:** A. Custom image, pushed to ECR as `aura:keycloak`. Build via the same CI flow as report-ms / BFF.

### 6.3 SpiceDB schema bootstrap
The 14 `.zed` files live with report-ms. With the operator pivot (§3.4) the SpiceDB Deployment itself is no longer our concern, but the schema apply is. Two ways to get them in:

| Option | How |
| :--- | :--- |
| **A. Job in `apps/spicedb/base/`** that mounts a ConfigMap of the .zed files and runs `zed schema write`. ConfigMap generated via Kustomize `configMapGenerator` from a local copy of the .zed files. Job waits for the operator-managed Service to be ready (probe `<name>-spicedb:50051`). | Schema lives in gitops repo; clean to diff. |
| **B. report-ms applies schema on startup** | No extra Job; coupling — schema can't deploy without report-ms |

**Recommendation:** A. Copy the `.zed` files into `apps/spicedb/base/schema/` and `configMapGenerator` them. Idempotent Job re-applies every sync.

---

## 7. Networking — Cloudflare Tunnel + Istio Gateway + VirtualServices

### 7.1 Public ingress architecture (✅ implemented)
The Mac mini is behind NAT; no public IPv4. Cloudflare Tunnel solves this without exposing any inbound port:

```
Browser → Cloudflare edge (HTTPS, Universal SSL) → Tunnel (outbound from in-cluster cloudflared)
       → http://istio-ingressgateway.istio-ingress.svc.cluster.local:80
       → public-gateway (matches *.auraenterprise.solutions)
       → per-app VirtualService (matches concrete host)
       → app Service
```

Single shared `Gateway` (`istio-ingress/public-gateway`, listens on `*.auraenterprise.solutions`); each app's `VirtualService` adds its concrete host. Cloudflare Tunnel uses a **single wildcard public hostname rule** (`*.auraenterprise.solutions → http://istio-ingressgateway.istio-ingress.svc.cluster.local:80`) — all routing logic lives in Istio.

### 7.2 Per-env hostnames (✅ implemented for dev + prod)
| Service | Dev | Prod |
| :--- | :--- | :--- |
| BFF | `app-dev.auraenterprise.solutions` | `app.auraenterprise.solutions` |
| Keycloak public | `accounts-dev.auraenterprise.solutions` | `accounts.auraenterprise.solutions` |
| Keycloak admin | `admin-accounts-dev.auraenterprise.solutions` | `admin-accounts.auraenterprise.solutions` |
| report-ms API (mobile/external clients; BFF→report-ms is cluster-DNS internal) | `api-dev.auraenterprise.solutions` | `api.auraenterprise.solutions` |

Staging hostnames are reserved (`app-staging.auraenterprise.solutions` etc.) but `apps-staging` workloads aren't deployed yet.

Admin / dashboard services (`argocd`, `vault`, `grafana`, `kiali`) are intentionally NOT in the Cloudflare tunnel's public hostname list — they live at `*.auraenterprise.solutions` in Istio but require either port-forward or Cloudflare Access SSO to reach. See README's Dashboards table.

### 7.3 Service-to-service inside the cluster
All internal calls use cluster DNS:
- BFF → report-ms: `http://report-ms.apps-<env>.svc:8080`
- report-ms → Keycloak admin API: `http://keycloak.apps-<env>.svc:8080`
- report-ms → SpiceDB: `<name>-spicedb.apps-<env>.svc:50051` — concrete name set by the per-env `SpiceDBCluster` CR's `metadata.name` (e.g. `dev-spicedb`).
- BFF → Keycloak OIDC: **external** (browser redirect) — `https://accounts.<env-host>`. Use the public hostname even though the BFF could reach Keycloak internally — OIDC issuer URL must match between BFF config and tokens.

### 7.4 mTLS
Istio's PERMISSIVE mTLS suffices for now. STRICT mTLS later, with these caveats:
- Postgres pods are opted out of injection (`sidecar.istio.io/inject=false`). DB traffic stays plaintext on the pod network. That's fine on a homelab single-host cluster.
- SpiceDB → datastore: plaintext when in-memory (no datastore); same opt-out as the other CNPG clusters when on postgres.

---

## 8. Sync waves — proposed extension

Existing waves 6 + 7 cover the operators and the Postgres clusters. The app layer goes at waves 8–10:

| Wave | Apps | Why |
| :--- | :--- | :--- |
| **5** | `spicedb-operator` (planned) | Cluster-scoped operator from upstream bundle. Installs CRDs (`SpiceDBCluster`, etc.) before any `apps/spicedb` overlay tries to apply them. |
| **8** | `keycloak-{dev,staging,prod}`, `spicedb-{dev,staging,prod}` (6) | Keycloak needs `postgres-keycloak` (wave 7); SpiceDB needs the operator (wave 5). They have no inter-dependency. The operator handles SpiceDB's own migrate Job. |
| **9** | `report-ms-{dev,staging,prod}` (3) | Needs Keycloak (OIDC) and SpiceDB (gRPC) ready. ArgoCD waits for wave 8 to be Healthy. report-ms's own Liquibase runs on startup. |
| **10** | `aura-report-website-{dev,staging,prod}` (3) | Needs report-ms reachable for proxy targets. Will eventually retry against transient backend failures via Next.js. |

In practice ArgoCD's Healthy gate is permissive — Deployments are "Healthy" once their pods are Ready, which only means the readiness probe passed. The waves are a courtesy for ordered first-deploy; runtime self-healing handles the rest.

---

## 9. Secrets strategy (✅ Vault chosen, partially implemented)

Three layers of secrets:

### 9.1 DB credentials (✅ done, CNPG-managed)
CNPG auto-generates `postgres-<db>-app` Secret per cluster. Consumed via `envFrom: secretRef` or `valueFrom: secretKeyRef`. Nothing imperative.

### 9.2 App secrets (HashiCorp Vault + Vault Secrets Operator)
**Chosen pattern (deviation from original SOPS+age plan):**
- Self-hosted Vault, single-node Raft, Shamir seal (manual unseal — no cloud KMS).
- VSO syncs `VaultStaticSecret` CRDs in each `apps-<env>` namespace → native K8s `Secret`s.
- Apps consume the `Secret`s via standard `envFrom` — Deployment YAML is unchanged from the imperative era.
- Rotation = `vault kv put aura/<env>/<svc> ...`; VSO picks up within 60s; `rolloutRestartTargets` on the `VaultStaticSecret` triggers a Deployment rollout.

**Implementation status:**
- ✅ Vault server, Vault Secrets Operator, default `VaultConnection`, `vault-auth-delegator` SA, Kubernetes auth method, `aura` KV-v2 mount, `aura-app-reader` policy — all landed.
- ✅ `report-ms-secrets` migrated to `VaultStaticSecret` (PoC).
- ❌ `bff-secrets`, `keycloak-admin-secret` still imperative. Same migration pattern — pending.

### 9.3 ECR pull credentials (✅ done, CronJob-rotated)
`core/ecr-rotator/` CronJob runs `aws ecr get-login-password` every 8h and refreshes the `ecr-pull` Secret in each `apps-<env>`. The `aws-credentials` Secret in the `ecr-rotator` namespace is the only imperative bootstrap here. Pending follow-up: move it under Vault too.

### 9.4 Per-secret inventory
| App | Vault path (target) | K8s Secret materialized | Keys | Migrated? |
| :--- | :--- | :--- | :--- | :--- |
| `aura-report-website` | `aura/<env>/bff` | `bff-secrets` | `AUTH_SECRET`, `KEYCLOAK_CLIENT_SECRET` | ❌ pending |
| `report-ms` | `aura/<env>/report-ms` | `report-ms-secrets` | `KEYCLOAK_CLIENT_SECRET`, `KEYCLOAK_CLIENT_UUID`, `SPICEDB_PRESHARED_KEY`, `AWS_ACCESS_KEY`, `AWS_SECRET_ACCESS_KEY`, `MAIL_PASSWORD` | ✅ done |
| `keycloak` | `aura/<env>/keycloak` (planned) | `keycloak-admin-secret` | `KEYCLOAK_ADMIN`, `KEYCLOAK_ADMIN_PASSWORD` | ❌ pending |
| `spicedb` | `aura/<env>/spicedb` (planned) | `<name>-spicedb-secrets` (referenced by `SpiceDBCluster.spec.secretName`) | `preshared_key`, `datastore_uri` (if not in-memory) | ❌ pending — operator not yet installed |
| (cluster-wide) | n/a (CronJob-managed) | `ecr-pull` in each `apps-<env>` | `.dockerconfigjson` | ✅ rotated by CronJob (`core/ecr-rotator/`) |

---

## 10. Approach comparison

Three reasonable ways to land this. Recommendation first.

### A. Kustomize raw manifests for all four apps (recommended)
- Each app is a Deployment + Service + ConfigMap + maybe a Job, hand-written.
- Mirrors what `apps/postgres-*` already does (Kustomize overlays over a base).
- All four apps follow the same shape — easy to write the second one once you've written the first.

**Pros:** Consistent with current pattern. Zero chart opacity. Easy to diff.
**Cons:** ~20 YAML files per app × 4 apps = ~80 files of mostly-Deployment-boilerplate.

### B. Upstream Helm charts where they exist (Keycloak, SpiceDB)
- Keycloak via `bitnami/keycloak` or `codecentric/keycloak`.
- SpiceDB via the official `spicedb-operator` (operator pattern) or a community chart.
- Custom apps (BFF, report-ms) stay Kustomize.

**Pros:** Less YAML for Keycloak/SpiceDB. Upstream-maintained best practices.
**Cons:** Two patterns in the same repo (Helm + Kustomize), debugging Helm-generated manifests is harder, customizations are values-deep. Bitnami's Keycloak chart has known issues with custom themes (requires same image-bake approach we'd do anyway).

### C. Operators everywhere (CNPG-style)
- `keycloak-operator` for Keycloak.
- `spicedb-operator` for SpiceDB.
- Custom apps still Kustomize.

**Pros:** Declarative CRs, operator handles upgrades + clustering.
**Cons:** Two more operators to maintain. Heavier resource footprint on a homelab. Operator behavior is opaque relative to raw Deployment YAML — harder to learn from.

**Picked A for Keycloak/BFF/report-ms, C for SpiceDB.** The raw-manifest pattern proved fine for stateful-ish containers, but SpiceDB's `migrate` + `phasedRollout` lifecycle is exactly what `spicedb-operator` is built to handle. The trade-off (one more operator) is worth not hand-rolling that logic. Keycloak stays on raw manifests until 25.x deployment gets painful.

---

## 11. Phased implementation — outcome

| Phase | Status | Notes |
| :--- | :--- | :--- |
| **1. Postgres adjustments + ECR pull creds** | ✅ Done | Postgres versions pinned per §4. ECR rotator at `core/ecr-rotator/`. |
| **2. Custom Keycloak image (themes baked in)** | ❌ Not yet | Currently using upstream `quay.io/keycloak/keycloak:25.0` + ConfigMap realm-import. Adequate until themes are needed. |
| **3. Land Keycloak** | ✅ Done (dev + prod) | Staging deferred until needed. |
| **3b. Land SpiceDB** | ❌ Not yet (pivoted to operator) | Original CNPG `postgres-spicedb` + hand-written Deployment torn down 2026-06-07. New plan: install `spicedb-operator` at sync-wave 5, per-env `SpiceDBCluster` CRs at wave 8, schema-apply Job in `apps/spicedb/base/`. |
| **4. Land report-ms** | ✅ Done (dev + prod) | Secrets are Vault-managed. |
| **5. Land aura-report-website (BFF)** | ✅ Done (dev + prod) | Secrets still imperative — migration to Vault pending. |
| **6. Public-gateway refactor + public ingress** | ✅ Done | Shared `public-gateway` on `*.auraenterprise.solutions` + Cloudflare Tunnel (in-cluster `cloudflared`). Different from original plan (no dnsmasq + `*.lab.lan`). |
| **6b. Secrets management** | 🟡 Partial | Vault + VSO + Kubernetes auth method are in. report-ms migrated as PoC. bff + keycloak-admin migrations pending. |
| **7. Refactor 12 Applications → 1 ApplicationSet** | ❌ Deferred | Per the Backlog. Defer until hand-written Applications are stable. |

---

## 12. Open questions / TODOs

1. **SpiceDB image tag + operator install method** — pending. Decide: pin operator bundle URL (`bundle.yaml` from a specific release) vs the upstream Helm chart in `authzed/spicedb-operator/charts/spicedb-operator`. Image tag for the SpiceDB pods themselves is set via `SpiceDBCluster.spec.version` (channel-driven, but pinning is supported).
2. **AWS S3 bucket per env** — overlays currently use `aura-dev` / `aura-prod` patches per `AWS_S3_BUCKET_NAME`. Confirm the bucket names match what's provisioned in AWS.
3. **Mobile client API endpoint** — `api-dev.auraenterprise.solutions` and `api.auraenterprise.solutions` VirtualServices are in place. Confirm `report-mobile` actually hits these vs. going via the BFF.
4. **Email enabled** — `MAIL_PASSWORD` is currently a Gmail app password in `report-ms-secrets`. Decide whether SMTP via personal Gmail is right long-term or move to SES.
5. **CNPG backups** — still on the backlog. Cannot run real production data without this.
6. **Keycloak custom theme image** — phase 2 work is pending. Currently using the upstream image; themes will come via custom-built image to ECR when the user is ready.
7. **Vault secrets-migration completion** — `bff-secrets`, `keycloak-admin-secret`, `aws-credentials` (for ECR rotator) all still imperative. Same `VaultStaticSecret` pattern as `report-ms-secrets` — pending.

---

## 13. Summary — what landed

- **3 of 4 app trees** built: `keycloak`, `report-ms`, `aura-report-website` (dev + prod each). SpiceDB still pending — operator pivot (2026-06-07) cleared the previous CNPG-backed scaffold; `core/spicedb-operator/` + `apps/spicedb/` are the next thing to scaffold.
- **Public ingress**: Cloudflare Tunnel (in-cluster `cloudflared`) + shared Istio `public-gateway` on `*.auraenterprise.solutions`. Single wildcard rule in Cloudflare; all routing is Istio VirtualServices.
- **ECR credential rotator**: `core/ecr-rotator/` CronJob (8h schedule) refreshes the `ecr-pull` Secret across `apps-*`.
- **Secrets**: HashiCorp Vault (self-hosted, single-node Raft, Shamir seal) + Vault Secrets Operator. `report-ms` migrated as PoC. `bff-secrets` + `keycloak-admin-secret` migrations pending — same per-app pattern.
- **No custom Keycloak image yet** — running upstream `quay.io/keycloak/keycloak:25.0` with realm-import via ConfigMap. Themes will come later via ECR-built image.

The original plan held up well except for two deliberate deviations: **Cloudflare Tunnel + auraenterprise.solutions domain** (instead of `dnsmasq` + `*.lab.lan`) and **Vault** (instead of SOPS+age). Both were re-decisions made during implementation based on what we learned along the way; the live state of the repo is the source of truth.
