# Aura Stack — Kubernetes Deployment Architecture

**Date:** 2026-06-03
**Status:** Draft for review, not yet implemented
**Scope:** Designing how `aura-report-website`, `report-ms`, Keycloak, and SpiceDB land on the gitops-managed cluster across `dev` / `staging` / `prod`.

This is a companion to [`GITOPS_STRATEGY.md`](./GITOPS_STRATEGY.md) (the *how we deploy*) and [`README.md`](./README.md) (the *what's currently deployed*). It describes the *what's about to be deployed*.

---

## 1. Overview

```
   ┌──────────────────┐  HTTPS  ┌─────────────────────────┐
   │   Browser/User   │ ◀─────▶ │ Istio Gateway (*.lab.lan│   external hostnames →
   └──────────────────┘         │   or *.auratest.dev)     │   *.lab.lan in homelab,
                                └─────────┬───────────────┘   auratest.dev in prod
                                          │
              ┌───────────────────────────┼───────────────────────────────┐
              ▼                           ▼                               ▼
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
       ┌─────────────────────────┬─────────────────────────────────────┬─────────┘
       ▼                         ▼                                     ▼
┌──────────────────┐    ┌──────────────────┐                  ┌──────────────────┐
│ postgres-aura    │    │ postgres-keycloak│                  │ postgres-spicedb │
│ CNPG, PG 18.0    │    │ CNPG, PG 16.4    │                  │ CNPG, PG 18.0    │
│ db: aura         │    │ db: keycloak     │                  │ db: spicedb      │
│ public + audit   │    │ self-init        │                  │ track_commit_ts=on│
└──────────────────┘    └──────────────────┘                  └──────────────────┘
```

Per environment (`apps-dev`, `apps-staging`, `apps-prod`):
- **3 Postgres clusters** — already scaffolded (`postgres-aura`, `postgres-keycloak`, `postgres-spicedb`).
- **3 stateful services** — Keycloak, SpiceDB, report-ms.
- **1 edge service** — aura-report-website.
- **1 Istio Gateway + multiple VirtualServices** — to expose BFF, Keycloak public/admin, and report-ms API.

---

## 2. What's already in place vs. what's new

| Layer | Status | Notes |
| :--- | :--- | :--- |
| Cluster, ArgoCD, Istio, observability | ✅ Done | Per `README.md`. |
| CNPG operator | ✅ Done | At sync-wave 6. |
| Postgres clusters (aura / keycloak / spicedb) × 3 envs | ✅ Done (skeleton) | At sync-wave 7. Needs version pinning + `track_commit_timestamp` patch (§4). |
| Istio Gateway routing (only `argocd.lab.lan` so far) | ⏳ Partial | Will need to broaden to `*.lab.lan` or add per-host hosts. |
| ECR image pull credentials | ❌ Not yet | Required by all 4 custom images. §6. |
| Application Deployments (BFF, report-ms, Keycloak, SpiceDB) | ❌ Not yet | The bulk of this doc. |
| Non-DB Secrets (NextAuth, Keycloak admin pw, SpiceDB preshared key, AWS S3 creds, OIDC client secrets, email) | ❌ Not yet | Imperative until SOPS+age lands. |
| Liquibase migrations (report-ms) | ✅ Built into app | Spring Boot auto-runs on startup; no Job needed. |
| SpiceDB migrate | ❌ Not yet | Will be an init container on the SpiceDB Deployment. |
| Keycloak realm import + custom themes | ❌ Not yet | Themes baked into a custom image (§6); realm-import.json mounted as ConfigMap. |

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

### 3.4 SpiceDB
- **Image:** `authzed/spicedb:latest` — **pin to a specific tag** before deploying; "latest" is unsafe in GitOps.
- **Listens on:** `:50051` (gRPC), `:8443` (HTTP gateway, optional).
- **DB:** `postgres-spicedb` (PG 18.0, `track_commit_timestamp=on`).
- **Required env vars / args:**
  - `SPICEDB_GRPC_PRESHARED_KEY` — Secret
  - `--datastore-engine=postgres`
  - `--datastore-conn-uri=postgres://...` — from CNPG Secret
- **Init container:** `authzed/spicedb migrate head` runs once before the server boots. Same image, same env (DB conn).
- **Schema bootstrap:** 14 `.zed` files in `report-ms/application/src/main/resources/spicedb/` — applied by a sidecar `Job` after migration completes, using `zed schema write index.zed`. Re-apply on every deploy (idempotent).
- **Reaches:** postgres-spicedb only.

---

## 4. Postgres adjustments needed (modify the existing CNPG `Cluster` CRs)

The current `apps/postgres-*/base/cluster.yaml` files have generic settings. Two of three clusters move to PG 18.0; postgres-keycloak stays on 16.4 to remain inside Keycloak 25's official support matrix.

| Cluster | Edit | Why |
| :--- | :--- | :--- |
| `postgres-aura` | Pin `spec.imageName` to `ghcr.io/cloudnative-pg/postgresql:18.0`. | report-ms's Liquibase changelog uses standard DDL; PG 18 is a strict superset. Verify on first dev sync. |
| `postgres-keycloak` | Pin `spec.imageName` to `ghcr.io/cloudnative-pg/postgresql:16.4`. | Keycloak 25 officially supports PG 13–16; staying inside that matrix avoids vendor-untested behavior. |
| `postgres-spicedb` | (a) `spec.imageName: ghcr.io/cloudnative-pg/postgresql:18.0`. (b) Add `spec.postgresql.parameters.track_commit_timestamp: "on"`. | SpiceDB requires PG 18 + CDC via `track_commit_timestamp` for the postgres datastore engine. |

These are diffable, minimal edits. No restructuring of the postgres tree.

---

## 5. Repo layout — proposed additions

Follow the established pattern: each app gets a DRY tree under `apps/<svc>/{base,overlays/<env>}/`, and 3 Applications under `bootstrap/apps/<svc>-<env>.yaml`, all using `spec.sourceHydrator`.

```
apps/
├── postgres-{aura,keycloak,spicedb}/   # existing
├── keycloak/
│   ├── base/                           # Deployment, Service, ConfigMap (realm-import), ServiceAccount
│   └── overlays/{dev,staging,prod}/    # replicas, resource limits, KC_HOSTNAME, image tag
├── spicedb/
│   ├── base/                           # Deployment (with migrate initContainer), Service, schema-apply Job
│   └── overlays/{dev,staging,prod}/    # replicas, resource limits, image tag, preshared-key Secret ref
├── report-ms/
│   ├── base/                           # Deployment, Service, env/Secret refs, actuator probes
│   └── overlays/{dev,staging,prod}/    # replicas, resource limits, image tag, DB url, AWS_S3_BUCKET_NAME
└── aura-report-website/
    ├── base/                           # Deployment, Service, env/Secret refs
    └── overlays/{dev,staging,prod}/    # replicas, resource limits, image tag, NEXTAUTH_URL, etc.

bootstrap/apps/
├── postgres-*-<env>.yaml               # existing × 9
├── keycloak-<env>.yaml                 # × 3
├── spicedb-<env>.yaml                  # × 3
├── report-ms-<env>.yaml                # × 3
├── aura-report-website-<env>.yaml      # × 3
└── (eventually one ApplicationSet replaces all 12 app entries; see §11)

core/
├── (existing) ...
└── routing/
    └── public-gateway.yaml             # refactor argocd-routing's Gateway → shared *.lab.lan
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
The 14 `.zed` files live with report-ms. Two ways to get them into SpiceDB:

| Option | How |
| :--- | :--- |
| **A. Job in `apps/spicedb/base/`** that mounts a ConfigMap of the .zed files and runs `zed schema write`. ConfigMap generated via Kustomize `configMapGenerator` from a local copy of the .zed files. | Schema lives in gitops repo; clean to diff. |
| **B. report-ms applies schema on startup** | No extra Job; coupling — schema can't deploy without report-ms |

**Recommendation:** A. Copy the `.zed` files into `apps/spicedb/base/schema/` and `configMapGenerator` them. Idempotent Job re-applies every sync.

---

## 7. Networking — Istio Gateway + VirtualServices

### 7.1 Refactor the gateway first
Currently `core/argocd/routing/gateway.yaml` is host-specific (`argocd.lab.lan`). Refactor to a shared `public-gateway` with `hosts: ["*.lab.lan"]` (and `*.auratest.dev` later) so we don't grow Gateways linearly with hosts.

Move it to `core/routing/public-gateway.yaml`, and have each app's `VirtualService` reference `routing/public-gateway` cross-namespace.

### 7.2 Per-env hostnames
| Service | Homelab host | Prod host |
| :--- | :--- | :--- |
| BFF | `app-dev.lab.lan` / `app-staging.lab.lan` / `app.lab.lan` | `auratest.dev` |
| Keycloak public | `accounts-dev.lab.lan` etc. | `accounts.auratest.dev` |
| Keycloak admin | `admin-accounts-dev.lab.lan` | `admin-accounts.auratest.dev` |
| report-ms API (public; the BFF calls it directly via cluster DNS internally) | `api-dev.lab.lan` (mobile client only) | `api.auratest.dev` |

### 7.3 Service-to-service inside the cluster
All internal calls use cluster DNS:
- BFF → report-ms: `http://report-ms.apps-<env>.svc:8080`
- report-ms → Keycloak admin API: `http://keycloak.apps-<env>.svc:8080`
- report-ms → SpiceDB: `spicedb.apps-<env>.svc:50051`
- BFF → Keycloak OIDC: **external** (browser redirect) — `https://accounts.<env-host>`. Use the public hostname even though the BFF could reach Keycloak internally — OIDC issuer URL must match between BFF config and tokens.

### 7.4 mTLS
Istio's PERMISSIVE mTLS suffices for now. STRICT mTLS later, with these caveats:
- Postgres pods are opted out of injection (`sidecar.istio.io/inject=false`). DB traffic stays plaintext on the pod network. That's fine on a homelab single-host cluster.
- SpiceDB → Postgres also plaintext.

---

## 8. Sync waves — proposed extension

Existing waves 6 + 7 cover the operator and the Postgres clusters. The app layer goes at waves 8–10:

| Wave | Apps | Why |
| :--- | :--- | :--- |
| **8** | `keycloak-{dev,staging,prod}`, `spicedb-{dev,staging,prod}` (6) | Each needs its Postgres ready (wave 7). They have no inter-dependency. SpiceDB's `migrate` runs as an init container, so it's self-gating. |
| **9** | `report-ms-{dev,staging,prod}` (3) | Needs Keycloak (OIDC) and SpiceDB (gRPC) ready. ArgoCD waits for wave 8 to be Healthy. report-ms's own Liquibase runs on startup. |
| **10** | `aura-report-website-{dev,staging,prod}` (3) | Needs report-ms reachable for proxy targets. Will eventually retry against transient backend failures via Next.js. |

In practice ArgoCD's Healthy gate is permissive — Deployments are "Healthy" once their pods are Ready, which only means the readiness probe passed. The waves are a courtesy for ordered first-deploy; runtime self-healing handles the rest.

---

## 9. Secrets strategy

We have **two windows** to think about:

### 9.1 Today (no SOPS yet)
- **DB credentials:** CNPG auto-generates `postgres-<db>-app` Secret per cluster. Consumed via `envFrom: secretRef`. **No action needed.**
- **Non-DB secrets** (NextAuth secret, Keycloak admin password, Keycloak client secrets, SpiceDB preshared key, AWS access key, email SMTP password): Created imperatively per env with `kubectl create secret`. Document the exact `kubectl` commands in this doc + the README's bootstrap section. Same pattern as the ArgoCD repo-write PAT today.

### 9.2 After SOPS+age lands (Backlog item)
- Move all the imperative Secrets to SOPS-encrypted YAML under `apps/<svc>/overlays/<env>/secrets.enc.yaml`.
- DB credentials stay auto-generated by CNPG; nothing changes there.
- The bootstrap "imperative" step disappears for SOPS-managed secrets.

### 9.3 Per-secret inventory (what each app needs)
| App | Imperative Secret name | Keys |
| :--- | :--- | :--- |
| `aura-report-website` | `bff-secrets` | `AUTH_SECRET`, `KEYCLOAK_CLIENT_SECRET` |
| `report-ms` | `report-ms-secrets` | `KEYCLOAK_CLIENT_SECRET`, `SPICEDB_PRESHARED_KEY`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, (`EMAIL_*` if enabled) |
| `keycloak` | `keycloak-bootstrap` | `KEYCLOAK_ADMIN`, `KEYCLOAK_ADMIN_PASSWORD` |
| `spicedb` | `spicedb-secrets` | `SPICEDB_GRPC_PRESHARED_KEY` |
| (cluster-wide) | `ecr-pull` (in each `apps-<env>` ns) | `.dockerconfigjson` (rotated by CronJob) |

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

**Pick A.** Land raw manifests for all four apps, follow the postgres pattern. If Keycloak's deployment turns out to be painful (it shouldn't — 25.x is a single statefulish container), revisit B for it alone. Operators are a much later concern.

