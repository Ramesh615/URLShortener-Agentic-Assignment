# Shortify — URL Shortener with Agentic Workflow

Shortify is a full-stack URL-shortening application with user authentication, link management, click analytics, and a prototype agentic engineering workflow.

The repository contains:
- **Frontend:** React 18 + Vite, Material UI, Tailwind CSS, Axios, and Chart.js.
- **Backend:** Java 21 + Spring Boot 3.4.1, Spring Security, Spring Data JPA, JWT, and Bean Validation.
- **Database:** MySQL 8 by default, with PostgreSQL driver support.
- **Agentic workflow:** A deterministic orchestration prototype that models requirements, planning, codebase analysis, design, implementation, testing, documentation, and release-readiness gates.

> **Important:** The agentic workflow currently uses built-in deterministic agent adapters. It does not call an external LLM, modify source files, or deploy software. Human approval is required before the implementation stage, and deployment is disabled.

## Features

### URL shortener
- Register and authenticate users.
- Create short links from original HTTP(S) URLs.
- View links belonging to the signed-in user.
- Redirect short codes to their destination.
- Update or delete owned short links.
- Track click events and view analytics over a date range.
- View total clicks grouped by date.

### Agentic engineering workflow
- Create a workflow run using a `GREENFIELD`, `BROWNFIELD`, or `AMBIGUOUS` scenario.
- Generate requirement, planning, codebase-analysis, and design task outputs.
- Pause ambiguous requirements for clarification.
- Pause before implementation for explicit human approval.
- Reject a proposed run and skip downstream tasks.
- Execute testing and documentation branches concurrently after implementation.
- Record workflow decisions and audit events.
- Replan an active workflow when requirements change.
- Expose basic workflow metrics and safe-stop behavior.

## Repository layout

```text
.
├── Images/                     # Application screenshots
├── Url-Shortner-Frontend/      # React/Vite frontend
│   ├── src/
│   │   ├── apis/               # Axios API client
│   │   ├── components/         # Pages and reusable UI
│   │   ├── contextApi/         # Shared application context
│   │   ├── hooks/
│   │   └── utils/
│   └── package.json
└── Url-Shortner-sb/            # Spring Boot backend
    ├── src/main/java/          # Controllers, services, entities, security
    ├── src/main/resources/
    ├── src/test/               # Tests
    ├── docker-compose.yaml
    └── pom.xml
```

## Architecture overview

```text
Browser
  |
  v
React + Vite frontend
  |
  | HTTP / JSON (JWT for protected endpoints)
  v
Spring Boot REST API
  |                  |
  |                  +--> Agentic workflow orchestrator
  |                       (in-memory workflow runs, task DAG,
  |                        approval/clarification gates, audit trail)
  |
  +--> Spring Security + JWT
  |
  +--> Spring Data JPA
           |
           v
       MySQL database
```

The URL shortener persists users, URL mappings, and click events through JPA. The workflow prototype stores workflow runs in application memory; workflow state is therefore lost when the backend process restarts.

## Prerequisites

- Java 21
- Node.js 18 or later and npm
- Docker Desktop / Docker Engine with Docker Compose (recommended for MySQL)
- Git

## Run locally

### 1. Start the database

Open a terminal:

```bash
cd Url-Shortner-sb
docker compose up -d mysql-db
```

The Compose configuration publishes MySQL on `localhost:3307` and creates:
- Database: `shortifydb`
- Username: `shortifyuser`
- Password: `shortifypassword`

These are development defaults. Change them before using the application outside a local environment.

### 2. Start the backend

In `Url-Shortner-sb`, run:

**Linux / macOS**
```bash
./mvnw spring-boot:run
```

**Windows**
```bat
mvnw.cmd spring-boot:run
```

The API starts at:

```text
http://localhost:9090
```

The default datasource settings are in `src/main/resources/application.properties`. They can be overridden using environment variables:

