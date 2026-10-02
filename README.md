# 🐱 DataCat

**Application telemetry and security monitoring dashboard.**

DataCat collects telemetry from applications (requests, logs, errors, authentication activity, user activity) and turns it into a single dashboard for monitoring, investigation and rule-based threat detection. It is designed around *applications*, not around any one app. The first monitored application is **OpsLane**.

> **Status: Phase 1 (frontend MVP), in active development.** The dashboard currently runs entirely on a local JSON file. There is no backend, database or authentication yet. See the [Roadmap](#roadmap).

---

## Table of contents

1. [Why DataCat](#why-datacat)
2. [Features](#features)
3. [Current status](#current-status)
4. [Architecture](#architecture)
5. [Getting started](#getting-started)
6. [Project structure](#project-structure)
7. [Telemetry data model](#telemetry-data-model)
8. [Dashboard pages](#dashboard-pages)
9. [Detection rules](#detection-rules)
10. [Planned telemetry API](#planned-telemetry-api)
11. [Security considerations](#security-considerations)
12. [Roadmap](#roadmap)
13. [Design principles](#design-principles)
14. [Contributing](#contributing)
15. [License](#license)

---

## Why DataCat

Most applications log to files nobody reads until something breaks. DataCat gives a small team one place to see what an application is doing and which signals look hostile:

- What is happening right now (events, requests, errors)?
- Who is using the app, and are logins succeeding or failing?
- Is something suspicious going on (brute force, permission abuse, traffic spikes)?
- Which alerts need attention, and what evidence is behind them?

DataCat **observes** applications. It does not manage their users and does not become an application's user-management system.

## Features

**Monitoring**
- Overview with key metrics, event-volume chart, application health and recent activity
- Event stream (all telemetry normalized into one timeline)
- Error tracking with occurrence counts, affected users and severity
- Request analytics: totals, success/failure, latency, status codes, top endpoints
- User analytics: totals, new/active users, growth chart

**Security**
- Authentication activity (`login.success`, `login.failed`, `logout`, `password.changed`, `password.reset`)
- Security events (`permission.denied`, `rate_limit.exceeded`, `suspicious.request`, `api_key.created`, `api_key.revoked`)
- Alerts with severity, status and lifecycle
- Detection rule definitions (rule-based, no ML/AI)

**Configuration**
- Application details and telemetry endpoint information
- API key listing (secrets are never displayed, only key ID and prefix)

## Current status

| Area | State |
|---|---|
| Dashboard UI (all pages) | Working prototype on dummy data |
| JSON-driven rendering | Working (`fetch` of a local JSON file) |
| JSON import / paste dialog | Planned |
| Application selector | UI present, single application only |
| Logs page with filters | Planned (Events page is the current stand-in) |
| Error detail / stack trace view | Planned |
| Detection engine | Planned (rules are displayed, not executed) |
| Telemetry API, backend, database | Planned |
| Authentication for DataCat users | Planned |

Known issues and gaps are tracked in the project analysis tracker (`datacat_analysis.xlsx`) or in the issue tracker.

## Architecture

### Phase 1 (current)

```
JSON file  ->  adapter / parser  ->  DataCat state  ->  UI renderers
```

The UI consumes a defined data structure rather than being coupled to a database. The JSON used today is meant to resemble what the backend will eventually return, so the source can be swapped without rewriting the presentation layer.

### Target architecture

```
OpsLane  ->  Telemetry API  ->  DataCat backend  ->  Database (D1)  ->  DataCat frontend
```

Planned stack: Cloudflare Workers + D1 for the backend and storage. The frontend is currently vanilla JavaScript and is planned to move to React (Vite) before the codebase grows further.

Two separate kinds of authentication are kept distinct throughout the project:

```
DataCat user  ->  manages Applications  ->  each has an API key  ->  Telemetry API
```

## Getting started

### Requirements

- Any modern browser
- Python 3 (or any static file server)

The app loads JSON with `fetch`, so it must be served over HTTP. Opening `index.html` directly from disk will not work.

### Run locally

```bash
git clone <your-repo-url>
cd datacat
python3 -m http.server 8000
```

Then open <http://localhost:8000>.

### Sample data

The dashboard reads:

```
data/datacat_dummy_telemetry.json
```

If this file is missing or invalid, the UI shows a "Could not load DataCat telemetry" message.

## Project structure

```
datacat/
├── index.html                          # App shell: sidebar, topbar, content mount point
├── styles.css                          # Dashboard styling
├── app.js                              # Data loading, navigation, page renderers, SVG charts
├── data/
│   └── datacat_dummy_telemetry.json    # Mock telemetry source (current data layer)
└── README.md
```

How `app.js` is organized:

- **Data layer:** `loadData()` fetches the JSON into `DATA`. This is the function to replace when moving to a real API.
- **Helpers:** formatting (`fmt`, `pct`, `timeAgo`), escaping (`esc`), UI builders (`pill`, `panel`, `metric`, `pageHead`).
- **Charts:** `lineChart()` renders dependency-free inline SVG.
- **Renderers:** one function per page (`renderOverview`, `renderEvents`, `renderErrors`, and so on) selected by `renderPage()`.

## Telemetry data model

### Base event

```json
{
  "id": "evt_001",
  "application": "opslane",
  "timestamp": "2026-10-01T12:32:15Z",
  "type": "login.failed",
  "level": "warning",
  "message": "Failed login attempt",
  "user": { "id": "usr_123" },
  "request": {
    "method": "POST",
    "endpoint": "/api/auth/login",
    "status": 401
  },
  "network": { "ip": "192.168.1.50" },
  "metadata": {}
}
```

Every event should carry enough information to identify the time, event type, application, endpoint or action, status, user (where applicable), IP (where applicable) and extra metadata. The exact schema may evolve, but the frontend is built around this abstraction.

### Dataset top-level structure

The current renderers expect these top-level keys:

| Key | Contents |
|---|---|
| `application` | Name, environment, version, framework, status, connection info, telemetry API details |
| `overview` | Headline metrics and health score |
| `charts` | Event volume time series, status code distribution |
| `recentActivity` | Latest signals for the overview feed |
| `requests` | Summary stats and top endpoints |
| `events` | Normalized event stream |
| `errors` | Grouped errors with occurrences, affected users, severity, status |
| `users` | Summary, growth series, recent users |
| `authentication` | Summary and activity list |
| `security` | Summary and security events |
| `alerts` | Alert queue |
| `detectionRules` | Rule definitions and trigger counts |
| `system` | Service health and latency |
| `apiKeys` | Key metadata (ID, prefix, scopes, usage). **Never secret values** |

> The schema is being formalized. Planned additions include `schema_version`, `receivedAt` (server time), `applications[]` for multi-application support, `related_event_ids` on alerts, `session_id`, `trace_id` and an error fingerprint for grouping.

### Alert lifecycle

Statuses: `Open` → `Investigating` → `Resolved`, or `False Positive`.

Each alert supports severity, status, time, application, description and a related event.

## Dashboard pages

| Section | Page | Purpose |
|---|---|---|
| Monitor | Overview | Metrics, event volume, health, recent activity, top endpoints |
| Monitor | Events | Full telemetry stream with search |
| Monitor | Errors | Grouped failures and severity |
| Monitor | Requests | Traffic, latency, status codes, endpoint health |
| Monitor | Users | Usage statistics and growth |
| Security | Authentication | Login/logout/password activity |
| Security | Security | Suspicious activity and evidence |
| Security | Alerts | Alert queue and lifecycle |
| Security | Detection Rules | Rule definitions and trigger counts |
| Configure | Application | Connection and telemetry endpoint details |
| Configure | API Keys | Telemetry credentials (metadata only) |

## Detection rules

Detection is **rule-based** by design. The initial rule set:

| Rule | Condition | Severity |
|---|---|---|
| Brute force | 5 failed logins within 60 seconds from the same IP | High |
| Error spike | 20 errors within 60 seconds | Medium |
| Permission abuse | Multiple 403 responses for the same user or IP | Medium |
| Suspicious request | Suspicious request pattern | High |
| Traffic spike | Request count exceeds a threshold | Medium |

Planned extensions: password spraying (one IP, many users) and credential stuffing (many IPs, one user).

The plan is to implement rules as pure functions (`detect(events) -> alerts`), test them against dummy data containing a deliberate attack scenario, and later run the same logic in the backend.

## Planned telemetry API

Applications will send telemetry to DataCat:

```
POST /api/telemetry
X-DATACAT-KEY: dev_xxxxxxxxx
```

```json
{
  "type": "login.failed",
  "timestamp": "2026-10-01T12:32:15Z",
  "user": {},
  "request": {},
  "network": {}
}
```

"Public" means publicly reachable, not open to anonymous spam: even during development, requests require an application API key.

Planned endpoints: `POST /api/telemetry`, `GET /api/events`, `GET /api/logs`, `GET /api/errors`, `GET /api/users/stats`, `GET /api/alerts`, `GET /api/stats`.

Planned alert delivery: DataCat sends an alert webhook back to the monitored application (for example `POST /api/telemetry/alerts` on OpsLane) with `alert_id`, `type`, `severity`, `application` and `message`.

## Security considerations

DataCat is a security tool and renders data that an attacker may influence, so these rules apply:

- **Escape everything.** Telemetry fields (endpoints, emails, user agents, messages) must never be inserted into the DOM unescaped. The planned JSON import and a move to React make this more important. Add a Content Security Policy.
- **No secrets in the UI.** API key secrets are never stored or rendered; only IDs and prefixes.
- **Ingestion hardening (planned):** hash stored API keys, bind each key to a single application, rate-limit per key, cap payload size, validate schemas, deduplicate by event `id`, and use server-side `receivedAt` rather than trusting client timestamps.
- **PII:** avoid logging passwords or tokens, and consider IP truncation or hashing and showing user IDs instead of names and emails.
- **Webhooks (planned):** sign alert webhooks with HMAC.
- **Telemetry poisoning:** forged or flooded events could trigger false alerts; the ingestion design must account for this.

## Roadmap

- [x] **Phase 1a:** UI skeleton and dashboard pages on dummy JSON
- [ ] **Phase 1b:** Formal JSON schema, adapter/validator, JSON import dialog, Logs page with working filters, error detail view, full alert lifecycle, hash routing
- [ ] **Phase 1c:** Detection rules as pure functions, tested against a dummy attack scenario
- [ ] **Phase 2:** OpsLane telemetry client and `POST /api/telemetry` with API key
- [ ] **Phase 3:** DataCat backend (receive, validate, store, retrieve, statistics)
- [ ] **Phase 4:** Database (D1): applications, api_keys, events, errors, alerts, users (hourly rollups for scale)
- [ ] **Phase 5:** DataCat user authentication (register, login, logout, profile) and application management
- [ ] **Phase 6:** Real detection and alerting on live telemetry
- [ ] **Phase 7:** Alert delivery via webhook to OpsLane

### Out of scope for V1

Database, user authentication, DataCat accounts, persistent storage, detection engine, SDK, webhooks, multi-tenancy, production API security, AI/ML, infrastructure monitoring.

## Design principles

1. **Build the UI first.** Don't wait for the backend to build the product.
2. **Define the data structure, parse it, render it, then replace the source** without rewriting the presentation layer.
3. **Applications, not OpsLane.** Nothing in the UI should assume there is only one monitored application.
4. **Observe, don't manage.** DataCat watches application users; it doesn't own them.
5. **Rule-based detection.** Predictable, explainable rules rather than ML.

## Contributing

1. Fork the repository and create a feature branch.
2. Keep render functions reading from normalized data shapes, not raw API responses.
3. Escape all dynamic values inserted into HTML.
4. Open a pull request describing the change and which roadmap item it addresses.

## License

Add a license before publishing (for example MIT). Until then, all rights reserved.
