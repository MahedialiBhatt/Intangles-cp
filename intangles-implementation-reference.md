# Intangles On-Prem — Implementation Reference

> This file exists to close the gaps between the master plan and a working implementation.
> It documents the exact tech stack, key decisions already made, what is already built,
> and forward-looking design choices (wire protocols, schemas, sequencing rules).
> Read this alongside `intangles-onprem-master-plan.md` before implementing any phase.

---

## Tech Stack

### Operator (`intangles-operator/`)

| Concern | Choice | Version |
|---|---|---|
| Language | Go | 1.22 |
| Operator framework | controller-runtime (Kubebuilder) | v0.18.2 |
| K8s client | client-go | v0.30.0 |
| K8s API types | k8s.io/api + apimachinery | v0.30.0 |
| WebSocket client | gorilla/websocket | v1.5.1 |
| Helm management | `helm upgrade --install` via `os/exec` (shell out) | system Helm |
| Logging | go-logr / zap | v1.4.1 / v1.26.0 |
| Metrics | prometheus/client_golang (built into controller-runtime) | v1.16.0 |
| Build | `go build` → single static binary | — |
| Container | Distroless or scratch base image | — |
| Local dev cluster | Kind | via `make local` |

**No ORM, no migration framework, no test framework beyond `go test`.** The operator has no test suite currently — `go vet` is the only linter.

### CP Server (`intangles-cp-platform/server/`)

| Concern | Choice | Version |
|---|---|---|
| Language | JavaScript (CommonJS, Node.js) | Node 20+ |
| HTTP framework | Fastify | v4.26.0 |
| WebSocket server | ws (standalone, not Fastify plugin) | v8.18.0 |
| Database driver | mongodb (official Node.js driver) | v5.9.2 |
| Database | MongoDB | any 6.x+ |
| Auth / JWT | jsonwebtoken | v9.0.2 |
| Static file serving | @fastify/static (optional — serves dashboard dist) | v7.0.4 |
| CORS | @fastify/cors | v8.5.0 |
| Process management | `node src/server.js` directly or PM2 (not committed) | — |
| No ORM | Raw MongoDB driver with `collection.find/insertOne/updateOne` | — |
| No test framework | None currently | — |

### CP Dashboard (`intangles-cp-platform/dashboard/`)

| Concern | Choice | Version |
|---|---|---|
| Language | JavaScript (plain JSX, no TypeScript) | — |
| Framework | React | 18.3.1 |
| Bundler | Vite | 5.4.0 |
| Routing | None — custom `useState` page state in App.jsx | — |
| State management | None — local component `useState` only | — |
| Styling | Inline styles (CSS-in-JS objects) + one global `index.css` | — |
| Component library | None — all UI primitives written by hand | — |
| API calls | Native `fetch` via a thin `src/api.js` wrapper | — |
| Persistence | `localStorage` (admin token, active page, active customer, settings) | — |
| No test framework | None currently | — |

**Build output:** `npm run build` → `dashboard/dist/`. The CP server serves this via `@fastify/static` if `dist/dashboard/` exists alongside the server binary. In development, Vite dev server runs on a separate port with proxy config.

### Infrastructure / Tooling

| Concern | Choice |
|---|---|
| Local K8s | Kind (Kubernetes-in-Docker) |
| Helm | System Helm CLI (operator shells out) |
| Container registry | GHCR (`ghcr.io/intangles/`) |
| Multi-arch builds | `docker buildx` (via `scripts/release.sh`) |
| CRD generation | `controller-gen` via `make generate` |
| Deployment automation | `make local` / `make redeploy` / `make e2e` / `make release` |

---

## Key Decisions Already Made

These are the choices locked into the existing codebase. New code must respect them — do not reintroduce alternatives.

### Architecture

| Decision | What was chosen | What was rejected and why |
|---|---|---|
| CP tech stack | Node.js + Fastify + MongoDB | Go API or PostgreSQL — team velocity, existing stack familiarity |
| HTTP framework | Fastify (not Express) | Express was considered but Fastify's schema-based validation and faster JSON serialization won |
| WS library | `ws` standalone (not Socket.io, not Fastify WS plugin) | Socket.io adds reconnect/room abstractions we don't need; `ws` is lower-level and predictable |
| Operator framework | controller-runtime (Kubebuilder) | Writing a raw K8s controller — Kubebuilder gives CRD scaffolding, reconcile loop, status subresource for free |
| Helm via shell-out | `os/exec helm upgrade --install` | Go Helm SDK — the SDK is complex and opinionated about release storage; shell-out is simpler and easier to debug |
| Single CRD | One `IntanglesPlatform` CRD per customer cluster | Multiple CRDs per concern — one CRD per customer is simpler; the operator manages everything through one reconcile loop |
| CP as source of truth | All desired state (components, packs, config) originates from CP | Letting operators write to CRD spec directly — CP must be the authority; direct spec writes bypass business logic |
| Pack → component resolution at config-request time | CP resolves component union on `GET /config/:agentId` | Storing resolved component list on customer doc — config-request resolution means pack changes are immediately reflected without data migrations |
| Smart-stop rule | Component removed only when no remaining pack references it | Always remove all pack components on pack removal — shared components (mongodb, rabbitmq) would be deleted when any pack is removed |
| APP_TYPE discriminator | Single image, APP_TYPE env var selects process | Separate images per service — one image is simpler to build, push, and version; APP_TYPE is already how the services work |

### Auth

| Decision | What was chosen | Why |
|---|---|---|
| Admin auth | Static token from env var (`CONTROL_PLANE_ADMIN_TOKEN`) | No user DB, no login endpoint — this is an internal ops tool with one admin identity |
| Agent auth | JWT signed with `CONTROL_PLANE_JWT_SECRET` | Stateless; operator can reconnect without a DB lookup on every request |
| Token versioning | `tokenVersion` field on customer doc, `tv` claim in JWT | Enables instant revocation via `regenerate-token` without a token blacklist DB |
| Agent ID format | `{company-slug}-{6-char-hex}` e.g. `acme-corp-a1b2c3` | Human-readable + collision-resistant; hex suffix prevents duplicate slugs for same company name |

