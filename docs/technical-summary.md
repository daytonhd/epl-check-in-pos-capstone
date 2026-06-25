# Technical Summary

## System Overview

The EPL Check-In / POS System is a point-of-sale and inventory check-in application built for the Electronics Prototyping Lab (EPL). It supports the operational workflow of a lab store: tracking inventory, managing customer records, processing purchases and returns, renting equipment and lockers, assembling course lab kits, and maintaining an auditable history of stock changes. The system pairs a staff-facing operational portal with a public-facing catalog, served by a single REST API over a relational database. It centralizes inventory accuracy, customer data, financial transactions, and rental lifecycles in one consistent system rather than across disconnected manual processes.

## Application Architecture

The application is organized as a full-stack monorepo with three tiers that communicate through a versioned REST API:

- **Backend** — a FastAPI (Python) service exposing JSON endpoints under `/api/v1`, with business logic concentrated in a CRUD layer and persistence handled through SQLModel.
- **Frontend** — a React + TypeScript single-page application built with Vite, consuming the backend through a generated, typed API client.
- **Database** — a PostgreSQL instance accessed via SQLModel/SQLAlchemy, with schema evolution managed by Alembic migrations.

The frontend and backend are decoupled: the backend publishes an OpenAPI schema, and the frontend generates its API client from that schema. Local orchestration is handled by Docker Compose, which runs the database, backend, frontend, a mail-capture service, a database admin UI, and a reverse proxy.

## Backend

The backend is built on **FastAPI** with **SQLModel** as the ORM and Pydantic for data modeling. Routing is split across domain-specific modules registered on a central `APIRouter`, covering authentication/login, users, categories, inventory items, inventory audit logging, an inventory ledger view, lab kits, customers, transactions, item rentals, returns, academic terms, lockers, locker rentals, and administrative utilities. A `private` route set is mounted only in the local environment.

Business logic is centralized in a large CRUD module rather than in route handlers, keeping endpoints thin. Validation operates at two levels: declarative field constraints on Pydantic/SQLModel schemas (string length limits, non-negative quantities and prices, minimum collection sizes) and domain rules enforced in the CRUD layer, which raise `ValueError` for invalid operations (for example, selling a non-purchasable item, renting without a rental price, or returning against an invalid original transaction). Routes translate these into HTTP `422` responses, and use `404`/`400` for not-found and state errors.

Authentication uses OAuth2 password flow with JWTs (HS256 via PyJWT) and password hashing through `pwdlib` configured with Argon2 and bcrypt. Authorization is expressed through FastAPI dependencies: `get_current_user` validates the token and checks that the account exists and is active, while `get_current_active_superuser` gates administrative actions on the `is_superuser` flag. Most operational endpoints require an authenticated user.

The service also includes integrity-conscious transaction handling. During checkout and returns, inventory rows are locked with `SELECT ... FOR UPDATE` to prevent concurrent stock races, stock levels are validated before commit, and writes are committed atomically. A background scheduler (APScheduler) dispatches rental and locker reminder and overdue emails on a daily cadence in a fixed timezone, recording outcomes in an email log. Email delivery uses Jinja2 templates, and Sentry is initialized outside the local environment. CORS origins are configurable.

## Frontend

The frontend is a **React 19 + TypeScript** application bundled with **Vite**. Routing is file-based via TanStack Router, server state is managed with TanStack Query, and tabular data uses TanStack Table. The UI is composed from Radix UI primitives wrapped in a local component library and styled with Tailwind CSS v4; forms use React Hook Form with Zod validation, and light/dark theming is supported.

The application is divided into two layout areas. A **staff portal** provides the dashboard, point-of-sale workflows (home, checkout/purchase, check-in, returns and refunds, and rentals), inventory item management, customer management, transaction history and detail, an audit log, user administration, and account settings. A **public-facing layout** exposes a browsable catalog by category, an inventory listing, search, item detail, and site settings. Authentication screens cover login, signup, password recovery, and password reset. POS state is coordinated through a dedicated context, with check-in and checkout logic factored into service modules.

## Database and Data Modeling

The system uses **PostgreSQL** with SQLModel mapping. The schema defines tables for the core domains: users, categories, inventory items, inventory audit logs, customers, lab kits and their line items, transactions and itemized transaction lines, item rentals, academic terms, lockers, locker rentals, application configuration, and an email log.