---

## 11. Phased implementation plan

Designed so each phase deploys cleanly and reverts cleanly. Roughly 4 hours of work end-to-end on a homelab.

### Phase 1 — Postgres adjustments + ECR pull creds (small commit)
- Edit `apps/postgres-aura/base/cluster.yaml`: `spec.imageName: ghcr.io/cloudnative-pg/postgresql:16.4`.
- Edit `apps/postgres-keycloak/base/cluster.yaml`: same.
- Edit `apps/postgres-spicedb/base/cluster.yaml`: `spec.imageName: ghcr.io/cloudnative-pg/postgresql:18.0`, add `track_commit_timestamp: "on"` under `postgres.parameters`.
- Add `core/ecr-cred-rotator/` — CronJob that runs `aws ecr get-login-password` every 8h and writes/refreshes a `Secret/ecr-pull` in each `apps-<env>` namespace. Documented imperative AWS credentials Secret as bootstrap.
- Commit.

### Phase 2 — Build & push the custom Keycloak image
- Add Dockerfile to `auth-server` repo: `FROM quay.io/keycloak/keycloak:25.0`, `COPY themes/ /opt/keycloak/themes/`.
- Tag as `aura:keycloak-<version>`, push to ECR.
- This is a one-time piece of work, not part of the gitops repo.

