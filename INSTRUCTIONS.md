# INSTRUCTIONS.md

# Intangles Platform — Repository Implementation Rules

This document defines repository structure, implementation expectations, development workflow, testing requirements, architecture rules, and operational standards for the Intangles platform.

It is the primary implementation contract for contributors and Claude Code.

---

# Repository Structure

```text
/
├── intangles-operator/              # Go Kubernetes operator
│
├── intangles-cp/
│   ├── intangles-cp-server/         # Node.js control plane backend
│   └── intangles-cp-dashboard/      # React/Vite control plane frontend
│
├── docs/                            # Shared architecture + operational docs
├── scripts/                         # Shared scripts
├── deploy/                          # Shared deployment manifests
└── README.md
```

---

# Repository Responsibilities

## intangles-operator/

Responsible for:
- Kubernetes reconciliation
- Unified component graph
- Dependency-aware orchestration
- Helm lifecycle management
- Semantic health evaluation
- Config synchronization
- WebSocket command handling
- Runtime orchestration
- Telemetry + heartbeat
- Licensing enforcement
- Offline/airgap behavior

Technology:
- Go
- Kubebuilder
- controller-runtime
- Helm SDK

---

## intangles-cp-server/

Responsible for:
- Customer management
- Pack/component catalog
- Desired-state generation
- Agent communication
- WebSocket hub
- Release management
- Telemetry ingestion
- Licensing
- Audit/event storage
- Dashboard APIs

Technology:
- Node.js
- Fastify
- MongoDB

---

## intangles-cp-dashboard/

Responsible for:
- Operational dashboard
- Customer management UI
- Component visibility
- Pack management
- Release management
- Command controls
- Health visualization
- Audit visibility

Technology:
- React
- Vite
- Zustand
- React Query

---

# Required Repository Files

Every major repo/folder must contain:

```text
README.md
CLAUDE.md
```

Required locations:

```text
/intangles-operator/README.md
/intangles-operator/CLAUDE.md

/intangles-cp/intangles-cp-server/README.md
/intangles-cp/intangles-cp-server/CLAUDE.md

/intangles-cp/intangles-cp-dashboard/README.md
/intangles-cp/intangles-cp-dashboard/CLAUDE.md
```

---

# README.md Requirements

Every README.md must contain:
- Architecture overview
- Folder structure
- Local development setup
- Build instructions
- Deployment instructions
- Kubernetes deployment steps
- Helm usage
- Environment variables
- Testing instructions
- Troubleshooting
- Operational commands
- Release/update workflow
- Rollback instructions
- Examples
- Known limitations
- Current phase status
- Dependencies/services used
- Common development workflows

README files must be updated whenever:
- architecture changes
- deployment changes
- commands change
- new dependencies added
- new phases implemented

No feature is complete without README updates.

---

# CLAUDE.md Requirements

Each project-specific CLAUDE.md must contain:
- Project architecture
- Important implementation rules
- Coding standards
- Local development commands
- Test commands
- Important constraints
- Ownership boundaries
- Integration expectations
- Anti-patterns to avoid
- Operational philosophy
- Migration expectations
- Folder responsibilities

CLAUDE.md files should optimize Claude Code behavior and reduce repeated prompting.

---

# Development Lifecycle

Every implementation must follow:

```text
Plan
→ Write
→ Test
→ Validate
→ Document
→ Refactor
→ Merge
```

Rules:
- No direct implementation without planning
- No merge without tests
- No merge without documentation updates
- No skipped validation
- Refactors must preserve backward compatibility unless explicitly approved

---

# Testing Requirements

Unit tests are mandatory for:

## Operator
- Reconcilers
- Handlers
- Dependency graph logic
- Health evaluators
- Retry/backoff logic
- Config parsers
- WS handlers
- Status evaluators
- Lifecycle transitions

## CP Server
- APIs
- Services
- WS hub
- Config generators
- Licensing
- Validation logic
- Audit/event systems

## Dashboard
- Components
- Hooks
- Zustand stores
- React Query logic
- UI utilities
- State transitions
- Polling/query behavior

---

# Test Expectations

Required:
- Unit tests
- Integration tests
- E2E tests for critical flows

Critical flows:
- Customer onboarding
- Config sync
- Dependency ordering
- Infra deployment
- Component lifecycle
- WS commands
- Rollback/update
- Offline recovery
- Licensing
- Backup/restore

Implementation is incomplete without tests.

---

# Implementation Order

Implementation must follow the master plan phases incrementally.

Do NOT skip foundational phases.

Order:
1. Core operator + CP
2. Remote control + updates
3. Unified component model
4. Production hardening
5. Observability
6. Code protection/build pipeline
7. Hardening + production scale
8. Licensing

Frontend phases must align with backend evolution.

---

# Engineering Principles

## Core Rules

