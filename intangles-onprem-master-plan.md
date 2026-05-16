# Intangles On-Prem — Master Plan

> Single source of truth. Replaces all previous plan docs.
> Update this file as features are added, completed, or dropped.
> Legend: ✅ Done · ⏳ In Progress · 🔲 Pending · ⛔ Not Planned (low priority — implement only when explicitly instructed)

---

## What We're Building

Deploy the Intangles telematics platform on **customer-owned Kubernetes clusters** as a managed product.

- Our code runs on their hardware, protected (compiled Go operator + obfuscated JS services)
- We retain full remote control (updates, packs, kill switch, observability)
- Customer data stays entirely in their infrastructure
- We manage everything via our control plane — no SSH/VPN access needed
- Same platform used for new region deployments

---

## Design Principles

1. **Control plane is the source of truth.** All desired state (components, packs, versions, overrides) originates from the CP. The operator never invents desired state — it reads from the CRD (written by CP sync) and reconciles toward it.
2. **Operator owns orchestration and runtime state.** The operator owns lifecycle transitions, dependency ordering, retry logic, health evaluation, and reconciliation behavior. Runtime knowledge (what's actually running, why it's degraded, when to retry) lives in the operator, not the CP.
3. **Components are the single deployment abstraction.** Every deployable unit — Helm release, Deployment, Job, DaemonSet, CronJob — is a `Component`. One lifecycle model, one status model, one reconciliation path.
4. **Dependency-graph-based ordering.** Deploy order is derived from `dependsOn[]` declarations on each component, not hardcoded phases. Explicit, testable, extensible without touching orchestration code.
5. **Backward compatibility by default.** Old CRD fields, old config shapes, old command names continue working during transitions. New fields are additive. Migrations are never forced.
6. **Kubernetes-native patterns.** Use K8s conditions, owner references, finalizers, annotations, and controller-runtime primitives. Avoid reimplementing what K8s already provides.
7. **Simplicity over enterprise complexity.** Add features when justified by actual operational need, not theoretical future requirements. Every abstraction must earn its place.

---

## Architecture

```
Our Control Plane (AWS)                    Customer K8s Cluster
─────────────────────                      ──────────────────────
Node.js API + MongoDB                      intangles-operator (Go binary)
React Dashboard                ←──WSS──→   └─ Reconciler (component graph → K8s state)
                                           └─ CP Client (config sync + polling fallback)
                                           └─ WS Client (push commands + config_change)
                                           └─ Health Evaluator (semantic readiness)
                                               │
                                           Unified Component Graph
                                           ├─ type:helm    → MongoDB, Redis, RMQ, ...
                                           ├─ type:job     → db-init (runs once)
                                           ├─ type:deployment → parser, datastore, ... (APP_TYPE)
                                           └─ (future) type:daemonset, cronjob, ...
```

**CP as source of truth:** The CP holds all desired state — components, packs, versions, overrides. It delivers config changes to the operator via WebSocket push (`config_change`) for immediate effect, with periodic HTTP polling as a reliable fallback for startup recovery and reconnect scenarios. The operator writes received config to the CRD and manages reconciliation toward that desired state.

**Operator owns orchestration:** The operator owns dependency resolution, retry behavior, health evaluation, lifecycle transitions, and all runtime knowledge. CP never directly manages K8s resources — it only communicates desired state and receives status via heartbeat.

**Unified Component Model:** Every deployable unit (infra, jobs, services) is a `Component` with shared lifecycle behavior, dependency declarations, and status model. The reconciler topologically sorts the dependency graph and deploys in correct order. CP sends a flat `components[]` list — no separate services/infra split.

**Component ownership boundary:** Each component owns only the K8s resources created by its handler. Resources are labeled with the component's name (`intangles.io/component: <name>`) and the platform's owner reference. A handler only reconciles and cleans up resources matching its own label — preventing cross-component conflicts and accidental deletion of resources belonging to other components.

**Pack Model:** Packs are named lists of component names. CP resolves the union of all assigned packs at config-request time (smart-stop: shared components stay running as long as any pack references them).

**APP_TYPE compatibility:** Preserved. `type: deployment` components set `APP_TYPE` env var — zero changes to existing service code.

---

## Phase 1 — Core Operator + Control Plane ✅

**Goal:** Operator deploys and manages services. CP communicates with agents.

| Item | Status | Notes |
|---|---|---|
| Go operator scaffolded (Kubebuilder) | ✅ | `intangles-operator/` repo |
| IntanglesPlatform CRD | ✅ | `api/v1alpha1/types.go` |
| Phase progression: WaitingForConfig → Provisioning → Initializing → Running | ✅ | `internal/reconciler/reconciler.go` |
| Config sync from CP (`GET /config/:agentId`) | ✅ | Writes to CRD spec |
| Helm infra manager (install/upgrade/uninstall) | ✅ | `internal/infra/infra.go` |
| Credential generation → K8s Secret (`intangles-credentials`) | ✅ | Random hex per `secretEnv` key |
| ConfigMap build with connection strings (`intangles-config`) | ✅ | Templates resolved from secret + ns |
| Service deployment from `spec.services` (APP_TYPE model) | ✅ | `internal/reconciler/features.go` |
| DB-init Job (seeds MongoDB, RMQ topology, admin user) | ✅ | `APP_TYPE=db_init` |
| Self-healing reconciler (30s interval) | ✅ | Drift detection + repair |
| CP Node.js API server | ✅ | Express + MongoDB |
| CP React dashboard | ✅ | Component+pack management |
| CP WebSocket hub (agent connections) | ✅ | Per-agent connection tracking |
| Customer CRUD + pack assignment | ✅ | |
| Component catalog (service + infra types) | ✅ | |
| Pack management with core pack auto-assign | ✅ | `core: true` packs locked |
| Per-component disable/enable toggle | ✅ | `deletedComponents` on customer |
| Customer onboarding script (`setup.sh`) | ✅ | Renders + applies CRD → operator → platform |
| CRD dry-run support | ✅ | `--dry-run` flag on setup.sh |
| Local development: `make local`, `make redeploy`, `make e2e` | ✅ | Kind cluster + hot-reload |
| E2E test with mock CP | ✅ | `make e2e ADMIN_TOKEN=...` |

---

## Phase 2 — Remote Control + Updates ✅

**Goal:** Full remote command system, operator self-update, kill switch.

| Item | Status | Notes |
|---|---|---|
| WebSocket client (persistent to CP) | ✅ | `internal/controlplane/websocket.go` |
| WS heartbeat every 30s with full platform status | ✅ | `{ type: "heartbeat", status: ... }` |
| WS linear reconnect (capped 30s) | ✅ | |
| WS commands: kill, reactivate, config_change | ✅ | |
| WS commands: agent_update, restart_service, scale_service | ✅ | |
| WS commands: delete_service, retry_infra, restart_infra | ✅ | |
| WS commands: upgrade_infra, delete_infra, reauth | ✅ | |
| Operator self-update (patches own Deployment image) | ✅ | `agent_update` command |
| Kill switch: scale all to 0 via `spec.paused` | ✅ | |
| Reactivate: clears paused + bumps sync annotation | ✅ | |
| License stub (returns true — full licensing in Phase 8) | ✅ | Validates format only; always permits |
| CP-side: live status join from `operator_status` collection | ✅ | Batch `$in` query |
| CP-side: batch pack query in config.js (no N+1) | ✅ | |
| CP dashboard: page refresh restores customer context | ✅ | localStorage |
| Release script (`scripts/release.sh`) | ✅ | Multi-arch build + push |
| Update operator script (`scripts/update-operator.sh`) | ✅ | Patches image + rollout wait |
| Release management in CP dashboard | ✅ | Publish + deploy |
| WS reconnect: exponential backoff with jitter | 🔲 | Replace linear reconnect with exp+jitter |
| Debounce `restart_service` commands (< 5s window) | 🔲 | Prevent restart storms from rapid clicks |
| WS config push + polling fallback | 🔲 | See Phase 3b — `config_change` WS triggers immediate re-fetch; 30s polling covers startup + reconnect |