### Phase 3 — Land Keycloak + SpiceDB (sync-wave 8)
- Scaffold `apps/keycloak/{base,overlays/<env>}` + `bootstrap/apps/keycloak-<env>.yaml` × 3.
- Scaffold `apps/spicedb/{base,overlays/<env>}` + `bootstrap/apps/spicedb-<env>.yaml` × 3.
- Copy `report-ms/application/src/main/resources/spicedb/*.zed` into `apps/spicedb/base/schema/`.
- Create the imperative Secrets: `keycloak-bootstrap`, `spicedb-secrets` per env.
- Mount realm-import via ConfigMap (`kubectl create configmap keycloak-realm --from-file=realm-import.json=...`).
- Commit + push + watch Apps go Healthy in dev first.

### Phase 4 — Land report-ms (sync-wave 9)
- Scaffold `apps/report-ms/{base,overlays/<env>}` + 3 Applications.
- Create `report-ms-secrets` per env.
- Wire env vars from the CNPG-auto-generated `postgres-aura-app` Secret.
- Verify against Keycloak (token validation works) and SpiceDB (a permission check works).

### Phase 5 — Land aura-report-website (sync-wave 10)
- Scaffold `apps/aura-report-website/{base,overlays/<env>}` + 3 Applications.
- Create `bff-secrets` per env.
- Open the browser, run through the OIDC redirect flow.