### CP Server Patterns

| Decision | What was chosen | Why |
|---|---|---|
| DB access pattern | `req.db` injected via Fastify `onRequest` hook | No connection pool management per-route; single shared connection via `getDb()` singleton |
| WS hub | In-memory `Map<agentId, WebSocket>` in `ws-hub.js` | Redis pub/sub or external broker — overkill for a single CP instance; in-memory is simpler and sufficient |
| Command queue | MongoDB `command_queue` collection | In-memory queue — DB queue survives CP restarts; queued commands delivered when agent reconnects |
| Status storage | Separate `operator_status` collection (not on customer doc) | Putting heartbeat status on customer doc causes write contention on every heartbeat; separate collection is isolated |
| Heartbeat join | Batch `$in` query at read time in `GET /api/customers` | Pre-joining at write time — read-time join keeps data separation clean; batch query is fast enough |
| Component data | Stored in `components` MongoDB collection, editable via API | Hardcoded component definitions — catalog must be editable at runtime without a deploy |
| Pack data | Stored in `packs` MongoDB collection, seeded via `seed.js` | Hardcoded pack definitions — same reason; packs evolve as the product grows |

### Dashboard Patterns

| Decision | What was chosen | Why |
|---|---|---|
| No routing library | `useState` page state in `App.jsx` + `localStorage` restore | React Router adds URL coupling — this is an internal ops tool where shareable URLs are not a priority (yet — FE Phase 1 adds routing) |
| No state library | Local `useState` in each page | Redux/Zustand adds abstraction overhead; every page manages its own data (to be consolidated in FE Phase 1) |
| Inline styles everywhere | CSS-in-JS style objects per component | CSS Modules or Tailwind — the team wanted a zero-build-config setup; inline styles colocate appearance with structure |
| Single `index.css` | Global resets + `.card`, `.badge`, `.modal` classes | CSS-in-JS only — a few truly global structural classes are better as real CSS; everything else is inline |
| Fingerprint-based poll dedup | `JSON.stringify` of status fields compared on each poll | React re-render on every poll — prevents unnecessary re-renders when heartbeat data hasn't changed |
| Adaptive polling | 5s when infra is pending-install/upgrade, else `customerSettings.customerPollMs` | Fixed interval — shorter interval during active Helm operations; longer interval at rest to reduce load |
| Lazy-load panels | Credentials and command queue fetched only on explicit click | Auto-loading — credentials are sensitive; command queue is rarely needed |
| Per-customer + global poll settings | `localStorage` keyed by customer ID, falls back to global | Single global setting — some customers need more aggressive polling during operations |
| `window.confirm()` for destructive actions | Browser native confirm dialog | Custom confirmation modal — native confirm blocks JS and is synchronous; sufficient for an internal tool (to be replaced in FE Phase 4) |

---

## What Is Already Built (Phase 1 + 2)

Use this as the baseline. Do not re-implement anything listed here — only extend it.

### Operator (Go)

**Files:**
- `cmd/main.go` — entrypoint; sets up controller-runtime manager, registers reconciler, registers WS command handlers
- `api/v1alpha1/types.go` — `IntanglesPlatform` CRD types: `IntanglesPlatformSpec`, `IntanglesPlatformStatus`, `ServiceSpec`, `InfraSpec`, `InfraComponent`, `ImageSpec`, `ComponentOverride`, `ServiceStatus`, `InfraStatus`, `HelmEvent`
- `api/v1alpha1/groupversion_info.go` — CRD group/version registration
- `api/v1alpha1/zz_generated.deepcopy.go` — generated DeepCopy methods (do not edit)
- `internal/reconciler/reconciler.go` — main reconcile loop; phase progression WaitingForConfig → Provisioning → Running
- `internal/reconciler/features.go` — `desiredServices()` and service Deployment reconciliation
- `internal/infra/infra.go` — Helm install/upgrade/uninstall; credential generation; ConfigMap build; template resolution
- `internal/controlplane/client.go` — HTTP client: `GetConfig()`, `ValidateLicense()`, `StoreCredentials()`
- `internal/controlplane/websocket.go` — persistent WS client; sends heartbeat every 30s; linear reconnect (to be replaced with exp+jitter in Phase 4)
- `internal/controlplane/auth.go` — JWT bearer token injection
- `hack/mock-controlplane/main.go` — mock CP for e2e tests

**Capabilities already working:**
- Full Helm infra install/upgrade/uninstall cycle
- Service Deployment creation and reconciliation
- ConfigMap + Secret credential generation and templating
- db-init Job lifecycle
- All 13 WS commands handled (kill, reactivate, config_change, agent_update, restart_service, scale_service, delete_service, retry_infra, restart_infra, upgrade_infra, delete_infra, reauth + one alias set)
- Operator self-update (patches own Deployment image)
- Kill switch (`spec.paused` → scale all to 0)
- License stub (always returns true)
- `make local` / `make redeploy` / `make e2e` / `make release`

### CP Server (Node.js)

**Files:**
- `server/src/server.js` — Fastify setup; plugin registration; static dashboard serving; `/healthz`
- `server/src/db.js` — MongoDB connect + `getDb()` singleton
- `server/src/auth.js` — `issueAgentToken()`, `verifyAgentToken()`, `generateAgentId()`, `requireAdmin`, `requireAgent` (with token version check)
- `server/src/ws-hub.js` — WebSocket server; agent connection map; heartbeat handler; keepalive pings; `sendCommand()`, `broadcastCommand()`, `queueCommand()`, `flushQueue()`, `getConnectedAgents()`
- `server/src/routes/config.js` — `GET /config/:agentId`; pack resolution (smart-stop); config response assembly
- `server/src/routes/customers.js` — full customer CRUD + all command routes
- `server/src/routes/components.js` — component catalog CRUD (upsert by name)
- `server/src/routes/packs.js` — pack CRUD + customer pack assignment/removal
- `server/src/routes/releases.js` — release CRUD + deploy-all
- `server/src/routes/license.js` — license stub (`GET /license/:agentId` always returns `{ valid: true }`)
- `server/src/data/components.js` — seed data: component definitions
- `server/src/data/packs.js` — seed data: pack definitions
- `server/scripts/seed.js` — seeds components + packs into MongoDB (idempotent; `--force` flag re-seeds)