---

## Phase 3 — Unified Component Model

**Goal:** Replace separate services/infra handling with a single generic `Component` abstraction. Split into three safe sub-phases to reduce migration risk. Backward-compatible throughout — existing deployments continue working unchanged at every sub-phase boundary.

### Why

The current split (`spec.services[]` + `spec.infra{}`) creates two separate code paths, two status models, and hardcoded ordering (infra → db-init → services). A unified model simplifies the reconciler, makes dependency ordering explicit and declarative, and enables the platform to evolve (new component types, per-component policies, semantic health) without touching core orchestration code.

---

### Phase 3a — Component Types + Status Model (internal refactor, no CRD change)

**Goal:** Introduce the Go type system and internal handler dispatch without touching the CRD schema or CP config format. Existing reconciliation continues working exactly as before.

#### Component Model (Go)

```go
type ComponentSpec struct {
    Name      string        // unique name: "mongodb", "parser", "db-init"
    Type      ComponentType // "helm" | "deployment" | "job" | "daemonset" | "cronjob"
    DependsOn []string      // names that must be Ready before this starts
    Version   string        // optional pinned version (safer upgrades, rollback, future canary)

    // type:deployment + type:job
    AppType   string
    Image     string            // image key from spec.images
    Lifecycle ComponentLifecycle // "continuous" | "runOnce" | "scheduled"

    // type:helm
    Chart         string
    ChartVersion  string
    SecretEnv     []string
    Values        map[string]string
    ConfigMapKeys map[string]string

    // Resources
    Memory   string
    CPU      string
    Replicas *int32

    // Config
    DefaultEnv map[string]string
}

type ComponentType     string // "helm" | "deployment" | "job" | "daemonset" | "cronjob"
type ComponentLifecycle string // "continuous" | "runOnce" | "scheduled"
```

**Component version field** enables the operator to detect version mismatches per component, support targeted rollback, and forms the foundation for future canary deployments — without requiring per-component image tags yet.

#### Component Status Model

```go
type ComponentStatus struct {
    Name               string
    Type               ComponentType
    Phase              ComponentPhase // Pending | Deploying | Ready | Degraded | Failed | Disabled
    Reason             string         // machine-readable
    Message            string         // human-readable
    LastTransitionTime metav1.Time
    Version            string
    // For deployments/jobs
    ReadyReplicas   int32
    DesiredReplicas int32
    Image           string
    // For helm
    HelmRelease string
    HelmRevision int
}

type ComponentPhase string
const (
    ComponentPhasePending   ComponentPhase = "Pending"   // waiting on dependencies
    ComponentPhaseDeploying ComponentPhase = "Deploying" // in progress
    ComponentPhaseReady     ComponentPhase = "Ready"     // healthy
    ComponentPhaseDegraded  ComponentPhase = "Degraded"  // running but health signal present (see Message)
    ComponentPhaseFailed    ComponentPhase = "Failed"    // error state
    ComponentPhaseDisabled  ComponentPhase = "Disabled"  // explicitly disabled
)
```

**Degraded** is distinct from Failed: the component is running but has a health signal worth surfacing (e.g. queue depth rising, redelivery rate elevated, consumer lag). This allows the dashboard to show warnings without triggering a reconcile/restart cycle.

#### Handlers (internal dispatch — no CRD change yet)

```
Reconciler wraps existing infra.go + features.go logic behind handler interfaces:
  HelmHandler.Reconcile(component)       ← wraps current infra.go
  DeploymentHandler.Reconcile(component) ← wraps current features.go
  JobHandler.Reconcile(component)        ← wraps current db-init logic

Each handler only reads/writes K8s resources labeled with its component name.
Resources not matching the handler's label are never touched.
```

| Item | Status | Notes |
|---|---|---|
| `ComponentSpec` / `ComponentStatus` / `ComponentPhase` Go types | 🔲 | New file: `api/v1alpha1/component_types.go` |
| `ComponentLifecycle` type + constants | 🔲 | `continuous`, `runOnce`, `scheduled` |
| `DeploymentHandler` wrapping current features.go | 🔲 | Same logic, handler interface |
| `HelmHandler` wrapping current infra.go | 🔲 | Same logic, handler interface |
| `JobHandler` wrapping current db-init logic | 🔲 | RunOnce lifecycle + completion tracking |
| Component ownership label (`intangles.io/component`) on all created resources | 🔲 | Applied by each handler on create; used as label selector for cleanup |
| Handler dispatch in reconciler (by component type) | 🔲 | Internal only — no CRD changes yet |
| Unit tests for each handler | 🔲 | |

---

### Phase 3a — Dashboard Changes

> Internal Go refactor only — no CRD or API shape changes. No dashboard work required for 3a.

---

### Phase 3b — CRD Migration + Dependency Graph + Config Push

**Goal:** Introduce `spec.components[]` and `status.componentStatuses[]` to the CRD. Old fields kept and still reconciled. Topological sort replaces hardcoded ordering. WS `config_change` triggers immediate re-fetch; periodic polling covers startup and reconnect.

#### CRD Changes

```go
type IntanglesPlatformSpec struct {
    AgentID      string `json:"agentID"`
    AgentToken   string `json:"agentToken"`
    ControlPlane string `json:"controlPlane"`
    Version      string `json:"version,omitempty"`

    // New unified field (Phase 3b)
    Components []ComponentSpec `json:"components,omitempty"`

    // Kept for backward compatibility (Phase 1/2 CRDs still work)
    Services []ServiceSpec `json:"services,omitempty"`
    Infra    *InfraSpec    `json:"infra,omitempty"`

    Images    ImagesSpec                   `json:"images,omitempty"`
    Overrides map[string]ComponentOverride `json:"overrides,omitempty"`
    Paused             bool     `json:"paused,omitempty"`
    DisabledComponents []string `json:"disabledComponents,omitempty"`
}

type IntanglesPlatformStatus struct {
    Phase             PlatformPhase
    Conditions        []metav1.Condition // FSM conditions with reason/message/lastTransitionTime
    ComponentStatuses []ComponentStatus  // unified — replaces serviceStatuses + infraStatuses
    CurrentVersion    string
    LastSyncTime      metav1.Time
}
```

#### Dependency Graph + Reconciler Flow

```
Reconciler receives spec.components[] (or converts services[]+infra{} → components if old format)
         │
         v
Topological sort (Kahn's algorithm) → ordered deploy list
         │
         v
For each component in sorted order:
  1. All dependsOn Phase=Ready? → if not, mark Pending, skip
  2. In disabledComponents? → ensure removed (by component label selector), mark Disabled
  3. Dispatch to handler (Helm / Deployment / Job)
  4. Evaluate semantic health (Phase 3c)
  5. Update ComponentStatus
         │
         v
Platform phase derived from component statuses:
  All Ready                          → Running
  Any Required component Failed      → Failed
  Any component Deploying/Pending    → Provisioning
  Any component Degraded, rest Ready → Running (warnings surfaced in Conditions)
```

