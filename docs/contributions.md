# Contributions

This document records my individual contribution areas for the EPL Check-In / POS System capstone. My work centered on the FastAPI / SQLModel backend: data modeling, database migrations, API endpoints, authorization, request validation, and automated tests, along with purchase-return integration and manual QA across the point-of-sale workflows.

## Backend and Database Contributions

### Store Credit Adjustment Endpoint and Audit Ledger

I built the manual store credit adjustment feature on the backend (PR #284, issue #279). I added a superuser-only `POST /customers/{customer_id}/store-credit-adjustments` endpoint that increases a customer's store credit balance and records each change in a dedicated `store_credit_adjustment` ledger. Each ledger row captures the customer, the acting user, the amount, the balance before and after, the reason, a reason code, and a created timestamp, written together with the balance update so the running balance and audit trail stay in sync.

I modeled the reason code as a string-backed application-level enum (`StoreCreditReasonCode`) to match the project's existing audit-log pattern. The request schema validates input before any balance change: it rejects zero and negative amounts, blank reasons, and invalid reason codes, and the endpoint returns 404 for unknown customers. The endpoint is administrative by design and does not create `Transaction` or `ItemizedTransaction` rows.

I covered the feature with targeted pytest coverage for successful adjustments, invalid requests, authorization behavior, and database updates. This gave staff an auditable way to credit customer accounts without routing administrative adjustments through the sales and transaction tables.

### Category and Subcategory Data Model

I delivered category and subcategory support in two stages. First (PR #212, issue #201), I added a nullable `subcategory` field to the `Category` model and its create, update, and public schemas, updated the response builder, refreshed the dev seed data, and added an Alembic migration. A subcategory can be set on create or update, or cleared by setting it to null.

Then (PR #236, issue #222), I changed the uniqueness rule from `category` alone to the composite pair `(category, subcategory)`, so a parent category can hold multiple subcategories while duplicate pairs remain blocked. This included an Alembic migration to drop the old single-column constraint and create the composite one, duplicate checks in both the create and update paths, null-subcategory handling, field-level normalization, and tests covering valid combinations, duplicate blocking, whitespace stripping, blank rejection, and PATCH duplicate handling.

### Inventory Schema Extensions

I extended the `InventoryItem` model in two areas. I added a nullable `rental_price` field alongside the regular price (PR #174, issue #107) so rentable items can carry a separate rental rate, settable on create and update and returned in API responses, backed by an Alembic migration and backend tests.

I also added a nullable `image_path` field (PR #262) with matching create, update, and public schema support plus a builder update, so an item's image location can be stored, returned, updated, or cleared. I scoped this to backend storage of the path value rather than file-upload handling. Both extensions shipped with their own Alembic migrations and inventory tests.

### Customer Deletion Integrity Handling

I fixed a data-integrity bug in customer deletion (PR #125, issue #106). Deleting a customer who had existing transactions raised a raw foreign-key `IntegrityError`, which surfaced to the client as a 500. I caught the error, rolled back the session, and returned a 409 with a clear message explaining that the customer cannot be deleted while transactions reference them, while preserving normal deletion for customers without transactions.

### Automated Tests and Migrations

Across these features I authored backend tests under `backend/tests/`, including coverage for store credit adjustments, category/subcategory behavior, and inventory fields. I also wrote the Alembic migrations that accompanied the schema changes for store credit adjustments, category/subcategory support, composite uniqueness, rental pricing, and image-path storage.

## Integration and QA Contributions

### Purchase Return Endpoint Integration

I fixed the purchase-return submission path on the POS returns page (PR #300). The page was calling a stale `submitPurchaseReturn` path; I rewired it to the existing `createPurchaseReturn` service against `POST /api/v1/returns/` and mapped selected return items into the expected request shape: `originalTransactionId`, `issueStoreCredit`, `itemId`, `quantity`, and `restock`.

I validated the change with a type-check and frontend build, then confirmed it end to end with a TC-17 retest: the return posted as a new transaction, the customer's store credit updated, and tracked inventory was restored.

### Manual QA: Cashier Checkout Workflow

I executed and documented the manual QA pass for the cashier checkout workflow (issue #248), a 19-case workflow check (TC-01 through TC-19) run against a seeded database. Coverage spanned single- and multi-item purchases, customer lookup, applying store credit, store-credit subtotal caps, tracked-versus-untracked inventory behavior, cart stock limits, item search, rental mode, no-customer and unknown-SKU rejections, receipt behavior, cart clearing after checkout, purchase returns, rental returns, and rental extensions. I recorded a pass result for all 19 cases.

## Frontend Contributions

### Sign-In Page UI

I updated the sign-in page (PR #74, issue #70): I removed the FastAPI starter logo and restyled the page and login button to match the home page in both light and dark mode.

## Technologies Used

FastAPI, SQLModel, Pydantic, PostgreSQL, Alembic, REST APIs, pytest, React, TypeScript, Docker, Git, and GitHub.