| Variable | Default |
|---|---|
| `SERVER_PORT` | `9090` |
| `SPRING_DATASOURCE_URL` | `jdbc:mysql://localhost:3307/shortifydb?...` |
| `SPRING_DATASOURCE_USERNAME` | `shortifyuser` |
| `SPRING_DATASOURCE_PASSWORD` | `shortifypassword` |
| `SPRING_DATASOURCE_DRIVER` | `com.mysql.cj.jdbc.Driver` |
| `JPA_DDL_AUTO` | `update` |
| `JWT_SECRET` | Development-only secret in properties |
| `JWT_EXPIRATION_MS` | `172800000` (48 hours) |
| `FRONTEND_URL` | `http://localhost:5173` |

**Security:** Set a unique, strong Base64-encoded JWT secret (at least 256 bits) through `JWT_SECRET` for every non-local environment. Do not commit production credentials or secrets.

### 3. Start the frontend

Open a second terminal:

```bash
cd Url-Shortner-Frontend
npm install
```

Create a `.env` file in the frontend directory:

```env
VITE_BACKEND_URL=http://localhost:9090
```

Start Vite:

```bash
npm run dev
```

Open the URL printed by Vite (normally `http://localhost:5173`).

### 4. Run backend and database with Docker Compose

From `Url-Shortner-sb`:

```bash
docker compose up --build
```

This starts MySQL and the backend. The backend is available at `http://localhost:9090`. The frontend is not included in this Compose file; start it separately using the steps above.

## REST API

Unless otherwise specified, protected endpoints require:

```http
Authorization: Bearer <JWT_TOKEN>
```

### Health

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/health` | Application health check |

### Authentication

**Register**

`POST /api/auth/public/register`

```json
{
  "username": "demo",
  "email": "demo@example.com",
  "password": "ChangeMe@123"
}
```

**Login**

`POST /api/auth/public/login`

```json
{
  "username": "demo",
  "password": "ChangeMe@123"
}
```

Use the token returned by the login response as the Bearer token for protected requests.

### URL management and analytics

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/api/urls/shorten` | Create a short URL |
| `GET` | `/api/urls/myurls` | List the authenticated user's URLs |
| `PUT` | `/api/urls/{shortUrl}` | Update the destination of an owned short URL |
| `DELETE` | `/api/urls/{shortUrl}` | Delete an owned short URL |
| `GET` | `/api/urls/analytics/{shortUrl}?startDate={start}&endDate={end}` | Click events for a URL and date-time range |
| `GET` | `/api/urls/totalClicks?startDate={date}&endDate={date}` | Daily click totals for the authenticated user |
| `GET` | `/{shortUrl}` | Redirect to the original URL |

**Create a short URL**

`POST /api/urls/shorten`

```json
{
  "originalUrl": "https://example.com/a/long/page"
}
```

**Update a destination**

`PUT /api/urls/{shortUrl}`

```json
{
  "originalUrl": "https://example.com/another/page"
}
```

**Get click analytics**

`GET /api/urls/analytics/{shortUrl}?startDate=2026-11-01T00:00:00&endDate=2026-11-08T23:59:59`

Dates use ISO local date-time format: `yyyy-MM-dd'T'HH:mm:ss`.

**Get daily click totals**

`GET /api/urls/totalClicks?startDate=2026-11-01&endDate=2026-11-08`

Dates use ISO local date format: `yyyy-MM-dd`.

**Redirect**

Open `http://localhost:9090/{shortUrl}` in a browser. A valid short code responds with HTTP `302` and redirects to the destination; an unknown code returns `404`.

## Agentic workflow API

All `/api/agentic/**` endpoints are protected by the application's general authentication rule. Send a valid Bearer token.

### Workflow lifecycle

```text
Create run
   |
   v
Requirements -> Planning -> Codebase analysis -> Design
   |
   +-- AMBIGUOUS scenario --> Wait for clarification --> Continue
   |
   v
Wait for human approval
   |
   +-- Reject --> REJECTED (pending tasks skipped)
   |
   +-- Approve
          |
          v
     Implementation
          |
          +------ Testing --------+
          |                       |
          +------ Documentation -+
                  (parallel)
                       |
                       v
                Release readiness
                       |
                       v
                   COMPLETED
```