**Routes already implemented** (exact paths — these already exist, do not recreate):
```
GET  /api/customers
GET  /api/customers/:id
POST /api/customers
PUT  /api/customers/:id/overrides
POST /api/customers/:id/components
DELETE /api/customers/:id/components/:name
POST /api/customers/:id/components/:name/restore
PUT  /api/customers/:id/infra-repos
PUT  /api/customers/:id/images
POST /api/customers/:id/kill
POST /api/customers/:id/reactivate
POST /api/customers/:id/command
POST /api/customers/:id/operator-update
POST /api/customers/:id/regenerate-token
POST /api/customers/:id/infra/:name/retry
POST /api/customers/:id/infra/:name/restart
POST /api/customers/:id/infra/:name/upgrade
GET  /api/customers/:id/command-queue
DELETE /api/customers/:id/command-queue
GET  /api/customers/:id/credentials
GET  /api/components
GET  /api/components/:name
PUT  /api/components/:name
DELETE /api/components/:name
GET  /api/packs
GET  /api/packs/:slug
PUT  /api/packs/:slug
DELETE /api/packs/:slug
POST /api/customers/:id/pack          (assign pack)
DELETE /api/customers/:id/pack/:slug  (remove pack)
GET  /api/releases
POST /api/releases
POST /api/releases/:version/deploy
GET  /config/:agentId                 (agent-authenticated)
POST /credentials/:agentId            (agent-authenticated)
GET  /license/:agentId                (agent-authenticated)
GET  /healthz
```

### CP Dashboard (React)

**Files:**
- `dashboard/src/App.jsx` — shell layout (sidebar nav + main content area); page state via `useState`; localStorage restore of last-viewed customer on mount; Login gate
- `dashboard/src/api.js` — thin `fetch` wrapper with admin token injection; all API calls as named functions under `api.customers`, `api.releases`, `api.components`, `api.packs`
- `dashboard/src/index.css` — global CSS: resets, `button`, `input`/`select`/`textarea`, `table`/`th`/`td`, `.badge` (+ `.badge-green/red/yellow/gray`), `.card`, `.modal-overlay`, `.modal`
- `dashboard/src/pages/Login.jsx` — full-viewport token input; stores token to `localStorage("cp_admin_token")`
- `dashboard/src/pages/CustomerList.jsx` — customer table with polling; create customer modal (shows one-time AGENT_TOKEN after creation); kill/reactivate per row; connection status with reconnecting detection (2min window)
- `dashboard/src/pages/CustomerDetail.jsx` — full customer management page; 5 tabs; background polling with fingerprint dedup; adaptive poll interval; all service + infra actions
- `dashboard/src/pages/Catalog.jsx` — component catalog; services + infra tabs; add/edit modal; apply-to-customer flow
- `dashboard/src/pages/Releases.jsx` — release list; publish modal; deploy-to-all
- `dashboard/src/pages/Settings.jsx` — global poll settings + `SettingsForm` component reused in CustomerDetail Settings tab; `loadSettings(customerId?)`, `saveSettings(settings, customerId?)`, `hasCustomSettings(customerId)`, `resetCustomSettings(customerId)` exported utilities

**Inline components already written in `CustomerDetail.jsx`** (not extracted to separate files yet — FE Phase 2 will extract them):
- `Flash` — dismissible success/error banner; auto-dismiss via `setTimeout`
- `StatCard` — label + value + optional subtitle card
- `ServiceRow` — expandable accordion row for deployment components; 4 action buttons; env blocks
- `EnvBlock` — key=value display block with dark background
- `ActionBtn` — small button with color + optional wide variant
- `InfoPair` — label + value display (uppercase label, optional monospace value)
- `ServiceOverridesModal` — edit memory + env overrides per service
- `ScaleModal` — replica count input
- `UpgradeInfraModal` — chart version input for Helm upgrades
- `PackInfoModal` — shows service + infra components in a pack
- `PackSection` — full pack management: assign/remove packs, enable/disable per-component

**Design system (implemented in `index.css` + inline styles):**

| Token | Value | Used for |
|---|---|---|
| Page bg | `#0f1117` | body |
| Card bg | `#161b27` | `.card` |
| Surface elevated | `#1e2230` | inputs, button bg |
| Surface deep | `#0d1018` | sidebar, read-only blocks |
| Surface deepest | `#070a10` | env var blocks, code blocks |
| Border default | `#1e2230` | cards |
| Border subtle | `#1a1f2e` | table rows |
| Border medium | `#2d3748` | inputs, modals |
| Text primary | `#e2e8f0` | main text |
| Text secondary | `#718096` | labels, meta |
| Text muted | `#4a5568` | timestamps, inactive |
| Accent blue | `#4f6ef7` | primary CTAs, active tab, focus ring |
| Green | `#48bb78` | success, connected, ready |
| Green bg | `#0f2a1a` / `#1a3a2a` | success backgrounds |
| Yellow | `#f6c90e` | warning, reconnecting, provisioning |
| Yellow bg | `#3a3218` | warning backgrounds |
| Red | `#fc8181` | error, killed, danger |
| Red bg | `#2a1010` / `#3a1a1a` | error backgrounds |

Font stack: `-apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif`. No custom font loaded. Monospace uses system monospace.