### Phase 6 — Public-gateway refactor + Istio routing for all apps
- Move `argocd-routing`'s Gateway → `core/routing/public-gateway.yaml` (hosts `*.lab.lan`).
- Update `argocd-routing`'s VirtualService to reference it.
- Add VirtualServices for BFF, Keycloak, report-ms in each `apps/<svc>/overlays/<env>/`.
- DNS pointing on Mac mini: `*.lab.lan` → host IP via dnsmasq.

### Phase 7 — Refactor 12 app Applications → 1 ApplicationSet
Only after the hand-written versions work end-to-end in dev. ApplicationSet matrix generator over `(service, environment)` collapses the 12 Application YAMLs into a single template. Defer per the existing Backlog item.

---

## 12. Open questions / TODOs to land before Phase 1

1. **SpiceDB image tag.** Pin a specific version, not `latest`. Pick the latest stable from `authzed/spicedb` releases at write time.
2. **AWS S3 bucket per env.** Are there 3 distinct buckets (`aura-dev`, `aura-staging`, `aura-prod`) or one shared? Doc assumes 3 distinct; confirm in the Phase 4 overlay.
3. **Mobile client API endpoint.** Is `api.auratest.dev` actually consumed by `report-mobile`, or is everything routed via the BFF? Doc shows `api.*.lab.lan` exposed; remove if not needed.
4. **Email enabled in homelab?** `AURA_EMAIL_ENABLED` is configurable. Default `false` for dev/staging, decide later for prod.
5. **api-gateway service** referenced in docker-compose but never defined — is it just an alias for the nginx reverse-proxy + report-ms? Doc assumes no separate api-gateway in K8s (Istio gateway replaces it).
6. **CNPG backups.** Still on the backlog. Phase 1 doesn't unblock production use — backups must precede any prod data.

---

## 13. Summary

- **4 new app trees** (`keycloak`, `spicedb`, `report-ms`, `aura-report-website`) following the established `apps/<svc>/{base,overlays/<env>}` Kustomize pattern.
- **12 new ArgoCD Applications** (3 envs × 4 services) using `spec.sourceHydrator`.
- **2 small CNPG `Cluster` patches** for Postgres versioning + `track_commit_timestamp`.
- **1 custom Keycloak image** (themes baked in).
- **1 ECR credential CronJob** managing pull secrets across the 3 `apps-*` namespaces.
- **1 Gateway refactor** to handle the growing list of hostnames.
- **Per-env imperative non-DB Secrets** until SOPS+age lands.

No new operators, no Helm charts. Consistent pattern with the existing platform.
