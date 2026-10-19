# Stays · Lisbon Hotel Reservations

A React hotel reservation app backed by a Rust API and PostgreSQL. This fork replaces the former Java services with one deployable Axum backend while retaining the existing React design system and HTTP API contracts.

![Stays hotel reservation search page](docs/screenshots/hotel-reservation-home.png)

## Case study and known defects

This is the **Rust rebuilt** implementation in [One Hotel Design, Five AI Implementations](https://marcelomiyake.com.br/posts/five-ai-hotel-implementations/), which compares five AI-built versions of the same hotel design, and in its [Part II on the cost of a new feature](https://marcelomiyake.com.br/posts/five-ai-hotel-implementations-part-2/). Part I inspected commit [`fba2378`](https://github.com/marcelomiyake/hotel-rust-rebuilt/commit/fba23789a507bf3179a214560ee7f215ad9eaa74).

The study found:

- Two overlapping requests cancelling the same reservation both subtracted inventory: with another booking still active, the probe ended with zero rooms reserved instead of one. The handler reads the booking before its cancellation transaction starts.
- A booking for 999 guests was accepted for a room type whose maximum is 2. Search filters by capacity; booking checks only that the count is positive.
- With an empty configured admin key, a request with an empty `X-Admin-Key` header updated a rate (HTTP 200).
- Anyone who knows a reservation's UUID can read it; there is no guest authentication.
- Creating the pending booking, charging, and confirming are separate commits, so a failure between them leaves a pending booking holding inventory.

No later commit fixes these. The passing tests and SonarQube Cloud results below do not cover them. This is a local learning implementation, not a production system.

## Product behavior

- Search Lisbon hotels by destination, dates, and guest count; compare availability and nightly rates.
- Reserve by room type. PostgreSQL stores nightly inventory, supports up to 10% overbooking, and uses version-checked updates for concurrent bookings.
- Replay a reservation with its UUID as an idempotency key. Reusing that key with changed details returns a conflict.
- Read reservation history by email and cancel a confirmed reservation. Cancellation releases inventory and records a simulated refund.
- Let staff manage hotels, room types, inventory, and nightly rates behind the admin key.

The seeded catalog contains two Lisbon hotels. Payments are simulated; the app does not collect card details or contact a hotel.

## Architecture

`services/backend` is a single Rust 2024 service using Axum, SQLx, and PostgreSQL. Its modules group the hotel catalog, rates, payments, reservations/search, API models, and HTTP error mapping. SQL schemas are in `services/backend/schema`. The Docker build compiles a small release binary and runs it as an unprivileged user.

The React frontend remains organized into domain, application, infrastructure, and presentation areas. Its `HttpHotelGateway` continues to call the same `/api` routes, and `frontend/src/styles.css` retains the established design system.

## Run on local Kind

Requirements: Docker, Kind, kubectl, and `openssl`.

```bash
./scripts/kind-up.sh
```

Open [http://localhost:8080](http://localhost:8080). The script creates the `hotel-reservation` cluster, builds and loads the Rust API and web images, and deploys the app. It creates local database and staff keys in `.kind-db-password` and `.kind-admin-key`; Git ignores those files.

```bash
kubectl -n hotel-reservation get deployments,pods,services
./scripts/kind-down.sh
```

## HTTP API contracts

Admin endpoints require the `X-Admin-Key` header. Request and response JSON fields are covered by the backend API contract test.

| Method and route | Purpose |
| --- | --- |
| `GET /health` | Kubernetes health check |
| `GET /api/search?destination=Lisbon&checkIn=YYYY-MM-DD&checkOut=YYYY-MM-DD&guests=2` | Search availability and rates |
| `GET /api/hotels?destination=Lisbon` | Read the hotel catalog |
| `GET /api/hotels/{id}` | Read a hotel and its room types |
| `GET /api/rates/quote?roomTypeId=…&checkIn=…&checkOut=…` | Quote every night in a stay |
| `POST /api/payments` | Charge a simulated payment |
| `PUT /api/payments/{reservationId}/refund` | Refund a simulated payment |
| `POST /api/reservation-journeys/events` | Record a privacy-minimal reservation journey event |
| `POST /api/reservations` | Create or replay a reservation |
| `GET /api/reservations?email=…` | Read a guest’s reservation history |
| `GET /api/reservations/{id}` | Read one reservation |
| `DELETE /api/reservations/{id}` | Cancel a reservation and refund its demo payment |
| `POST /api/admin/hotels` | Add a hotel |
| `PUT, DELETE /api/admin/hotels/{id}` | Update or remove a hotel |
| `POST /api/admin/hotels/{hotelId}/room-types` | Add a room type |
| `PUT, DELETE /api/admin/room-types/{id}` | Update or remove a room type |
| `PUT /api/admin/inventory` | Update future inventory |
| `PUT /api/admin/rates` | Update one nightly rate |
| `POST /api/admin/rates/schedule` | Create a nightly rate schedule |

## Reservation abandonment tracking

The frontend starts a journey when a guest opens a stay's details, then records screen transitions with a random journey UUID and an increasing event sequence. The UUID is reused as the reservation ID if the guest completes the booking. The event tables store only the UUID, screen, event type, sequence, and server timestamp; they do not store guest names, email addresses, or browser identifiers.

`reservation_analytics.journeys` keeps the latest screen and activity time, while `reservation_analytics.journey_events` keeps the ordered event history. A journey with no activity for 30 minutes and no successful payment appears as `ABANDONED` in the `reservation_analytics.journey_outcomes` view. Paid bookings appear as `COMPLETED`; failed payments are classified separately. The view calculates status at query time, so no background job or administrative frontend is required.

Query the journeys that have become abandoned and see where they stopped:

```sql
SELECT journey_id, last_screen, started_at, last_activity_at, abandoned_at
FROM reservation_analytics.journey_outcomes
WHERE status = 'ABANDONED'
ORDER BY abandoned_at DESC;
```

## Development and quality checks

Frontend checks:

```bash
cd frontend
npm ci
npm test
npm run lint
npm run build
```

Backend formatting, linting, and coverage (Rust toolchain, `cargo-llvm-cov`, and a reachable disposable PostgreSQL database are required):

```bash
cd services/backend
cargo fmt --check
cargo clippy --all-targets --all-features -- -D warnings
DATABASE_URL=postgres://hotel_app:hotel_local@localhost:5432/hotel_test \
  cargo llvm-cov --all-targets
```

The verified frontend suite has **15 passing tests** and **92.38% line coverage**. The Rust suite has **15 passing tests** (14 unit tests and one API contract test) and **91.39% line coverage**.

### SonarQube Cloud

The public [Hotel Reservation System – Rust Backend SonarQube Cloud project](https://sonarcloud.io/project/overview?id=marcelomiyake_hotel-dry-kiss-yagni-refactored-ddd-cleanarch-cqrs-rermade-rust) uses scanner-based analysis; Automatic Analysis is disabled for this project. The scan ran the Rust Enterprise sensor, imported Rust coverage in Sonar’s generic coverage format, imported Clippy findings in the generic external issues format, and imported frontend LCOV coverage. See Sonar’s [Rust analysis documentation](https://docs.sonarsource.com/sonarqube-server/analyzing-source-code/languages/rust), [generic issue import format](https://docs.sonarsource.com/sonarqube-cloud/analyzing-source-code/importing-external-issues/generic-issue-data), and [coverage import guide](https://docs.sonarsource.com/sonarqube-cloud/analyzing-source-code/test-coverage/overview).

The latest verified analysis reports **0 open issues across the project**, **91.4% overall coverage**, **92.2% new-code coverage**, and a **passed quality gate**. The project has one historical issue in the closed state. Strict Clippy completed without warnings.

To prepare the Rust reports for a later analysis, first run coverage and Clippy from `services/backend` with `DATABASE_URL` set to a disposable database:

```bash
mkdir -p target/sonar
cargo llvm-cov --all-targets --lcov --output-path target/sonar/rust-lcov.info
cargo clippy --all-targets --all-features --message-format=json -- -D warnings \
  > target/sonar/rust-clippy.jsonl
cd ../..
python3 scripts/sonar-rust-reports.py "$PWD" \
  services/backend/target/sonar/rust-lcov.info \
  services/backend/target/sonar/rust-clippy.jsonl
```

Run `npm test` in `frontend` to create its LCOV report, then run SonarScanner CLI from the repository root with `SONAR_TOKEN` and `SONAR_HOST_URL=https://sonarcloud.io` set in the environment. The project settings are in `sonar-project.properties`.

### Lighthouse and SEO

The production frontend was audited with Lighthouse 13.5.0 using the desktop preset. It scored **100** for performance, accessibility, best practices, and SEO (and 100 in the experimental agentic-browsing category). The page includes a title, description, robots directives, Open Graph and Twitter metadata, TravelAgency JSON-LD, `robots.txt`, `llms.txt`, and an AI Catalog manifest.

## AI build record

This implementation was generated with Codex (GPT-6 Luna, max effort). The original prompts, token usage, cost estimates, and change counts for the initial build and the reservation-abandonment feature are in [docs/ai-build-record.md](docs/ai-build-record.md).