#### Config Push + Polling Fallback

```
Primary:  CP pushes config_change via WS
          → operator triggers immediate GET /config/:agentId
          → writes to CRD spec → reconcile triggered

Fallback: Periodic HTTP poll (every 30s)
          → catches missed pushes, covers startup, covers reconnect window
          → same GET /config/:agentId path

Both paths write to the same CRD spec field.
Operator does not reconcile twice if config is unchanged (checksum comparison).
```

#### CP Config Response (new format)

```json
{
  "version": "2.5.3",
  "images": { ... },
  "components": [
    { "name": "mongodb",   "type": "helm",       "chart": "bitnami/mongodb",  "dependsOn": [] },
    { "name": "redis",     "type": "helm",       "chart": "bitnami/redis",    "dependsOn": [] },
    { "name": "rabbitmq",  "type": "helm",       "chart": "bitnami/rabbitmq", "dependsOn": [] },
    { "name": "db-init",   "type": "job",        "appType": "db_init",        "lifecycle": "runOnce", "dependsOn": ["mongodb","rabbitmq"] },
    { "name": "parser",    "type": "deployment", "appType": "parser",         "dependsOn": ["db-init"] },
    { "name": "datastore", "type": "deployment", "appType": "datastore",      "dependsOn": ["db-init"] },
    { "name": "data-apis", "type": "deployment", "appType": "internal-app",   "dependsOn": ["db-init"] }
  ]
}
```

