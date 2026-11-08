# Agentic URL Shortener — Architecture and Engineering Decisions

## 1. Existing application (brownfield baseline)
The repository contains a React/Vite frontend and a Spring Boot 3.4 / Java 21 backend. The backend already has JWT registration/login, URL mapping CRUD, redirects, click events, and SQL persistence. The agentic layer is added under `com.url.shortner.agent`; existing URL APIs remain under `/api/urls`.

## 2. Components
- **REST/API layer** — `/api/agentic/**` accepts run requests, approvals, rejections, replanning requests, and metrics queries.
- **Orchestrator** — `AgenticWorkflowService` owns run state, DAG dependencies, execution gates, retry policy, safe-stop behavior, and state snapshots.
- **Bounded agents** — requirement, planning, codebase-analysis, design, implementation, testing, documentation, and release-readiness tasks. Current adapters are deterministic demonstration agents; they do not call an external LLM.
- **Policy / human gate** — implementation cannot start until an explicit reviewer approval is recorded. Release readiness never deploys automatically.
- **Audit trail** — every stage transition and approval decision is timestamped with actor/action/details.
- **Metrics** — total/completed/failed runs, success rate, retry and rollback counters, and completion latency.

## 3. Dependency graph and control flow

```mermaid
flowchart TD
  A[Requirement understanding] --> B[Task planning]
  B --> C[Brownfield codebase analysis]
  C --> D[Architecture and design]
  D --> C{Ambiguous?}\n  C -->|Yes| Q[Clarification gate]\n  Q --> H{Human approval}\n  C -->|No| H
  H -->|Approved| I[Implementation]
  H -->|Rejected| X[Rejected / safe stop]
  I --> T[Testing]
  I --> DOC[Documentation]
  T --> G[Release readiness synchronization]
  DOC --> G
  G --> Z[Completed — no deployment]
```

The testing and documentation tasks are independent children of implementation and synchronize at release readiness. Each task has explicit dependencies; a task cannot run until all dependencies succeed.

## 4. Governance and safety
- No deployment, production mutation, or external side effect is performed by the agent.
- Implementation is held behind a human approval endpoint.
- Rejected work is marked skipped downstream and retained in the audit trail.
- Task retries are bounded; unexpected failure transitions the run to `SAFE_STOP`.
- Replanning invalidates prior successful task outputs and reruns upstream stages, returning to the approval gate.
- URL destinations accept only absolute HTTP/HTTPS URLs; `javascript:`, `file:`, and relative destinations are rejected.
- Secrets are read from environment variables. Never commit real database credentials or production JWT secrets.
- Existing URL update/delete operations retain owner checks.

## 5. Key decisions and trade-offs
| Decision | Rationale | Trade-off |
|---|---|---|
| Add orchestration to existing Spring Boot service | Brownfield enhancement; preserves API and UI | Prototype is a modular monolith, not a distributed agent platform |
| Explicit DAG | Makes dependencies and synchronization reviewable | DAG is currently defined in code |
| Deterministic agent adapters | Runnable and reproducible without LLM keys | Does not demonstrate model quality/tool calling |
| In-memory workflow state | Keeps prototype easy to run | Runs/audit are lost on restart; use DB persistence for production |
| Human approval before implementation | High-impact change-control boundary | Reviewer identity/role mapping needs enterprise IAM integration |
| No automatic deployment | Safe default | Release step reports readiness only |
| MySQL local default | Matches included Docker Compose | PostgreSQL can be configured using environment overrides |

## 6. Production hardening backlog
1. Persist runs, tasks, approvals, decisions, and audit events in relational tables.
2. Add optimistic locking and idempotency keys for run creation and approval actions.
3. Integrate a real LLM through a provider abstraction with strict tool allow-lists and JSON-schema output validation.
4. Execute task workers asynchronously through a durable queue; add leases, timeouts, cancellation, and dead-letter handling.
5. Enforce reviewer RBAC and separation of duties through enterprise identity.
6. Add code-diff sandboxing, SAST/dependency scanning, SBOM generation, and signed build artifacts.
7. Replace prototype latency/MTTR estimates with measured task/run telemetry and incident recovery timestamps.
8. Add rate limiting, abuse detection, URL reputation checks, and retention controls for click analytics.
