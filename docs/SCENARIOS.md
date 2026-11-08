# Demonstration Scenarios

All scenarios use `POST /api/agentic/runs`, then inspect the returned workflow and audit trail.

## 1. Greenfield
Requirement: `Build a URL shortener that accepts HTTP(S) destinations, generates unique short codes, redirects visitors, and records click analytics.`
Scenario: `GREENFIELD`

Expected behavior:
- Requirement, plan, codebase impact, and design tasks complete.
- Run pauses at `WAITING_FOR_APPROVAL`.
- Reviewer approves using `/approve`.
- Implementation, testing, and documentation complete; release readiness synchronizes both validation branches.
- Run ends `COMPLETED`. No deployment is triggered.

## 2. Brownfield
Requirement: `Add an expiry date to existing short links without breaking the current React dashboard, JWT authentication, redirect route, or analytics APIs.`
Scenario: `BROWNFIELD`

Expected behavior:
- Codebase analysis explicitly identifies existing controllers, service, entities, repositories, JWT security, and frontend.
- Design output preserves current routes and calls out schema migration and expired-link behavior.
- Human approval is required before implementation stage.
- Validation includes backward compatibility, migration, redirect, analytics, and ownership checks.

## 3. Ambiguous
Requirement: `Make links expire automatically and show useful analytics.`
Scenario: `AMBIGUOUS`

Expected behavior:
- Orchestrator pauses at `WAITING_FOR_CLARIFICATION` because expiry duration, timezone, analytics dimensions, and retention were not specified.
- Resolve through `POST /api/agentic/runs/{runId}/clarify`; only then does the workflow reach the human implementation-approval gate.
- Suggested questions: default expiry? configurable per link? click analytics by day/device/referrer? retention period? privacy requirements?

## API examples

Create a run (JWT required):

```bash
curl -X POST http://localhost:9090/api/agentic/runs \
  -H 'Authorization: Bearer <JWT>' -H 'Content-Type: application/json' \
  -d '{"scenario":"GREENFIELD","requirement":"Build a URL shortener with click analytics"}'
```

Inspect:

```bash
curl http://localhost:9090/api/agentic/runs/<RUN_ID> -H 'Authorization: Bearer <JWT>'
```

Clarify an ambiguous requirement:

```bash
curl -X POST http://localhost:9090/api/agentic/runs/<RUN_ID>/clarify \\
  -H 'Authorization: Bearer <JWT>' -H 'Content-Type: application/json' \\
  -d '{"decision":"Default expiry is 90 days","rationale":"Product owner confirmed this default"}'
```

Approve:

```bash
curl -X POST http://localhost:9090/api/agentic/runs/<RUN_ID>/approve \
  -H 'Authorization: Bearer <JWT>' -H 'Content-Type: application/json' \
  -d '{"reviewer":"reviewer@example.com","comment":"Design and scope reviewed"}'
```

Reject:

```bash
curl -X POST http://localhost:9090/api/agentic/runs/<RUN_ID>/reject \
  -H 'Authorization: Bearer <JWT>' -H 'Content-Type: application/json' \
  -d '{"reviewer":"reviewer@example.com","comment":"Clarify retention requirements first"}'
```

Replan:

```bash
curl -X POST http://localhost:9090/api/agentic/runs/<RUN_ID>/replan \
  -H 'Authorization: Bearer <JWT>' -H 'Content-Type: application/json' \
  -d '{"decision":"Use configurable expiry","rationale":"Product owner confirmed per-link expiry"}'
```

Metrics: `GET /api/agentic/metrics` (JWT required).
