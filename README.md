# Bruno API Automation — Restful-Booker

Automated API test suite for the public [Restful-Booker](https://restful-booker.herokuapp.com) hotel booking API, built with [Bruno](https://www.usebruno.com/) and wired into GitHub Actions with HTML/JUnit/JSON reporting.

[![API Tests](https://github.com/qasimmahmood95/bruno-api-automation/actions/workflows/api-tests.yml/badge.svg)](https://github.com/qasimmahmood95/bruno-api-automation/actions/workflows/api-tests.yml)
![Bruno](https://img.shields.io/badge/Bruno-CLI%20%5E3.5.1-f4aa41)
![Node](https://img.shields.io/badge/Node-%3E%3D18-339933)
![License](https://img.shields.io/badge/License-MIT-blue)

## About

This is an SDET portfolio project demonstrating how I approach API test automation: a full CRUD lifecycle exercised end-to-end with request chaining, a negative suite with proper test isolation, dual-layer assertions (declarative + scripted), and a CI pipeline that runs on every push to `main`, every PR, and nightly against the live API.

Every request carries a `docs` block explaining *why* it asserts what it asserts — including the places where Restful-Booker deviates from HTTP conventions, which the tests pin deliberately rather than paper over.

## Test coverage

### Positive suite — booking lifecycle (E2E)

Requests run in sequence; `bookingId` and `token` are captured via `vars:post-response` and consumed downstream.

| # | Request | Method | Endpoint | Expected | Validates |
|---|---------|--------|----------|----------|-----------|
| 1 | Health Check | GET | `/ping` | 201 | API is reachable before the suite runs |
| 2 | Get Booking IDs | GET | `/booking` | 200 | Non-empty array; every entry has a numeric `bookingid` |
| 3 | Create Booking | POST | `/booking` | 200 | Numeric id returned; response echoes the full submitted booking |
| 4 | Get Booking | GET | `/booking/{id}` | 200 | Response schema (all required keys) plus field values from step 3 |
| 5 | Create Token | POST | `/auth` | 200 | Non-empty auth token issued |
| 6 | Update Booking | PUT | `/booking/{id}` | 200 | Full replacement applied, authenticated via token cookie |
| 7 | Partial Update Booking | PATCH | `/booking/{id}` | 200 | Patched field changed; untouched fields preserved |
| 8 | Delete Booking | DELETE | `/booking/{id}` | 201 | Booking deleted (API returns 201, not 200/204) |
| 9 | Cannot Get Booking After Deleting | GET | `/booking/{id}` | 404 | Delete actually removed the resource |

### Negative suite — validation, auth and authorization

A setup request creates a dedicated fixture booking (`negativeBookingId`) so the invalid-token tests never depend on seeded data or on the positive suite having run.

| # | Request | Method | Endpoint | Expected | Validates |
|---|---------|--------|----------|----------|-----------|
| 1 | Setup — Create Fixture Booking | POST | `/booking` | 200 | Fixture created; id captured for the invalid-token tests |
| 2 | Cannot Create Booking with Invalid Name | POST | `/booking` | 500 | Numeric values in string fields are rejected |
| 3 | Cannot Create Booking with Missing Date | POST | `/booking` | 500 | Missing required `bookingdates.checkout` is rejected |
| 4 | Cannot Create Booking with Invalid JSON | POST | `/booking` | 400 | Malformed JSON payload is rejected (wire-level nuance documented in the request) |
| 5 | Cannot Create Token with Invalid Credentials | POST | `/auth` | 200 + `"Bad credentials"` | No token issued; failure reason returned (API quirk — see below) |
| 6 | Cannot Update Booking with Invalid Token | PUT | `/booking/{fixtureId}` | 403 | Unauthorized full update is forbidden |
| 7 | Cannot Delete Booking with Invalid Token | DELETE | `/booking/{fixtureId}` | 403 | Unauthorized delete is forbidden |

## Project structure

```
bruno-api-automation/
├── bruno.json                     # Bruno collection config
├── collection.bru                 # Collection-level docs
├── environments/
│   └── Restful-Booker.bru         # baseUrl + auth credentials
├── Positive API Testing/          # 9 requests: full booking lifecycle
│   ├── folder.bru                 # seq: 1 — runs first
│   └── *.bru
├── Negative API Testing/          # 7 requests: setup fixture + error cases
│   ├── folder.bru                 # seq: 2 — runs second
│   └── *.bru
├── .github/workflows/
│   └── api-tests.yml              # CI pipeline
└── package.json                   # @usebruno/cli + npm scripts
```

## Getting started

**Prerequisites:** Node.js >= 18.

```bash
npm ci                 # install @usebruno/cli

npm test               # run both suites against the Restful-Booker environment
npm run test:positive  # positive lifecycle suite only
npm run test:negative  # negative suite only
npm run test:report    # full run with HTML + JUnit reports in reports/
```

To explore the collection interactively, open the repo root in the [Bruno desktop app](https://www.usebruno.com/downloads), select the **Restful-Booker** environment, and run the folders in order. Requests within a suite are chained via runtime variables, so run a folder as a whole rather than individual requests.

### Running against a local API instance

The public demo instance is shared and occasionally slow or down. Restful-Booker ships a Docker image, so the whole suite can also run hermetically:

```bash
docker run -d -p 3001:3001 mwinteringham/restfulbooker
npx bru run --env Restful-Booker --env-var baseUrl=http://localhost:3001
```

## CI/CD

The [API Tests workflow](.github/workflows/api-tests.yml) runs on:

- every push to `main`
- every pull request
- a nightly cron at 06:17 UTC (catches regressions in the live API; scheduled off the top of the hour, which GitHub throttles)
- manual dispatch

Each run executes `npm run test:report` with one automatic retry and a hard job timeout — the target is a shared demo API on Heroku, so a single retry absorbs cold starts without masking real failures. Reports are uploaded as a build artifact (14-day retention) and a per-test summary is published as a check run via `mikepenz/action-junit-report`, so pass/fail is visible directly in the PR checks tab without downloading anything.

## Design decisions

- **Request chaining.** `bookingId`, `token`, and `negativeBookingId` are captured in `vars:post-response` and consumed by later requests — no hardcoded ids, and the positive suite is a genuine end-to-end lifecycle.
- **Test isolation.** The negative suite creates its own fixture booking instead of borrowing seeded data or leaning on the positive suite, and the invalid-token tests guard against a silently missing fixture with a fail-fast pre-request check. Either suite can run standalone.
- **Deterministic ordering.** `folder.bru` `seq` values fix suite order, and per-request `seq` values fix request order within each suite, so CLI runs are reproducible.
- **Two assertion layers.** Declarative `assert` blocks pin status codes and key fields at a glance; `tests {}` blocks with chai-style `expect` are reserved for checks `assert` can't express — schema keys, body-echo comparisons, array shape, cross-request preservation. No check is duplicated across the two layers.
- **Pin actual behaviour, don't idealize it.** Where Restful-Booker deviates from HTTP conventions, the tests assert what the API really does, and the `docs` blocks flag each deviation as a would-be bug report.
- **Parameterized configuration.** Base URL and credentials live in the environment file, not in requests. These are Restful-Booker's public, documented demo credentials — in a real project they'd be injected from a secret store, never committed.
- **Self-documenting requests.** Every `.bru` file has a `docs` block stating intent and rationale, readable in both the repo and the Bruno app.

## Known API quirks

These are deliberate assertions of Restful-Booker's actual behaviour. In a production engagement, each would be filed as a bug against the service.

| Behaviour | Conventional | Restful-Booker actual |
|-----------|--------------|-----------------------|
| `GET /ping` success | 200 OK | **201 Created** |
| `DELETE /booking/{id}` success | 200 / 204 | **201 Created** |
| Invalid field types / missing required fields | 400 Bad Request | **500 Internal Server Error** |
| Auth with bad credentials | 401 Unauthorized | **200 OK** with `{"reason": "Bad credentials"}` |

## License

[MIT](LICENSE) © Qasim Mahmood