**Interaction patterns already implemented:**
- Fingerprint dedup polling in `CustomerDetail` — `JSON.stringify` of 10 status fields; skip re-render if unchanged
- Adaptive poll interval — 5s when any infra component is `pending-install` / `pending-upgrade` / `not installed`; falls back to `customerSettings.customerPollMs` otherwise
- Lazy-load panels — credentials and command queue fetched only on explicit button click
- Post-action reload — mutations call `load()` directly or via `setTimeout(load, reloadAfterMs)`
- localStorage persistence — `cp_admin_token`, `cp_page`, `cp_customer_id`, `cp_settings`, `cp_settings_<customerId>`
- Customer context restore on refresh — `App.jsx` useEffect reads `cp_customer_id` from localStorage, fetches customer, navigates to CustomerDetail
- Connection status derivation — `connected: true` = Connected; `active: false` = Killed; last heartbeat within 2 min = Reconnecting; otherwise Offline

---

---

## 1. Auth Model

### Token Types

Two distinct token types — never interchangeable.

**Admin Token**
- A static secret set via `CONTROL_PLANE_ADMIN_TOKEN` env var on the CP server.
- Sent as `Authorization: Bearer <token>` on all CP dashboard API calls.
- No JWT — plain string comparison in `requireAdmin` middleware.
- Never sent to or stored by the operator.

**Agent Token**
- JWT signed with `CONTROL_PLANE_JWT_SECRET` env var on the CP server.
- Payload: `{ agentId: "acme-corp-a1b2c3", customerId: "Acme Corp", type: "agent", tv: 0, iat, exp }`
- Expires in 365 days. Expiry is not currently enforced by the operator (deferred to Phase 8).
- `tv` (token version) is compared against `customers.tokenVersion` in `requireAgent` — mismatch = revoked.
- Used by the operator for all CP communication: `Authorization: Bearer <agentToken>` on HTTP + WS.
- Issued once at customer onboarding, stored in `customers.agentToken` and written into the operator CRD spec.
- `requireAgent` middleware verifies JWT signature + token version match. On DB error it allows through (avoids breaking operator on transient DB issues).

### Agent ID Format

Generated from customer name: lowercase, non-alphanumeric → hyphens, leading/trailing hyphens stripped, **plus a 6-character random hex suffix**.
Example: "Acme Corp" → `acme-corp-a1b2c3`.

Note: no `intangles-` prefix in the generated ID — the prefix was removed. IDs are just `{slug}-{hex}`.

---

## 2. Environment Variables

### CP Server (`intangles-cp-platform/server/`)

| Var | Required | Default | Description |
|---|---|---|---|
| `CONTROL_PLANE_PORT` | No | `3010` | HTTP port |
| `MONGO_URI` | Yes | — | MongoDB connection string |
| `CONTROL_PLANE_JWT_SECRET` | Yes | `dev-secret-change-in-prod` | Signs agent tokens — must be stable across restarts |
| `CONTROL_PLANE_ADMIN_TOKEN` | Yes | `admin-dev-token` | Static secret for dashboard API access |
| `LOG_LEVEL` | No | `info` | Fastify logger level |

### Operator (reads from CRD spec, not env vars)

The operator reads all runtime config from its CRD spec — not environment variables. The only env vars the operator process itself reads are standard Kubernetes operator vars:

| Var | Source | Description |
|---|---|---|
| `POD_NAMESPACE` | Downward API | Namespace the operator runs in |
| `WATCH_NAMESPACE` | Deployment env | Namespace to watch for CRDs (defaults to POD_NAMESPACE) |

Agent credentials, CP URL, and all config come from `IntanglesPlatform.spec` — not env vars.

---

## 3. MongoDB Collection Schemas

### `customers`

```js
{
  _id: ObjectId,
  name: "Acme Corp",
  agentId: "intangles-acme-corp",          // unique, generated at onboarding
  agentToken: "<jwt>",                      // issued once, stored for operator setup.sh
  packs: ["core-pack"],                     // assigned pack slugs
  extraComponents: [],                      // component names added directly (not via pack)
  deletedComponents: [],                    // explicitly removed; overrides pack membership
  overrides: {                              // per-component overrides
    "parser": { memory: "512Mi", env: { LOG_LEVEL: "debug" } },
    "mongodb": { memory: "2Gi" }
  },
  images: {                                 // null until set from Operator tab
    backend: "ghcr.io/intangles/backend",
    dataApis: "ghcr.io/intangles/data-apis",
    registry: "ghcr.io/intangles",
    pullSecret: "regcred"
  },
  infraRepos: [],                           // legacy; now sourced from component catalog
  vehicleLimit: 1000,
  notes: "",
  active: true,                             // false = killed/revoked
  connected: false,                         // updated by ws-hub on connect/disconnect
  lastHeartbeat: ISODate,                   // updated by ws-hub on each heartbeat
  lastConnected: ISODate,
  lastDisconnected: ISODate,
  createdAt: ISODate,
  updatedAt: ISODate
}
```

### `operator_status`

Written by `ws-hub.js` on every heartbeat. Never written by admin routes.

```js
{
  _id: ObjectId,
  agentId: "intangles-acme-corp",
  lastHeartbeat: ISODate,
  updatedAt: ISODate,
  status: {                                 // mirrors IntanglesPlatformStatus from operator
    phase: "Running",                       // WaitingForConfig | Provisioning | Running
    currentVersion: "2.5.3",
    previousVersion: "2.5.1",
    licenseValid: true,
    infraReady: true,
    clusterUID: "abc-123",
    infraComponents: [                      // current (Phase 1/2) — replaced by componentStatuses in Phase 3b
      { name: "mongodb", chart: "bitnami/mongodb", version: "13.x.x", status: "deployed", error: "", events: [] }
    ],
    services: [                             // current (Phase 1/2) — replaced by componentStatuses in Phase 3b
      { name: "parser", ready: true, replicas: 1, desiredReplicas: 1, image: "...", memory: "256Mi" }
    ],
    // Added in Phase 3b:
    componentStatuses: [
      {
        name: "mongodb", type: "helm", phase: "Ready",
        reason: "", message: "", lastTransitionTime: ISODate,
        version: "13.x.x", helmRelease: "mongodb", helmRevision: 4
      },
      {
        name: "parser", type: "deployment", phase: "Ready",
        reason: "", message: "", lastTransitionTime: ISODate,
        readyReplicas: 1, desiredReplicas: 1, image: "ghcr.io/intangles/backend:2.5.3"
      }
    ]
  }
}
```