- Kubernetes-native patterns first
- Backward compatibility by default
- Simplicity over enterprise complexity
- Avoid unnecessary abstractions
- Avoid premature optimization
- Components are the single deployment abstraction
- CP owns desired state
- Operator owns orchestration/runtime behavior
- APP_TYPE compatibility preserved unless explicitly migrated
- Prefer additive migrations
- Avoid rewriting stable systems unnecessarily

---

# Operator Engineering Rules

Required:
- Dependency-aware reconciliation
- Idempotent reconciliation
- Semantic health evaluation
- Retry with backoff/jitter
- Reconcile backpressure protection
- Safe CRD migrations
- Capability negotiation support
- Component ownership boundaries
- Graceful shutdown handling
- Config checksum rollout behavior

Avoid:
- Giant God reconcilers
- Hardcoded ordering
- Duplicate deployment logic
- Direct infra/service split logic
- Infinite retries
- Blocking reconciliation loops

---

# Frontend Engineering Rules

Required:
- Lightweight architecture
- Zustand for global state
- React Query for server-state/polling
- Reusable UI primitives
- Component-centric architecture
- Operational UX over marketing UI
- Shared query/cache infrastructure
- Lightweight accessibility support

Avoid:
- Redux
- Material UI
- Tailwind migration
- Heavy charting systems
- GraphQL
- Microfrontends
- Excessive global state
- Complex design systems

---

# Backend Engineering Rules

Required:
- Modular services
- Clear ownership boundaries
- Config-driven behavior
- Stateless APIs where possible
- Auditability
- Safe retry behavior
- Migration-safe schema evolution

Avoid:
- Hardcoded customer behavior
- Tight coupling to operator internals
- Duplicate config generation logic
- Shared mutable state

---

# CI/CD Requirements

Required pipelines:
- Linting
- Unit tests
- Integration tests
- E2E tests
- Multi-arch builds
- Image verification
- Obfuscation verification
- Release tagging
- Changelog generation

Operator releases must:
- build successfully
- pass tests
- support rollback
- support phased rollout

---

# Documentation Standards

Every major feature must include:
- README updates
- Architecture notes
- Operational notes
- Examples
- Migration instructions
- Known limitations
- Troubleshooting notes

Large architectural changes must be documented in:
```text
/docs/architecture/
```

---

# Forbidden Anti-Patterns

Do NOT introduce:
- Giant monolithic reconcilers
- Hardcoded feature/service mappings
- Duplicate deployment logic
- Uncontrolled polling loops
- Frontend state chaos
- Heavy frameworks without justification
- Breaking changes without migration path
- Undocumented operational behavior
- Hidden retry loops
- Direct customer-specific logic
- Premature microservice decomposition

---

# Master Plan + Implementation Reference

Development MUST use BOTH:
- intangles-onprem-master-plan.md
- intangles-implementation-reference.md

The implementation reference file is mandatory context and acts as:
- implementation truth source
- API contract source
- protocol reference
- schema reference
- existing capability reference
- migration compatibility reference

Do NOT re-implement already completed functionality documented in the implementation reference.

Before implementing any feature:
1. Check current implementation status
2. Check existing API contracts
3. Check existing schema/contracts
4. Preserve backward compatibility
5. Extend existing architecture instead of rewriting stable systems

The implementation reference overrides assumptions.

---

# Change Rules

If architectural, schema, folder, naming, protocol, lifecycle, or implementation changes are required during development:
- update README.md
- update CLAUDE.md
- update architecture docs
- update migration notes
- update implementation phases/status
- update API documentation
- update deployment documentation
- update operational notes

Documentation updates are mandatory whenever behavior changes.

---

# Full Automation Development Mode

Claude Code should operate in full automation development mode.

Expected behavior:
- analyze repository structure first
- identify missing implementations
- identify broken architecture areas
- implement phases incrementally
- automatically create/update tests
- automatically update docs
- automatically refactor when needed
- preserve backward compatibility
- avoid asking repetitive implementation questions unless blocked
- prefer safe incremental migrations
- reuse existing implementations whenever possible

Do NOT:
- rewrite working systems unnecessarily
- introduce large framework migrations
- add enterprise complexity without operational justification
- skip tests/documentation
- break existing APIs/protocols
- ignore implementation-reference contracts

---

# Development Priority

Priority order:
1. Stability
2. Correctness
3. Backward compatibility
4. Observability
5. Maintainability
6. Performance
7. Scalability
8. Enterprise features

Avoid premature optimization.

---

# Required Validation Before Merge

Every completed implementation must validate:
- builds successfully
- tests pass
- docs updated
- migrations documented
- rollback path exists
- backward compatibility preserved
- API compatibility preserved
- operational impact documented

---

# Operational Philosophy

The platform is:
- operational-first
- Kubernetes-native
- component-driven
- control-plane-managed
- startup-pragmatic
- incrementally scalable

Prioritize:
- maintainability
- observability
- operational simplicity
- migration safety
- predictable behavior

Avoid enterprise complexity unless operationally justified.

---

# Final Rule

No implementation is considered complete until:
- tests pass
- documentation updated
- rollback path exists
- operational behavior documented
- backward compatibility validated
- examples included