### Endpoints

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/api/agentic/runs` | Create a workflow run |
| `GET` | `/api/agentic/runs/{runId}` | Get run state, tasks, decisions, and audit trail |
| `POST` | `/api/agentic/runs/{runId}/clarify` | Submit clarification |
| `POST` | `/api/agentic/runs/{runId}/approve` | Approve implementation |
| `POST` | `/api/agentic/runs/{runId}/reject` | Reject implementation |
| `POST` | `/api/agentic/runs/{runId}/replan` | Replan an active run |
| `GET` | `/api/agentic/metrics` | Get aggregate workflow metrics |

**Create a run**

`POST /api/agentic/runs`

```json
{
  "requirement": "Build a URL shortener with click analytics",
  "scenario": "GREENFIELD"
}
```

Supported scenarios:
- `GREENFIELD` — new capability / application.
- `BROWNFIELD` — change to an existing codebase.
- `AMBIGUOUS` — intentionally pauses for clarification before approval.

The response includes a generated `id`, status, current stage, task map, decisions, timestamps, and audit trail. Save the returned `id` as `runId`.

**Clarify an ambiguous run**

`POST /api/agentic/runs/{runId}/clarify`

```json
{
  "decision": "Default expiry is 90 days",
  "rationale": "Confirmed by the product owner"
}
```

After clarification, the workflow proceeds to the human approval gate.

**Approve implementation**

`POST /api/agentic/runs/{runId}/approve`

```json
{
  "comment": "Design reviewed and approved"
}
```

**Reject implementation**

`POST /api/agentic/runs/{runId}/reject`

```json
{
  "comment": "The proposed scope needs security changes"
}
```

**Replan an active workflow**

`POST /api/agentic/runs/{runId}/replan`

```json
{
  "decision": "Add configurable link expiration",
  "rationale": "Updated product requirement"
}
```

Completed or rejected runs cannot be replanned.

**Workflow metrics**

`GET /api/agentic/metrics`

Returns total and completed runs, failed/safe-stopped runs, success rate, retry count, rollback count, and average completed-run latency. These metrics are in-memory prototype metrics, not durable operational telemetry.

### Workflow statuses

- `RUNNING`
- `WAITING_FOR_CLARIFICATION`
- `WAITING_FOR_APPROVAL`
- `COMPLETED`
- `REJECTED`
- `FAILED`
- `SAFE_STOP`

The workflow service uses a task dependency graph. Testing and documentation are independent branches that run concurrently after implementation, and release readiness waits for both branches. Exceptions trigger a safe-stop path and human-review audit events.

## Tests and build

Run backend tests:

```bash
cd Url-Shortner-sb
./mvnw test
```

On Windows:

```bat
mvnw.cmd test
```

Build the backend JAR:

```bash
./mvnw clean package
```

Frontend lint and production build:

```bash
cd Url-Shortner-Frontend
npm run lint
npm run build
```

The repository includes an agent workflow service test covering approval, rejection, clarification, duplicate approval protection, and replanning.

## Monitoring

Spring Boot Actuator exposes the configured health, info, metrics, and Prometheus endpoints under `/actuator` (for example, `/actuator/health` and `/actuator/prometheus`). Access to details depends on the application's security configuration.

## Notes and limitations

- Workflow run state is held in memory and is not persisted across backend restarts.
- Built-in workflow agents produce deterministic descriptive artifacts; they are not autonomous LLM agents.
- The release stage checks workflow readiness only. It does not deploy the application.
- The Compose setup is intended for local development. Replace default passwords and JWT secrets before any shared or production deployment.
- The frontend API client reads its backend URL from `VITE_BACKEND_URL`; configure this at frontend build/dev time.