### `components`

```js
{
  _id: ObjectId,
  name: "parser",                           // unique; used as appType for services, release name for infra
  type: "service",                          // "service" | "infra"   (Phase 3b adds: "job" | "daemonset" | "cronjob")
  description: "MQTT message parser",

  // type: service fields
  image: "backend",                         // "backend" | "data-apis"
  memory: "256Mi",
  cpu: "100m",
  defaultEnv: { LOG_LEVEL: "info" },

  // type: infra fields
  chart: "bitnami/mongodb",
  chartVersion: "13.x.x",
  secretEnv: ["MONGO_ROOT_PASSWORD"],
  values: { "auth.rootUsername": "root" },
  configMapKeys: { MONGO_URI: "mongodb://root:{{ secret \"MONGO_ROOT_PASSWORD\" }}@mongodb:27017" },

  // Phase 3b additions (additive — safe to add to existing docs)
  dependsOn: ["mongodb", "rabbitmq"],       // component names that must be Ready first
  lifecycle: "continuous",                  // "continuous" | "runOnce" | "scheduled" (for jobs)

  createdAt: ISODate,
  updatedAt: ISODate
}
```

### `packs`

```js
{
  _id: ObjectId,
  slug: "core-pack",                        // unique; referenced by customers.packs[]
  name: "Core Pack",
  description: "Required for all deployments",
  core: true,                               // auto-assigned to new customers; locked in UI
  components: ["datastore", "parser", "mongodb", "rabbitmq", "redis", "internal-app"],
  createdAt: ISODate,
  updatedAt: ISODate
}
```

### `releases`

```js
{
  _id: ObjectId,
  version: "2.5.3",                         // semver string; used as operator image tag
  notes: "Bug fixes and performance improvements",
  urgent: false,                            // if true: shown with red badge in dashboard
  createdAt: ISODate
}
```

### `command_queue`

Commands queued when the agent is offline. Flushed by `ws-hub.js` on agent reconnect.

```js
{
  _id: ObjectId,
  agentId: "intangles-acme-corp",
  command: { type: "agent_update", image: "ghcr.io/intangles/operator:2.5.3" },
  createdAt: ISODate,
  // Phase 4 FE additions (command lifecycle tracking):
  sentAt: ISODate,                          // set by ws-hub when ws.send() succeeds
  ackedAt: ISODate,                         // set when operator sends { type: "ack", commandId: "..." }
  failedReason: ""                          // set if ack carries error field
}
```

---

## 4. REST API Contract

All routes are on the CP server. Auth type: `Admin` = `requireAdmin`, `Agent` = `requireAgent`.

| Method | Path | Auth | Request | Response |
|---|---|---|---|---|
| `POST` | `/api/auth/login` | None | `{ token }` | `{ ok: true }` or 401 |
| `GET` | `/api/customers` | Admin | — | `Customer[]` (with joined status) |
| `GET` | `/api/customers/:id` | Admin | — | `Customer` (with joined status) |
| `POST` | `/api/customers` | Admin | `{ name, vehicleLimit, notes }` | `{ ...customer, agentToken }` — token shown once |
| `PUT` | `/api/customers/:id` | Admin | `{ name, vehicleLimit, notes, images }` | `Customer` |
| `PUT` | `/api/customers/:id/overrides` | Admin | `{ overrides: { [name]: { memory, env } } }` | `{ ok }` |
| `POST` | `/api/customers/:id/packs/:slug` | Admin | — | `{ ok }` |
| `DELETE` | `/api/customers/:id/packs/:slug` | Admin | — | `{ ok }` |
| `POST` | `/api/customers/:id/components/:name/enable` | Admin | — | `{ ok }` |
| `POST` | `/api/customers/:id/components/:name/disable` | Admin | — | `{ ok }` |
| `POST` | `/api/customers/:id/command` | Admin | `{ type, ...payload }` | `{ sent: bool }` |
| `GET` | `/api/customers/:id/command-queue` | Admin | — | `CommandEntry[]` |
| `DELETE` | `/api/customers/:id/command-queue` | Admin | — | `{ ok }` |
| `GET` | `/api/customers/:id/credentials` | Admin | — | `{ [key]: { value, updatedAt } }` |
| `GET` | `/api/customers/:id/activity` | Admin | — | `ActivityEntry[]` (max 50, newest first) — **Phase 4** |
| `POST` | `/api/customers/:id/kill` | Admin | — | `{ ok }` — sets `active: false`, sends kill command |
| `POST` | `/api/customers/:id/reactivate` | Admin | — | `{ ok }` — sets `active: true`, sends reactivate |
| `GET` | `/api/components` | Admin | — | `Component[]` |
| `POST` | `/api/components` | Admin | `Component fields` | `Component` |
| `PUT` | `/api/components/:name` | Admin | `Component fields` | `Component` |
| `DELETE` | `/api/components/:name` | Admin | — | `{ ok }` |
| `POST` | `/api/components/:name/apply` | Admin | `{ customerId }` | `{ ok }` |
| `GET` | `/api/packs` | Admin | — | `Pack[]` |
| `POST` | `/api/packs` | Admin | `Pack fields` | `Pack` |
| `PUT` | `/api/packs/:slug` | Admin | `Pack fields` | `Pack` |
| `DELETE` | `/api/packs/:slug` | Admin | — | `{ ok }` |
| `GET` | `/api/releases` | Admin | — | `Release[]` |
| `POST` | `/api/releases` | Admin | `{ version, notes, urgent }` | `Release` |
| `POST` | `/api/releases/:version/deploy-all` | Admin | — | `{ sent: N }` |
| `GET` | `/config/:agentId` | Agent | — | Config response (see §5) |
| `POST` | `/telemetry/ingest` | Agent | Metrics payload | `{ ok }` — **Phase 5** |
| `POST` | `/credentials/:agentId` | Agent | `{ credentials: { [key]: value } }` | `{ ok }` |
| `GET` | `/license/:agentId` | Agent | — | `{ valid: true }` — stub in Phase 1–7 |

