# Testing, Validation, Risks and Limitations

## Automated tests
`AgenticWorkflowServiceTest` verifies:
- greenfield execution pauses at approval and completes after approval;
- rejection prevents downstream work;
- ambiguous requests record an assumption;
- approval is rejected when no approval gate is active.

Run backend tests with `./mvnw test` (or `mvn test`).

## Manual end-to-end validation
1. Start MySQL and backend.
2. Register/login using `/api/auth/public/register` and `/api/auth/public/login`.
3. Use the returned JWT for `/api/urls/shorten`; verify redirect and click analytics.
4. Create a workflow run for each of the three scenarios.
5. Confirm approval gate, approve/reject, inspect task states and audit trail.
6. Confirm metrics change after runs complete.

## Main risks and mitigations
- **Malicious redirect destinations:** only HTTP(S) absolute URLs are accepted. Production should additionally apply domain reputation/deny lists and SSRF-safe preview handling (do not fetch destination URLs server-side).
- **Short-code collision:** secure random generation plus unique database constraint; production should retry duplicate-key races.
- **Analytics privacy:** click events can contain personal data when enriched with IP/referrer/device. This prototype records timestamps only; define consent, retention, and anonymization before adding more fields.
- **Agent hallucination/tool misuse:** current agents are deterministic and have no unrestricted tools. A future LLM integration must use allow-listed tools, structured output validation, budgets, and human gates.
- **Lost orchestration state:** runs are in-memory and disappear on restart. Persist workflow state and audit events before production use.
- **False reliability metrics:** current metrics are prototype-level counters and elapsed duration, not SLO-grade MTTR. Instrument task durations and incident recovery events for operational reporting.
- **Authorization:** `/api/agentic/**` requires authentication. Reviewer role enforcement and separation of duties must be connected to real enterprise IAM before production.
- **Rollback:** current safe-stop skips pending tasks and records rollback intent; it does not reverse arbitrary source/database mutations because agents do not apply changes or deploy.