| Item | Status | Notes |
|---|---|---|
| CRD: add `spec.components[]` (additive — old fields remain) | 🔲 | `make generate` after types change |
| CRD: `status.componentStatuses[]` replaces `serviceStatuses` + `infraStatuses` | 🔲 | Old status fields kept during transition |
| Reconciler: convert old `services[]`/`infra{}` → `components[]` if new field absent | 🔲 | Migration shim in reconciler |
| Topological sort (Kahn's algorithm) | 🔲 | `internal/reconciler/graph.go` |
| Unified reconciler loop (single ordered pass through component graph) | 🔲 | Replaces split infra/service reconciliation |
| WS `config_change` → immediate GET /config re-fetch + reconcile | 🔲 | Primary config delivery path |
| Periodic polling fallback (30s) for startup + reconnect recovery | 🔲 | Config checksum comparison to skip no-op reconciles |
| WS commands: generic aliases (`restart_component`, etc.) | 🔲 | Old command names (`restart_service`, etc.) keep working |
| CP config.js: emit `components[]` (alongside old fields for transition) | 🔲 | CP-side schema update |
| CP catalog: components carry `type` + `dependsOn` fields | 🔲 | Extend existing component model |
| deploy/crd.yaml regenerated | 🔲 | `make generate` |
| Mock CP updated to emit `components[]` | 🔲 | `hack/mock-controlplane/main.go` |
| E2E test: verify dependency ordering (db-init before parser) | 🔲 | |

### Phase 3b — Dashboard Changes

The heartbeat shape changes from separate `services[]` + `infraComponents[]` to a unified `componentStatuses[]`. The dashboard must consume the new shape while staying backward-compatible with agents still sending the old shape.

| Item | Status | Notes |
|---|---|---|
| CustomerDetail: render Services tab from `status.componentStatuses[]` where `type=deployment` | 🔲 | Falls back to `status.services[]` if `componentStatuses` absent (old agent) |
| CustomerDetail: render Infrastructure tab from `status.componentStatuses[]` where `type=helm` | 🔲 | Falls back to `status.infraComponents[]` if absent |
| CustomerDetail: show `type=job` components in a new "Jobs" section or merged into Infra tab | 🔲 | db-init and future jobs need a visible status row |
| CustomerDetail: show `Pending` phase — component waiting on dependency | 🔲 | New phase not currently rendered; show as gray "waiting on: X" |
| CustomerDetail: show `Disabled` phase — component explicitly off | 🔲 | Distinct from stopped/failed; show as muted with re-enable action |
| CustomerDetail: show `dependsOn` in component detail expand (read-only) | 🔲 | Helps operators understand why a component is Pending |
| CustomerList: status bar "Services N/M" counts from `componentStatuses[]` | 🔲 | Same fallback logic |
| CP catalog: `dependsOn` field on component form (comma-separated names) | 🔲 | Lets CP operators set dependency graph |
| CP catalog: `type` field extended to include `job`, `daemonset`, `cronjob` options | 🔲 | Currently only `service` and `infra` |

---

### Phase 3c — Semantic Health Evaluation

**Goal:** Health evaluation beyond pod readiness. Degraded states, infrastructure connectivity, queue health, and consumer progress — surfaced in `ComponentStatus` without triggering spurious restarts.

#### Health Signals (per component type)

| Component | Signals Evaluated |
|---|---|
| MongoDB (helm) | TCP connect to primary, replication lag < threshold, disk < threshold |
| RabbitMQ (helm) | Management API reachable, consumer count > 0 on critical queues, DLQ depth == 0 |
| Redis (helm) | PING response, evictions == 0, memory < threshold |
| Deployments | readyReplicas == desiredReplicas, restart rate < threshold over 5-minute window, no OOMKilled pods |
| Jobs (runOnce) | Job completed (succeeded), not in exponential backoff loop |

**Retry storm detection:** If a deployment's restart count exceeds 3 within a rolling 5-minute window, the component moves to `Degraded` (not `Failed`) with `Reason: RestartStorm`. The window resets when restarts fall below threshold. Operator surfaces this in status and heartbeat — does not auto-restart or scale down.

**Degradation evaluation window:** All rate-based signals (restart rate, queue depth trend, redelivery rate) use a 5-minute rolling window. A single spike does not trigger a phase transition — the signal must persist across the window. This prevents flapping during normal startup bursts.

**Consumer progress** (RabbitMQ): queue depth trending up over 5 minutes + consumers present = `Degraded/SlowConsumer`. Queue depth rising + no consumers = `Failed/NoConsumers`.

**Dependency-aware readiness:** A component is only `Ready` when both its own health checks pass AND all `dependsOn` components are `Ready`. A component moves to `Degraded` if any direct dependency moves from `Ready` to `Degraded`.

| Item | Status | Notes |
|---|---|---|
| Infrastructure health probes (MongoDB connect, RMQ API, Redis PING) | 🔲 | Lightweight — run on reconcile pass, not a separate polling loop |
| Queue health evaluation (depth trend, consumer count, DLQ) | 🔲 | RMQ management API |
| Restart storm detection (> 3 restarts in 5-minute rolling window) | 🔲 | Per-deployment; surfaces as Degraded/RestartStorm |
| Rate-based degradation with 5-minute rolling window | 🔲 | Prevents phase flapping on transient spikes |
| `Degraded` phase propagation to direct dependents | 🔲 | Downstream marks Degraded if upstream is Degraded |
| Health signals included in heartbeat status to CP | 🔲 | Drives CP dashboard health view |

### Phase 3c — Dashboard Changes

| Item | Status | Notes |
|---|---|---|
| CustomerDetail: `Degraded` phase — yellow warning row distinct from red Failed | 🔲 | Component row shows yellow indicator + `Reason` string (e.g. "RestartStorm", "SlowConsumer") |
| CustomerDetail: `Message` field shown below component name when Degraded or Failed | 🔲 | Human-readable explanation from operator |
| CustomerDetail: health signal badges on infra rows (e.g. "DLQ: 5", "Lag: high") | 🔲 | Sourced from heartbeat `componentStatuses[].message` |
| CustomerDetail: `LastTransitionTime` shown as relative time on component row | 🔲 | "In this state for 12m" — helps triage |
| CustomerDetail: Degraded propagation visible — show "(dependency degraded)" on downstream components | 🔲 | Reason field will carry this; render it distinctly |
| CustomerList: Degraded components reflected in per-customer status (warning indicator) | 🔲 | Customer row could show amber "⚠ 2 degraded" alongside connected status |

---

## Phase 4 — Operator Production-Hardening

**Goal:** Operator is stable, predictable, and safe in production at scale.

### Stability + Safety

| Item | Status | Notes |
|---|---|---|
| CRD validation (required fields: agentID, agentToken, controlPlane) | 🔲 | `+kubebuilder:validation` markers on types.go |
| FSM conditions on CRD status (reason + message + lastTransitionTime) | 🔲 | Kubernetes conditions pattern |
| ConfigMap checksum annotation → auto-restart services on config change | 🔲 | Hash `intangles-config` → pod template annotation |
| Graceful shutdown (preStop hook + terminationGracePeriodSeconds) | 🔲 | Per-component Deployment spec |
| Operator /healthz and /readyz probes | 🔲 | HTTP endpoints on operator pod |
| Leader election (for HA operator — 2 replicas, one active) | 🔲 | `--leader-elect` flag + lease |
| Configurable Helm timeout (default 10m, override per component) | 🔲 | Currently hardcoded |
| Helm retry threshold (stop after N failures, surface to ComponentStatus) | 🔲 | Currently retries indefinitely |
| Local config cache (offline mode if CP unreachable at startup) | 🔲 | Cache last good config to disk |
| WS reconnect: exponential backoff with jitter | 🔲 | Replaces current linear reconnect |
| Debounce `restart_*` commands (< 5s window) | 🔲 | Prevent restart storms from rapid dashboard clicks |

### Reconciliation Backpressure

Prevents config storms and excessive reconcile loops when many events arrive simultaneously (e.g. CP push + annotation bump + WS command in quick succession).

| Item | Status | Notes |
|---|---|---|
| Reconcile deduplication (coalesce multiple events into one reconcile pass) | 🔲 | controller-runtime work queue deduplicates by key — ensure no bypasses |
| Rate-limited reconcile queue (token bucket, default 10 rps) | 🔲 | `workqueue.NewItemExponentialFailureRateLimiter` in controller setup |
| Config-change debounce (5s settle window before re-fetching config) | 🔲 | Prevents back-to-back fetches on rapid WS pushes |
| Per-component reconcile backoff (exponential on repeated failures, max 10m) | 🔲 | Only failing component backs off; healthy components unaffected |

### Security + Resource Governance

| Item | Status | Notes |
|---|---|---|
| Least-privilege RBAC (namespace-scoped where possible) | 🔲 | Audit ClusterRole; move to Role for namespace-only operations |
| Pod Security Standards (baseline profile minimum, restricted where feasible) | 🔲 | `pod-security.kubernetes.io/enforce: baseline` on intangles namespace |
| PodDisruptionBudget — only created when replicas > 1 | 🔲 | Single-replica components skip PDB; multi-replica get `minAvailable: 1` |
| ResourceQuota + LimitRange for `intangles` namespace | 🔲 | Prevent resource exhaustion on shared clusters |
| NetworkPolicies — namespace isolation with targeted allow rules | 🔲 | See below |

#### NetworkPolicy Scope

Avoid aggressive default-deny egress that breaks K8s cluster operations. Apply namespace isolation with explicit allow rules only for known traffic patterns:

```
Ingress rules:
  - Allow: service ↔ infra within namespace (app ports)
  - Allow: operator → all pods within namespace (management)
  - Allow: infra management ports within namespace (RMQ 15672, MongoDB 27017, Redis 6379)

Egress rules:
  - Allow: kube-dns (UDP/TCP 53) — required for K8s DNS resolution
  - Allow: K8s API server — required for operator + service account operations
  - Allow: operator → CP (HTTPS 443) — heartbeat, config sync, WS
  - Allow: operator → image registry (HTTPS 443) — image pulls
  - Allow: infra ↔ infra within namespace — replication, clustering
  - Allow: services → infra within namespace — DB/queue connections
  - Default deny: everything else (cross-namespace, arbitrary internet)
```

Start with ingress isolation only if egress deny causes issues in a specific customer environment. Egress rules can be added incrementally.

### Credential Rotation

Lightweight rotation for auto-generated infra credentials (MongoDB, Redis, RMQ passwords stored in `intangles-credentials` Secret). Triggered by `rotate_credentials` WS command or manually via CP dashboard.

```
CP sends rotate_credentials command (or operator triggers on schedule)
         │
         v
For each specified credential key:
  1. Generate new random value
  2. Update K8s Secret (intangles-credentials)
  3. Helm upgrade affected infra component with new credential
  4. Wait for infra component to return Ready
  5. Rebuild intangles-config ConfigMap (new connection strings)
  6. Rolling restart all deployment components that dependsOn this infra
         │
         v
Update ComponentStatus for each affected component
Heartbeat reports rotation complete
```

| Item | Status | Notes |
|---|---|---|
| `rotate_credentials` WS command handler | 🔲 | Triggers rotation for specified component(s) |
| Per-credential rotation (update Secret → Helm upgrade → ConfigMap rebuild) | 🔲 | Controlled sequence; not all-at-once |
| Rolling restart of dependent deployments after credential rotation | 🔲 | Uses `dependsOn` graph to identify affected services |
| Rotation status surfaced in ComponentStatus + heartbeat | 🔲 | Dashboard shows rotation in progress |

### Lightweight Observability (Operator Metrics)

Minimal Prometheus metrics exposed on the operator pod's `/metrics` endpoint. No Prometheus deployment required — customer's existing stack can scrape if they want. Operator always exposes; scraping is opt-in.

| Item | Status | Notes |
|---|---|---|
| `reconcile_duration_seconds` histogram (per platform) | 🔲 | How long each reconcile pass takes |
| `reconcile_failures_total` counter (per platform, per reason) | 🔲 | Tracks error rate in reconciler |
| `component_health` gauge (per component: 0=failed, 1=degraded, 2=ready) | 🔲 | Current health state as a metric |
| `operator_ws_reconnects_total` counter | 🔲 | WS connection stability signal |
| Operator `/metrics` HTTP endpoint (Prometheus format) | 🔲 | Port 8383; no auth needed (internal only) |

> Full Prometheus stack deployment, Grafana dashboards, alerting rules, and SLO definitions remain deferred until customer demand justifies them.

### Phase 4 — Dashboard Changes

| Item | Status | Notes |
|---|---|---|
| CustomerDetail: FSM conditions panel in Overview tab | 🔲 | Show `status.conditions[]` as a timeline — type, reason, message, lastTransitionTime. Helps ops understand why platform is stuck |
| CustomerDetail Operator tab: "Rotate Credentials" button | 🔲 | Sends `rotate_credentials` command; shows rotation in-progress state on Infra tab |
| CustomerDetail Infra tab: show rotation in-progress status per component | 🔲 | `ComponentStatus` carries rotation state; highlight affected components |
| CustomerDetail Operator tab: link to operator `/metrics` endpoint | 🔲 | Read-only URL display with copy button; useful for customer's Prometheus scrape config |
| CustomerDetail: Helm timeout override input per infra component | 🔲 | Exposed via component overrides — input in ServiceOverridesModal or inline on Infra tab |
| CustomerDetail Infra tab: show Helm retry count + "max retries reached" state | 🔲 | When operator stops retrying, show clear error state with manual retry button |

---

## Phase 5 — Observability

**Goal:** Full visibility into customer platform health without direct access.

| Item | Status | Notes |
|---|---|---|
| Pod health collector (restarts, OOMKills, ready state, CPU/mem) | 🔲 | Via K8s metrics API |
| RabbitMQ collector (queue depth, DLQ, consumers, redelivery rate) | 🔲 | Via RMQ management API |
| MongoDB collector (replication lag, connections, disk usage) | 🔲 | Via MongoDB driver |
| Redis collector (memory, evictions, hit ratio, clients) | 🔲 | Via redis INFO command |
| Alert engine (configurable thresholds, severity: info/warning/critical) | 🔲 | |
| Telemetry reporter (sends metrics to CP, never customer data) | 🔲 | POST /telemetry/ingest |
| CP dashboard: health view (alerts, per-component drill-down) | 🔲 | |
| Diagnostic collection command (`collect_diagnostics`) | 🔲 | Last N log lines on demand |
| Live component status visible in customer detail | ✅ | From heartbeat via `operator_status` |

### Phase 5 — Dashboard Changes

New "Health" tab on CustomerDetail. Replaces the current "live from heartbeat" text-only status with actual metric views and alert surfacing.

| Item | Status | Notes |
|---|---|---|
| CustomerDetail: new "Health" tab | 🔲 | Sits between Infrastructure and Operator tabs |
| Health tab: RabbitMQ panel — queue depth, DLQ count, consumer count, redelivery rate per queue | 🔲 | Table or card per queue; color-coded thresholds |
| Health tab: MongoDB panel — replication lag, connections, disk usage | 🔲 | Key metrics with threshold indicators |
| Health tab: Redis panel — memory %, evictions, hit ratio | 🔲 | |
| Health tab: active alerts list — severity badge, metric name, current value vs threshold, first-seen time | 🔲 | Sorted by severity; critical at top |
| Health tab: "Collect Diagnostics" button → sends `collect_diagnostics` command | 🔲 | Shows returned log snippet inline |
| CustomerList: health score or alert count visible per customer row | 🔲 | e.g. "🔴 1 critical" or "🟡 2 warnings" as a column |
| CustomerDetail Overview stat bar: replace License card with Health Score card (Phase 5) | 🔲 | License moves to Phase 8; health score more immediately useful |

---

## Phase 6 — Code Protection + Build Pipeline

**Goal:** Services cannot be reverse-engineered by customers.

| Item | Status | Notes |
|---|---|---|
| JavaScript obfuscation pipeline (intangles, data_apis, berries) | 🔲 | `javascript-obfuscator`: string encrypt + var rename + control flow flatten |
| CI: verify obfuscated image works before push | 🔲 | Dry-run each APP_TYPE after obfuscation |
| cosign image signing + operator signature verification | 🔲 | Reject unsigned images |
| Private ECR registry setup (per-customer pull credentials) | 🔲 | |
| `intangles-db-init` image (separate from backend) | 🔲 | Seeds collections, RMQ topology, admin user |

### Phase 6 — Dashboard Changes

> No dashboard changes required for Phase 6. Code protection and build pipeline are CI/ops concerns only.

---

## Phase 7 — Hardening + Production Scale

**Goal:** Ready for multiple real customers.

| Item | Status | Notes |
|---|---|---|
| Air-gapped support (offline image mirror workflow) | 🔲 | Customer-hosted registry; operator reads registry override from CRD |
| Agent token rotation (CP pushes new token via `reauth` command) | 🔲 | `reauth` command already wired; CP-side rotation flow needed |
| Customer resource sizing from CP config (vehicleCount → replica counts) | 🔲 | CP embeds replica recommendations per component based on vehicleCount |
| Multi-customer stress test (5+ agents vs CP simultaneously) | 🔲 | Load test before first real customer |
| Documentation: customer onboarding guide + ops runbook | 🔲 | |

### Auto-Scaling (HPA)

Initial HPA targets CPU + memory (available via standard K8s metrics server). HPA creation must respect the component dependency graph — a scaleout of `parser` increases RabbitMQ consumption load, which in turn requires `datastore` to scale to keep up. Scaling one component without awareness of its downstream dependents can cause queue backup cascades (parser scales up → datastore queue fills → RMQ memory pressure → parser throttled). The operator tracks dependency ordering and can optionally scale dependent consumers together.

| Item | Status | Notes |
|---|---|---|
| HPA for `type:deployment` components — CPU + memory triggers | 🔲 | Operator creates HPA when `replicas: auto` in spec |
| CP config: `minReplicas` / `maxReplicas` per component | 🔲 | CP sets bounds; operator creates HPA within them |
| Dependency-aware scale coordination (scale consumers with producers) | 🔲 | Optional; controlled via CP config flag per component |
| Queue-depth HPA (future) | ⛔ | Requires KEDA or custom metrics adapter; defer until a customer needs it |

### Backup Automation

MongoDB is the primary backup target — it holds all persistent business data. RabbitMQ topology can be re-seeded from db-init; definitions export is a safety net. Redis is ephemeral (cache + session); backup is optional.

| Item | Status | Notes |
|---|---|---|
| MongoDB backup CronJob (mongodump → S3/MinIO/GCS) | 🔲 | Operator creates CronJob; schedule + target from CP config |
| RabbitMQ definitions export (exchanges/queues/bindings snapshot) | 🔲 | Via management API; much lighter than full backup |
| Redis backup | ⛔ | Redis is a cache; backup only if customer stores non-reproducible data in it |
| Backup monitoring (last successful run, size, age) | 🔲 | Surfaced in heartbeat status |

### Capability Negotiation (future-safe, lightweight)

Operator advertises its supported capabilities to CP on connect. CP uses this to avoid sending config fields the operator version doesn't understand — enabling rolling upgrades across a mixed fleet without breaking old operators.

| Item | Status | Notes |
|---|---|---|
| Operator advertises `capabilities[]` in heartbeat (e.g. `["components/v2","hpa","pdb"]`) | 🔲 | Simple string list in heartbeat payload |
| CP skips unknown fields when targeting older operator versions | 🔲 | CP-side: check capabilities before emitting new config shape |
| `minOperatorVersion` field in CP config response | 🔲 | Operator logs warning if below minimum; no hard block initially |

### Dashboard Changes

| Item | Notes |
|---|---|
| HPA config in Catalog component form: `Replicas` field accepts integer or `auto`; when `auto` is selected, show `Min` / `Max` inputs | Catalog modal — ServiceSpec fields; validate min ≤ max |
| HPA live status in Services tab: when a component has `replicas: auto`, show `N / min–max` (current / bounds) instead of `N / desired` | Read from `componentStatuses[].replicas` + HPA fields in status |
| Backup config in Infra tab per backup-related component (MongoDB CronJob): show schedule, last run timestamp, last run result (success / failed) | Sourced from `componentStatuses[]` for the backup CronJob component |
| Operator tab — new "Capabilities" read-only row: shows `capabilities[]` advertised by connected operator (e.g. `components/v2`, `hpa`, `pdb`) | Useful for ops team to confirm operator version compatibility |
| Operator tab — show `minOperatorVersion` warning banner if the running operator reports a version below the CP-required minimum | Pull from last heartbeat; display inline in Operator tab header |
| Air-gapped registry: Operator tab — "Registry Prefix" and "Pull Secret" inputs already exist; add tooltip: "Use a customer-hosted mirror for air-gapped clusters" | No new field needed; documentation/tooltip only |

---

## Phase 8 — Licensing

**Goal:** Enforce customer licenses, vehicle limits, and time-based expiry. Phases 1–7 use a stub that always returns true.

| Item | Status | Notes |
|---|---|---|
| RSA keypair generation (control plane holds private key) | 🔲 | `openssl genrsa -out private.pem 2048` |
| JWT license issuance at customer onboarding (agentID + vehicleLimit + expiry) | 🔲 | CP signs with private key |
| Operator: JWT signature verification (RSA public key embedded in binary) | 🔲 | Replaces current stub |
| Operator: local license cache (7-day offline grace) | 🔲 | Cached to disk; re-verified on startup |
| Operator: vehicle limit enforcement (pause if exceeded) | 🔲 | Via CP heartbeat response |
| Operator: license expiry enforcement (scale to 0, enter dormant) | 🔲 | |
| Cluster fingerprint (license tied to K8s cluster UID) | 🔲 | Prevents license copying to another cluster |
| CP: license revocation (instant via WS kill + license blacklist) | 🔲 | |
| Air-gapped license (extended grace period, no CP reachability needed) | 🔲 | Long-lived JWT for offline clusters |
| CP dashboard: license status, expiry, vehicle count usage | 🔲 | |

### Dashboard Changes

| Item | Notes |
|---|---|
| CustomerDetail status bar — wire existing `License` stat card to real data: show `Valid until <date>` or `Expired <date ago>` with green/red color | Currently shows `licenseValid: true/false`; replace with expiry date from status |
| License stat card subtitle: show vehicle usage vs limit (e.g. `840 / 1000 vehicles`) | Pull from `status.vehicleCount` + `spec.vehicleLimit` |
| License `Revoked` state: distinct red badge (separate from `Expired`) — shown when CP has blacklisted this license | Add `licenseRevoked` field to status; display as `⊘ Revoked` |
| License expiry warning: amber badge when expiry is within 30 days (`⚠ Expires in N days`) | Client-side: compute days from expiry timestamp |
| CustomerList — add license expiry as a sortable column (date or `⚠ soon` / `⊘ revoked`) | Optional; useful when managing a fleet of customers |
| CustomerDetail Operator tab — "Regenerate License" button (visible only to CP admins) when license is expired or revoked | Calls CP endpoint to re-issue license JWT; sends `reauth`-style command to operator |

---

## Control Plane Dashboard

The CP dashboard is an **internal ops tool** — used by the Intangles team, not by customers. Operational clarity, fast access to runtime state, and reliable command execution matter more than visual polish. The current design is deliberately minimal; improvements should make it more robust and operationally capable, not more complex.

### Current Architecture

| Concern | Implementation |
|---|---|
| Framework | React 18, plain JSX (no TypeScript) |
| Bundler | Vite |
| Routing | None — page state via `useState` + `localStorage` |
| State management | None — all local component state |
| Server state / polling | Manual `useEffect` loops in each page; fingerprint dedup in CustomerDetail |
| Styling | Custom CSS-in-JS inline styles + one global `index.css`; fixed dark theme |
| Component library | None — all primitives duplicated per-page as inline style objects |

Polling logic is duplicated across `CustomerList`, `CustomerDetail`, and individual tabs. Mutations trigger manual `load()` / `setTimeout` calls for re-fetch. Adaptive polling intervals (5s during infra operations, configurable otherwise) live in page-level state.

---

### Dashboard Design Principles

1. **Operational clarity first.** Every UI decision should make runtime state more readable and ops actions more reliable. Aesthetics are secondary.
2. **Shared infrastructure, not shared pixels.** Extract query/cache/mutation logic into a common layer. Keep UI components local unless duplication across three or more pages justifies extraction.
3. **Status-driven, not page-driven.** As the backend evolves toward the unified component model, the dashboard reflects component status and lifecycle transitions — not just a list of flat pages.
4. **No abstraction without operational justification.** Don't introduce a new layer unless it directly removes friction from an ops workflow or prevents a category of bugs.

---

### FE Phase 1 — Infrastructure Foundation

**Goal:** Replace scattered polling/mutation/state logic with shared infrastructure. No visible changes to the UI — this is a refactor that makes every subsequent phase easier and removes a class of bugs (stale data, race conditions, duplicate fetches).

#### Routing — React Router v6

Replace `useState` + `localStorage` page tracking with URL-based routing. Browser back/forward works correctly. Direct customer links work (useful when handing off an incident).

```
/customers              → CustomerList
/customers/:id          → CustomerDetail
/releases               → Releases
/catalog                → Catalog
/settings               → Settings
```

`localStorage` restore of last-viewed customer becomes a redirect: if `cp_customer_id` is set, navigate to `/customers/:id` on mount.

#### State Management — Zustand

Zustand for global app state: auth token, active customer reference, sidebar UI state. Replaces scattered `useState` in `App.jsx` and prop-drilling through the component tree. No reducers, no action creators — plain `set()` calls.

```js
// Entire global store — ~30 LOC
const useStore = create((set) => ({
  token:    localStorage.getItem("cp_admin_token") || "",
  customer: null,
  setToken:    (t) => set({ token: t }),
  setCustomer: (c) => set({ customer: c }),
}));
```

#### Server State — TanStack Query (React Query v5)

All API calls — customer list, customer detail, catalog, releases — move into TanStack Query. This consolidates:

- Background polling → `refetchInterval` (static or dynamic callback)
- Fingerprint dedup → stale-time config (skip re-render if data unchanged)
- Retry logic → query client defaults
- Post-mutation re-fetch → `queryClient.invalidateQueries()` replaces all manual `load()` + `setTimeout` calls
- Adaptive polling (5s during infra operations) → `refetchInterval: (data) => data?.phase === "Provisioning" ? 5000 : pollMs`

| Item | Status | Notes |
|---|---|---|
| React Router v6 — replace localStorage page state with URL routing | 🔲 | `src/router.jsx`; customer detail at `/customers/:id` |
| Zustand store — auth token + active customer + global UI state | 🔲 | Replaces App.jsx useState + prop-drilling; `src/store.js` |
| TanStack Query setup (queryClient with default stale-time + retry config) | 🔲 | `src/lib/queryClient.js` |
| Migrate `CustomerList` polling to `useQuery` with `refetchInterval` | 🔲 | Remove manual `useEffect` polling loop |
| Migrate `CustomerDetail` status polling to `useQuery` with adaptive interval | 🔲 | Dynamic `refetchInterval` callback replaces in-component logic |
| Migrate all mutations (kill, reactivate, restart, scale, etc.) to `useMutation` | 🔲 | `onSuccess: () => queryClient.invalidateQueries(...)` |
| Remove all manual `useEffect` polling loops post-migration | 🔲 | Cleanup pass after all queries migrated |

---

### FE Phase 2 — Reusable UI Primitives

**Goal:** Extract the patterns that are copy-pasted across every page into a small set of shared components. Reduces inline-style duplication without changing the visual design or switching to an external component library. Each primitive is ~30–80 LOC, uses the existing color tokens, and has no external dependencies.

#### Components to Extract

| Component | What It Replaces | Location |
|---|---|---|
| `Button` | Inline `style` button objects with variant/size/loading/disabled permutations | `src/components/ui/Button.jsx` |
| `Badge` | Inline badge divs with status colors — currently using `.badge` CSS class inconsistently | `src/components/ui/Badge.jsx` |
| `Modal` | Duplicated modal overlay + close button + click-outside handler across 10+ modals | `src/components/ui/Modal.jsx` |
| `Card` | Inline `.card` class divs with optional header and action slots | `src/components/ui/Card.jsx` |
| `Table` | Repeated table structure (full-width, uppercase headers, hover rows) | `src/components/ui/Table.jsx` |
| `StatusDot` | Colored dot + label (connected / reconnecting / offline / killed) — duplicated in CustomerList + CustomerDetail header | `src/components/ui/StatusDot.jsx` |
| `EmptyState` | Ad-hoc empty state text per page — no consistent treatment | `src/components/ui/EmptyState.jsx` |
| `InfoPair` | Label + value row — already exists inline in CustomerDetail; needs extraction | `src/components/ui/InfoPair.jsx` |
| `Flash` | Two separate implementations (CustomerDetail vs CustomerList) — consolidate | `src/components/ui/Flash.jsx` |

`Modal` handles focus-trap, `Esc` to close, and focus-return to trigger button — making accessibility improvements in FE-4 easier to apply uniformly.

| Item | Status | Notes |
|---|---|---|
| `Button` component (primary / danger / secondary / ghost; sm/md; loading; disabled) | 🔲 | |
| `Badge` component (green / yellow / red / gray variants) | 🔲 | |
| `Modal` component (overlay, close, click-outside, keyboard close) | 🔲 | Focus-trap built in for FE-4 |
| `Card` component (header + actions slots) | 🔲 | |
| `Table` helpers (`Thead`, `Tr`, `Td` — hover + alignment) | 🔲 | |
| `StatusDot` component (connection + phase state) | 🔲 | |
| `EmptyState` component (icon-optional, message, action button slot) | 🔲 | |
| `InfoPair` component (label + value, monospace option) | 🔲 | |
| `Flash` component (unified — success/error, auto-dismiss, click-dismiss) | 🔲 | Replaces two current implementations |
| Migrate pages to use extracted primitives — no visual change | 🔲 | Incremental; page by page |

---

### FE Phase 3 — Component-Centric Architecture

**Goal:** Align the dashboard's data model with the backend's unified component model (backend Phase 3b/3c). Status rendering becomes shared across all component types instead of being split across Services tab (deployments) and Infrastructure tab (Helm releases).

> **Dependency:** Do not start until backend Phase 3b ships `componentStatuses[]` in the heartbeat. The migration shim for old agents (which still send `services[]` + `infraComponents[]`) is handled in backend Phase 3b; the dashboard must respect it.

#### Component Status Rendering

A single `ComponentRow` renders any `ComponentStatus` record regardless of type (helm / deployment / job). The row adapts its action buttons and expand content based on `type` — replacing the current separate `ServiceRow` and infra row implementations.

Phase-to-UI mapping (shared constants, consistent across all pages):

| Phase | Color | Icon | Label |
|---|---|---|---|
| Ready | Green `#48bb78` | ● | Ready |
| Degraded | Yellow `#f6c90e` | ⚠ | Degraded |
| Failed | Red `#fc8181` | ✕ | Failed |
| Deploying | Blue `#4f6ef7` | ↻ | Deploying |
| Pending | Muted `#718096` | ◌ | Pending |
| Disabled | Muted `#4a5568` | ⊘ | Disabled |

`Pending` shows "Waiting for: mongodb, rabbitmq" below the component name — derived from `dependsOn` in `componentStatuses[]`. This is text, not a graph visualization. No canvas or D3 needed.

`ComponentStatusList` renders the full list sorted by severity: Failed → Degraded → Deploying → Pending → Ready → Disabled. Highest-attention items always surface to the top.

| Item | Status | Notes |
|---|---|---|
| Phase color/icon/label constants (`src/lib/componentPhase.js`) | 🔲 | Single truth for phase → color mapping |
| `ComponentRow` — shared row for any component type + phase | 🔲 | `src/components/platform/ComponentRow.jsx`; replaces ServiceRow + infra row |
| `ComponentStatusList` — sorted list with phase-order | 🔲 | `src/components/platform/ComponentStatusList.jsx` |
| `DependencyBadge` — tooltip listing `dependsOn` + upstream phase | 🔲 | Inline tooltip; no third-party tooltip library; amber when dependency is Degraded |
| CustomerDetail Services + Infrastructure tabs → `ComponentStatusList` (filtered by type) | 🔲 | After backend Phase 3b ships |
| CustomerDetail: "Jobs" section in Services tab for `type:job` components | 🔲 | db-init + future jobs; shows lifecycle state (runOnce: Completed / Failed) |
| CustomerDetail stat bar: "Services N/M" derives from `componentStatuses[]` | 🔲 | Fallback to old shape if `componentStatuses` absent |
| CustomerList: per-customer component counts from unified status | 🔲 | |

---

### FE Phase 4 — Operational Improvements

**Goal:** Practical improvements for day-to-day ops. Each item must have direct operational value — it either reduces time-to-diagnose or prevents a class of ops error.

#### Search + Filtering

Client-side — no server round-trips. Filters the currently-loaded list.

- CustomerList: text search by company name or agent ID; status filter pills (All / Connected / Offline / Degraded)
- Catalog: component name search (services tab + infra tab)
- Pack management: component name filter within the pack expand view

#### Command Queue Lifecycle

Current state: command queue shows type + timestamp + payload. No visibility into whether the command was received, executed, or failed.

New per-entry lifecycle:

```
Created (in queue) → Sent (WS delivered) → Acked (operator confirmed) → Done
                                         ↘ Failed (with reason message) → Retry button
```

Entries auto-collapse when Done. Failed entries stay visible with red border + reason text + manual retry button. The server must track `sentAt`, `ackedAt`, `failedReason` per command entry.

#### Global Connection Indicator

Sidebar bottom section: single indicator showing the dashboard's WS connection to CP (Connected / Reconnecting / Disconnected). Currently there is no way to know if the WS dropped — the dashboard silently polls HTTP while the WS is down, and operators don't know commands won't be delivered.

#### Activity / Audit Timeline

Lightweight per-customer activity feed showing the last 50 CP-originated actions: kill, reactivate, agent_update, rotate_credentials, config_change push, etc. Stored in MongoDB per customer. Surfaced in CustomerDetail Operator tab as a collapsible timeline. Rolling 50-event window — no pagination.

This is not an audit log for compliance — it's an ops tool for answering "who ran what and when did the platform change state."

#### Accessibility Baseline

Pragmatic improvements to make the dashboard keyboard-navigable for the ops team. Not a full WCAG audit.

- All modals: focus-trap (from FE-2 `Modal`), Esc to close, focus returns to trigger button on close
- All buttons: `type="button"` (prevent accidental form submit)
- All icon-only buttons: `aria-label`
- CustomerList: arrow keys to move between rows, Enter to open customer detail
- Status indicators that use color only: add text label (most already have one — audit for gaps)

| Item | Status | Notes |
|---|---|---|
| CustomerList: client-side search (name + agent ID) | 🔲 | Filters rendered list; debounce 200ms |
| CustomerList: status filter pills (All / Connected / Offline / Degraded) | 🔲 | |
| Catalog: component name search (services + infra tabs) | 🔲 | |
| Pack management: component name filter in pack expand | 🔲 | |
| Command queue: full lifecycle status (Created → Sent → Acked → Failed) | 🔲 | Requires server-side `sentAt` / `ackedAt` / `failedReason` fields on command entries |
| Command queue: failed command retry button + reason display | 🔲 | |
| Global WS connection indicator in sidebar | 🔲 | Shows CP-to-dashboard WebSocket health; not per-agent |
| Per-customer activity timeline (last 50 CP actions) | 🔲 | `GET /customers/:id/activity`; stored in MongoDB; Operator tab collapsible section |
| Modal focus-trap + Esc close + focus restore (from FE-2 Modal) | 🔲 | Covered by FE-2; list here for tracking |
| Icon-only buttons: `aria-label` audit + fix | 🔲 | |
| CustomerList keyboard navigation (arrow keys + Enter) | 🔲 | |

---

### Dashboard Not Planned

Do not implement unless explicitly instructed.

| Item | Why |
|---|---|
| Redux | Zustand (FE-1) covers all global state needs with far less boilerplate |
| Tailwind CSS migration | Custom CSS-in-JS design system works; migration has zero operational value |
| Material UI / Ant Design / Chakra / shadcn | External component library is unnecessary overhead for an internal ops tool |
| GraphQL or WebSocket subscriptions for data | REST + WebSocket heartbeat model is sufficient |
| Dark/light mode toggle | Dark theme is fixed and intentional for this tool |
| Recharts / ECharts / chart visualization library | Phase 5 health metrics are key numbers + threshold indicators, not charts |
| Onboarding flows / guided tours | Not a customer-facing product |
| Drag-and-drop (pack ordering, component sorting) | `dependsOn` graph is explicit; visual drag ordering adds no operational value |
| Mobile / responsive design | Internal ops tool; desktop-only is correct |
| Microfrontend architecture | Single Vite app; no team-scale problem that would justify the overhead |
| Advanced analytics / reporting dashboard | Metrics telemetry (Phase 5) feeds a simple health view, not a reporting product |
| i18n / localization | Internal English-only tool |
| TypeScript migration | Adds correctness but also migration overhead; not worth it until the team has TypeScript expertise across the board |

---

## Future Evolution Notes

These are architectural directions worth keeping in mind as the platform matures — not on the roadmap now, but the current design doesn't block them.

**Component lifecycle events / internal state bus:** As the number of component types and lifecycle transitions grows, a lightweight internal event model (e.g. `ComponentTransitioned`, `HealthCheckFailed`, `DependencyReady`) would decouple handlers from the reconciler and make retries + orchestration logic testable in isolation. Not needed until the reconciler becomes hard to follow (~1000+ LOC or 4+ component types with cross-cutting concerns). Prefer a simple in-process event channel over a full event-driven architecture when the time comes.

**Controller decomposition:** The unified reconciler may eventually be split into focused sub-controllers (infra, service, job, health, upgrade) to improve testability and parallel reconciliation. Defer until operational complexity actually requires it — premature decomposition adds coordination overhead without benefit.

**Canary deployments:** The `version` field on `ComponentSpec` (Phase 3a) is the prerequisite. When canary is needed, the operator can deploy a second Deployment with the new version alongside the existing one and gradually shift traffic. Not planned until a customer requires it.

---

## Not Planned (Do Not Implement Until Explicitly Instructed)

| Item | Why Deferred |
|---|---|
| Full Prometheus stack deployment (Prometheus server, Grafana, alerting rules) | Lightweight operator metrics endpoint (Phase 4) is sufficient. Full stack is customer infra |
| Auto-remediation framework with cooldowns | Phase 5 observability + manual commands cover this. Automated remediation adds risk |
| Multi-cluster per customer (dev/staging/prod aggregated dashboard) | Separate agent IDs per env works today. Aggregated view is a dashboard feature |
| GitOps compatibility (ArgoCD/Flux) | Operator IS the gitops layer. Customer doesn't manage CRD content — CP does |
| Policy engine extensibility (OPA/Kyverno) | Add if a specific customer mandates it |
| Grouped runtime images (parser/APIs/analytics separate) | APP_TYPE + single image is simpler. Split only if services need conflicting runtimes |
| Vault / External Secrets integration | K8s Secrets are sufficient for on-prem. Add only if a customer mandates it |
| SBOMs, admission policies, runtime security (Falco) | Add for enterprise customers with formal security requirements |
| Disaster recovery orchestration | MongoDB backup (Phase 7) is the foundation. Full DR is a later topic |
| Full TLS automation (cert-manager for all internal services) | Internal cluster traffic isolated by NetworkPolicies. Add if customer requires mTLS everywhere |
| Formal SLO/SLA framework | Define after first real customer's operational data |
| Stateful upgrade orchestration (drain → upgrade → restore per DB node) | Rolling updates cover application services. Needed only for major infra schema migrations |

---

## New Region Deployment

The same platform and operator are used to deploy a new region:

1. Add the region's CP URL (new CP instance or multi-tenant existing one)
2. Customer onboarding flow identical — `setup.sh` + agent credentials from CP
3. Images served from region-local registry (ECR in that region) to minimize latency
4. No operator changes required — region is another agent connecting to CP

---

## Key Decisions

| Decision | Rationale |
|---|---|
| Node.js API for CP (not Go) | Already built, team knows it, fast iteration |
| MongoDB for CP (not PostgreSQL) | Consistent with existing platform stack |
| Unified component model (Phase 3) | Single lifecycle path for all deployable units; dependency graph replaces hardcoded ordering |
| Component+pack model (not features map) | Composable, extensible, no hardcoded feature→service mappings in operator |
| CP defines desired state; operator owns orchestration | CP is the source of truth for what should run. Operator decides how, when, and in what order to get there |
| APP_TYPE as service discriminator | Zero changes to existing service code |
| DependsOn graph (not hardcoded infra→services phases) | Explicit, testable, extensible to any component ordering |
| WS push + polling fallback for config delivery | WS gives immediate propagation; polling ensures reliability at startup and after reconnects |
| Component ownership labels | Prevents cross-component resource conflicts and accidental cleanup |
| Licensing last (Phase 8) | Zero value until first paying customer. Stub returns true; all other value delivered first |
| Obfuscation (not binary/bytecode) | Bytecode breaks dynamic `require()`, proto files, native addons |
| Single cluster per customer | Simplest operational model. Multi-cluster is a dashboard concern, not operator concern |
| React Router (not custom nav) | URL-based routing gives browser history + direct links at zero cost; the custom useState nav can't be linked |
| TanStack Query (not SWR or Redux RTK Query) | Best-in-class dedup, adaptive intervals, mutation invalidation, devtools — solves the exact polling/retry problem with minimal setup |
| Zustand (not Context, Redux, or Jotai) | Fits scope: ~30 LOC store, no reducers, no providers, no boilerplate |
| Custom UI primitives (not external component library) | Design system is intentional; premature standardization on an external library would make it harder to match the existing dark-theme design |
| Component-centric dashboard (FE-3) mirrors backend unified model | Shared `ComponentRow` replacing split ServiceRow + infra row reduces code duplication and stays in sync as backend adds types |