---

## 5. Config Response Shape

`GET /config/:agentId` — consumed by the operator.

### Current (Phase 1/2)

```json
{
  "version": "2.5.3",
  "images": {
    "backend": "ghcr.io/intangles/backend",
    "dataApis": "ghcr.io/intangles/data-apis",
    "registry": "ghcr.io/intangles",
    "pullSecret": "regcred"
  },
  "services": [
    {
      "appType": "parser",
      "image": "backend",
      "memory": "256Mi",
      "cpu": "100m",
      "defaultEnv": { "LOG_LEVEL": "info" }
    }
  ],
  "infra": {
    "repos": [
      { "name": "bitnami", "url": "https://charts.bitnami.com/bitnami" }
    ],
    "components": [
      {
        "name": "mongodb",
        "chart": "bitnami/mongodb",
        "chartVersion": "13.x.x",
        "secretEnv": ["MONGO_ROOT_PASSWORD"],
        "values": { "auth.rootUsername": "root" },
        "configMapKeys": { "MONGO_URI": "mongodb://root:{{ secret \"MONGO_ROOT_PASSWORD\" }}@mongodb:27017" }
      }
    ]
  },
  "overrides": {
    "parser": { "memory": "512Mi", "env": { "LOG_LEVEL": "debug" } }
  }
}
```

### Phase 3b (adds `components[]`, keeps old fields for backward compat)

```json
{
  "version": "2.5.3",
  "images": { ... },
  "components": [
    { "name": "mongodb",   "type": "helm",       "chart": "bitnami/mongodb",  "chartVersion": "13.x.x", "secretEnv": ["MONGO_ROOT_PASSWORD"], "values": {}, "configMapKeys": {}, "dependsOn": [] },
    { "name": "rabbitmq",  "type": "helm",       "chart": "bitnami/rabbitmq", "chartVersion": "12.x.x", "secretEnv": ["RABBITMQ_PASSWORD"],   "values": {}, "configMapKeys": {}, "dependsOn": [] },
    { "name": "redis",     "type": "helm",       "chart": "bitnami/redis",    "chartVersion": "19.x.x", "secretEnv": ["REDIS_PASSWORD"],      "values": {}, "configMapKeys": {}, "dependsOn": [] },
    { "name": "db-init",   "type": "job",        "appType": "db_init",        "image": "backend", "lifecycle": "runOnce", "dependsOn": ["mongodb", "rabbitmq"] },
    { "name": "parser",    "type": "deployment", "appType": "parser",         "image": "backend", "memory": "256Mi", "defaultEnv": {}, "dependsOn": ["db-init"] },
    { "name": "datastore", "type": "deployment", "appType": "datastore",      "image": "backend", "memory": "256Mi", "defaultEnv": {}, "dependsOn": ["db-init"] },
    { "name": "internal-app", "type": "deployment", "appType": "internal-app", "image": "data-apis", "memory": "512Mi", "defaultEnv": {}, "dependsOn": ["db-init"] }
  ],
  "services": [ ... ],    // kept — old operators ignore components[], use services[]
  "infra": { ... },       // kept — same reason
  "overrides": { ... }
}
```

**Operator migration rule:** If `components[]` is present and non-empty, use it. If absent, fall back to `services[]` + `infra{}`. Never use both simultaneously.

---

## 6. WebSocket Protocol

### Transport

- Operator connects to `ws://<controlPlane>/ws/agent/<agentId>`
- Auth: `Authorization: Bearer <agentToken>` in the upgrade request headers
- Operator reconnects with exponential backoff (initial 1s, max 30s, +jitter) on any close/error

### Operator → CP Messages

**Heartbeat** (sent every 30s)
```json
{
  "type": "heartbeat",
  "status": { ... }    // full IntanglesPlatformStatus — see operator_status.status in §3
}
```

**Ack** (Phase 4 — confirms command execution; sent after handler completes)
```json
{
  "type": "ack",
  "commandId": "<ObjectId of command_queue entry>",
  "ok": true,
  "error": ""          // non-empty string if handler failed
}
```

### CP → Operator Commands

Every command is a JSON object with a `type` field. Payload fields per type:

```json
{ "type": "kill" }

{ "type": "reactivate" }

{ "type": "config_change" }
// No payload — operator triggers immediate GET /config/:agentId

{ "type": "agent_update", "image": "ghcr.io/intangles/intangles-operator:2.5.3" }

{ "type": "restart_service", "appType": "parser" }
// Phase 3b alias: { "type": "restart_component", "name": "parser" }

{ "type": "scale_service", "appType": "parser", "replicas": 3 }
// Phase 3b alias: { "type": "scale_component", "name": "parser", "replicas": 3 }

{ "type": "delete_service", "appType": "parser" }
// Phase 3b alias: { "type": "delete_component", "name": "parser" }

{ "type": "retry_infra", "name": "mongodb" }
// Phase 3b alias: { "type": "retry_component", "name": "mongodb" }

{ "type": "restart_infra", "name": "mongodb" }
// Phase 3b alias: { "type": "restart_component", "name": "mongodb" }

{ "type": "upgrade_infra", "name": "mongodb", "chartVersion": "13.6.0" }
// Phase 3b alias: { "type": "upgrade_component", "name": "mongodb", "chartVersion": "13.6.0" }

{ "type": "delete_infra", "name": "mongodb" }
// Phase 3b alias: { "type": "delete_component", "name": "mongodb" }

{ "type": "reauth", "token": "<new jwt>" }

// Phase 4:
{ "type": "rotate_credentials", "components": ["mongodb", "rabbitmq"] }
// Empty array means rotate all credential-bearing infra components

// Phase 5:
{ "type": "collect_diagnostics", "component": "parser", "lines": 100 }
// Operator responds with a heartbeat carrying diagnostics payload in status
```