Schema design follows several consistent patterns. Monetary values use fixed-precision `Decimal`/`Numeric` columns, item metadata is stored in a `JSONB` column, timestamps are timezone-aware, users are keyed by UUID, and several tables use database identity columns for primary keys. Relationships are declared with explicit foreign keys and deletion policies — `RESTRICT` to preserve historical and financial records, `CASCADE` for owned child rows such as kit and transaction line items, and `SET NULL` where references are optional. Integrity constraints include unique fields (email, SKU, locker number, term name), a composite unique constraint on category/subcategory, and non-negativity checks on quantities and prices. Enumerated status fields (rental, locker, locker-rental, audit reason codes, email kind/status) constrain state values.

Schema evolution is managed through Alembic migrations, including merge migrations reconciling parallel heads, a cascade-delete refactor, and a migration that creates an inventory ledger database view used by the ledger endpoint.

## API and Integration Flow

Communication is REST/JSON under the `/api/v1` prefix. The backend customizes OpenAPI operation IDs (tag plus route name) to produce clean client method names. The frontend client is generated from the backend's OpenAPI document using `@hey-api/openapi-ts`, producing typed SDK services, request/response types, and JSON schemas (`sdk.gen.ts`, `types.gen.ts`, `schemas.gen.ts`) backed by an axios transport.

A `generate-client.sh` script retrieves the schema from a running server when available, or otherwise generates it in-process, then regenerates the client. A companion check script and pre-commit hook verify that the committed client stays in sync with the backend contract. List endpoints follow a consistent paginated shape (a `data` array plus a `count`), with filter parameters supplied through dependency-injected filter models, and dedicated public response models separate API output from internal table models.

## Testing and Quality

Backend testing uses **pytest** across roughly twenty-nine test modules organized into route-level API tests, CRUD unit tests, integration tests for transaction, return, and locker flows, script tests, and email tests. Tests run against a real database session through FastAPI's `TestClient`, with shared fixtures for the database and for superuser/normal-user token headers. Coverage is measured and enforced in CI with a 90% minimum threshold.

Frontend testing uses **Playwright** end-to-end specs covering authentication, signup, login, password reset, admin, item, and user-settings flows, executed across four shards in CI with an authenticated setup step. Static quality is enforced by `ruff` (lint and format) and `mypy` in strict mode for Python, and `biome` for the frontend, all wired into pre-commit hooks alongside the client-sync check.

Continuous integration runs through GitHub Actions, including backend tests, sharded Playwright tests, a pre-commit workflow, a Docker Compose test, a contract/integration check that regenerates the client and builds the frontend to detect contract drift, and coverage publishing.

## Local Development and Configuration

Local development is driven by **Docker Compose**. The stack defines a PostgreSQL service with a health check, an Adminer database UI, a Mailcatcher SMTP capture service, a prestart service that waits for the database and applies migrations, the backend and frontend services, a database-backup service, and a Traefik reverse proxy for development routing. `docker compose watch` supports live development, and individual services can be stopped to run the backend or frontend locally instead.

Configuration is centralized in a root `.env` file consumed through Pydantic settings, covering the API prefix, secret key, token expiry, CORS origins, PostgreSQL connection, SMTP and email-scheduler options, default locker fee, and the initial superuser. Settings validation warns locally and fails outside local environments when placeholder secrets remain unchanged. Python dependencies are managed with `uv` in a workspace layout, and the frontend uses `bun`. Documented local endpoints include the frontend (5173), backend (8000), Swagger UI and ReDoc, Adminer (8080), Mailcatcher (1080), and the Traefik dashboard (8090).

## Technical Stack

- **Backend:** Python (3.10+), FastAPI, SQLModel, Pydantic / pydantic-settings, Alembic, PyJWT, pwdlib (Argon2 + bcrypt), APScheduler, `emails` + Jinja2, psycopg 3, Sentry SDK
- **Frontend:** React 19, TypeScript, Vite, TanStack Router / Query / Table, Tailwind CSS v4, Radix UI, React Hook Form, Zod, axios
- **Database:** PostgreSQL
- **Testing / Quality:** pytest, coverage, Playwright, mypy (strict), ruff, biome, pre-commit / prek
- **Tooling / DevOps:** Docker / Docker Compose, Traefik, Adminer, Mailcatcher, GitHub Actions CI, `@hey-api/openapi-ts` client generation, `uv`, `bun`