**Command ID:** When the CP sends a command from the queue it should include the MongoDB `_id` so the operator's ack can reference it:
```json
{ "type": "restart_service", "appType": "parser", "_commandId": "64a1b2c3d4e5f6..." }
```
For commands sent directly (not from queue) `_commandId` is omitted; ack is skipped.

---

## 7. Go Handler Interface (Phase 3a)

The reconciler dispatches to handlers via this interface. All three handlers (`HelmHandler`, `DeploymentHandler`, `JobHandler`) implement it.

```go
// ComponentHandler manages the full lifecycle of one component type.
type ComponentHandler interface {
    // Reconcile brings the component's K8s resources to the desired state.
    // Returns the current ComponentStatus after reconciliation.
    // Must be idempotent — called on every reconcile pass.
    Reconcile(ctx context.Context, req ComponentReconcileRequest) (ComponentStatus, error)

    // Delete removes all K8s resources owned by this component.
    // Uses label selector: intangles.io/component=<name>
    Delete(ctx context.Context, name string, namespace string) error
}

type ComponentReconcileRequest struct {
    Component  ComponentSpec
    Platform   *IntanglesPlatform    // full CRD — for namespace, images, overrides
    Client     client.Client         // controller-runtime client
    Recorder   record.EventRecorder
}
```

**Dispatch in reconciler:**
```go
switch component.Type {
case ComponentTypeHelm:       handler = r.helmHandler
case ComponentTypeDeployment: handler = r.deploymentHandler
case ComponentTypeJob:        handler = r.jobHandler
}
status, err = handler.Reconcile(ctx, req)
```

**Component ownership label** applied by every handler on every created/updated resource:
```go
labels["intangles.io/component"] = component.Name
labels["intangles.io/platform"]  = platform.Name
```
`Delete()` uses these labels as a selector — never touches resources without them.

---

## 8. Health Probe Connection Strategy (Phase 3c)

The operator connects to infra components **directly within the cluster** using the connection strings in the `intangles-config` ConfigMap. It does not exec into pods or use K8s metrics API.

**Credential source:** The `intangles-credentials` K8s Secret — same secret the operator already creates and maintains. The operator reads it directly via the K8s client (`client.Get`).

**Config source:** Rendered connection strings from `intangles-config` ConfigMap. The operator already builds this — it reads it back to extract hostnames and ports for probe targets.

**Probe implementations:**

```go
// MongoDB: TCP dial to mongodb:27017 — lightweight, no auth needed for reachability
func probeMongoDB(host string, port int) error {
    conn, err := net.DialTimeout("tcp", fmt.Sprintf("%s:%d", host, port), 3*time.Second)
    if err != nil { return err }
    conn.Close()
    return nil
}

// Redis: PING command using go-redis client
func probeRedis(addr string, password string) error {
    rdb := redis.NewClient(&redis.Options{Addr: addr, Password: password})
    defer rdb.Close()
    return rdb.Ping(context.Background()).Err()
}

// RabbitMQ: HTTP GET to management API /api/overview
func probeRabbitMQ(host string, user string, password string) error {
    url := fmt.Sprintf("http://%s:15672/api/overview", host)
    // Basic auth with management credentials from intangles-credentials secret
    ...
}
```

Probes run **on every reconcile pass** for Ready/Degraded components — not a separate goroutine or ticker. If a probe fails, the component moves to Degraded (not Failed) — the operator does not restart or re-deploy.

---

## 9. Credential Rotation Sequencing (Phase 4)

`rotate_credentials` command specifies a list of infra component names. The operator processes them **sequentially, not concurrently** to prevent credential conflicts.

**Per-component rotation steps (strict order, stop on failure):**

```
1. Generate new random hex value (32 chars) for each key in component.SecretEnv
2. Write new values to intangles-credentials K8s Secret (patch, not replace)
3. Helm upgrade the component with new --set values (same as regular upgrade path)
4. Wait for component to return to Ready phase (poll ComponentStatus, timeout 10m)
   → If timeout/failure: leave Secret with new value, set ComponentStatus.Phase = Failed,
     ComponentStatus.Reason = "CredentialRotationFailed". Do NOT roll back Secret.
     Reason: rolling back would leave old Secret but Helm may have partially applied new values.
     Operator surfaces failure clearly; human intervention required.
5. Rebuild intangles-config ConfigMap with new connection strings (re-run ensureConfigMap)
6. Rolling restart all deployment components where dependsOn includes this infra component
   (bump restartedAt annotation on each Deployment — same as restart_service handler)
7. Update ComponentStatus: Phase = Ready, add rotation timestamp to Message field
```

**Status field during rotation:** Use a dedicated `ComponentStatus.Reason` value:
```go
const ReasonCredentialRotationInProgress = "CredentialRotationInProgress"
const ReasonCredentialRotationFailed     = "CredentialRotationFailed"
```
Phase stays `Deploying` while rotation is in progress. Returns to `Ready` on success.

---

## 10. Command Queue Lifecycle (Phase 4)

**Who writes what:**

| Field | Who writes it | When |
|---|---|---|
| `createdAt` | CP server — `queueCommand()` in ws-hub.js | When agent is offline and command can't be sent immediately |
| `sentAt` | CP server — `flushQueue()` in ws-hub.js | After `ws.send()` succeeds (no error thrown) |
| `ackedAt` | CP server — `handleAgentMessage()` in ws-hub.js | When a `{ type: "ack", commandId, ok: true }` message arrives from operator |
| `failedReason` | CP server — `handleAgentMessage()` | When ack carries `ok: false, error: "..."` |

**For commands sent immediately** (agent online): `createdAt` is not set (document never created). The `sentAt` is inferred from the command-send timestamp logged in the customers route. To support full lifecycle for directly-sent commands too, the customers route should insert a command doc with `createdAt` = `sentAt` = now when the command is sent directly.

**"Retry" from dashboard:** Sends the same command type + payload again. Does not reuse the failed queue entry — creates a new one.

**Auto-cleanup:** Entries with `ackedAt` set are deleted after 24h. Entries older than 72h with no `sentAt` are deleted (agent never came online). Failed entries persist until manually cleared or retried.

---

## 11. Activity Timeline (Phase 4)

### Collection: `customer_activity`

```js
{
  _id: ObjectId,
  customerId: ObjectId,                     // ref to customers._id
  agentId: "intangles-acme-corp",
  type: "kill",                             // see event types below
  payload: { image: "..." },               // command payload or relevant fields (no credentials)
  actorIp: "10.0.0.1",                     // req.ip from admin route
  timestamp: ISODate
}
```

**Retention:** Rolling 50 entries per customer (insert + trim in single operation):
```js
await db.collection("customer_activity").insertOne(entry);
// Keep only latest 50
const docs = await db.collection("customer_activity")
  .find({ customerId }, { projection: { _id: 1 }, sort: { timestamp: -1 }, skip: 50 })
  .toArray();
if (docs.length) await db.collection("customer_activity")
  .deleteMany({ _id: { $in: docs.map(d => d._id) } });
```

**Events to log** (every write action from admin routes):

| Event type | Triggered by |
|---|---|
| `kill` | Kill command sent |
| `reactivate` | Reactivate command sent |
| `agent_update` | Operator update sent |
| `config_change` | Config change pushed |
| `restart_service` | Restart command sent |
| `scale_service` | Scale command sent |
| `delete_service` | Delete service command sent |
| `upgrade_infra` | Infra upgrade command sent |
| `delete_infra` | Infra delete command sent |
| `rotate_credentials` | Credential rotation triggered |
| `overrides_updated` | Customer overrides saved |
| `images_updated` | Service images saved |
| `pack_added` | Pack assigned to customer |
| `pack_removed` | Pack removed from customer |
| `component_enabled` | Component re-enabled |
| `component_disabled` | Component disabled |
| `token_regenerated` | Agent token regenerated |

---

## 12. Dashboard Fallback — Old Agent Shape → Unified Component Status

When `status.componentStatuses[]` is absent (old operator), the dashboard must synthesize a compatible view from `status.services[]` and `status.infraComponents[]`.

**Mapping rules:**

```js
// Old services[] → ComponentStatus equivalent
function serviceToComponent(s) {
  return {
    name: s.name,
    type: "deployment",
    phase: s.ready ? "Ready" : "Failed",
    reason: s.ready ? "" : "NotReady",
    message: "",
    readyReplicas: s.replicas,
    desiredReplicas: s.desiredReplicas,
    image: s.image,
    lastTransitionTime: null
  };
}

// Old infraComponents[] → ComponentStatus equivalent
const helmPhaseMap = {
  "deployed":        "Ready",
  "failed":          "Failed",
  "pending-install": "Deploying",
  "pending-upgrade": "Deploying",
  "uninstalling":    "Deploying",
  "":                "Pending"
};
function infraToComponent(c) {
  return {
    name: c.name,
    type: "helm",
    phase: helmPhaseMap[c.status] || "Failed",
    reason: c.error ? "HelmError" : "",
    message: c.error || "",
    helmRelease: c.name,
    lastTransitionTime: null
  };
}
```

**Detection:** Old shape if `status.componentStatuses` is `undefined` or `null`.
New shape if `status.componentStatuses` is an array (may be empty).

Apply fallback in the data layer (TanStack Query `select` transform in FE Phase 1), not in individual components. Every consumer of status data then sees a uniform `componentStatuses[]` shape regardless of operator version.

---

## 13. CP Deployment

The CP is a Node.js/Fastify server + React/Vite static dashboard. No containers or orchestration required for the CP itself — it runs wherever the Intangles team hosts their backend.

**Server:** `node server/src/server.js` — or via PM2: `pm2 start server/src/server.js --name cp-server`

**Dashboard:** Built with `npm run build` in `dashboard/`. Vite outputs to `dashboard/dist/`. Served as static files by the Node.js server (or a CDN/nginx in front of it).

**The CP is not on-prem** — it always runs on Intangles infrastructure (AWS). Only the operator runs on customer clusters.

---

## 14. Deprecated / Renamed Fields Reference

When Claude encounters old field names in code, this is the current → future mapping:

| Old (Phase 1/2) | New (Phase 3b+) | Location |
|---|---|---|
| `spec.services[]` | `spec.components[]` (type=deployment/job) | CRD spec |
| `spec.infra.components[]` | `spec.components[]` (type=helm) | CRD spec |
| `spec.infra.repos[]` | Auto-registered by HelmHandler from known registries | CRD spec |
| `status.services[]` | `status.componentStatuses[]` (type=deployment) | CRD status |
| `status.infraComponents[]` | `status.componentStatuses[]` (type=helm) | CRD status |
| `ServiceSpec.appType` | `ComponentSpec.AppType` | Go types |
| `InfraComponent.name` | `ComponentSpec.Name` | Go types |
| `restart_service` command | `restart_component` (old name still handled) | WS protocol |
| `delete_service` command | `delete_component` (old name still handled) | WS protocol |
| `customers.deletedComponents[]` | same field name — now covers all component types | MongoDB |

Old names always continue to work — backward compatibility is never broken mid-transition.
