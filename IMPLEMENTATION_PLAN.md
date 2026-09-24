# Electronics Showroom Billing, Tally Sync and WhatsApp Platform

## 1. Project Decision

Build a small Python application layer around the existing TallyPrime installation.

- TallyPrime remains the source of truth for accounting, GST, billing and stock.
- Billing continues in TallyPrime.
- A local nightly sync utility runs on the Tally computer.
- The utility reads that day's completed invoices from Tally and sends safe invoice data to the AWS application.
- The AWS application sends invoice documents through the official WhatsApp Business Cloud API.
- Marketing messages are sent only to customers with recorded WhatsApp marketing consent.
- No real-time polling, Android app or replacement accounting system is required for the first release.

## 2. Target Workflow

```text
Staff creates bill in TallyPrime
        |
        v
Nightly Windows Task Scheduler job
        |
        v
Local Python Tally sync utility
        |
        | HTTPS outbound request
        v
AWS Flask application
        |
        +--> PostgreSQL database
        |
        +--> Invoice/message queue
        |
        v
Meta WhatsApp Cloud API
        |
        v
Customer receives invoice on the number stored in Tally
```

The nightly job must be idempotent. Running it twice must not create duplicate customers, invoices or WhatsApp messages.

## 3. Recommended Technology Stack

### Backend

- Python 3.12+
- Flask with application factory and Blueprints
- SQLAlchemy 2.x
- Alembic for database migrations
- Pydantic for configuration and external payload validation
- `httpx` for outbound HTTP calls
- `tenacity` for bounded retries
- Gunicorn for production WSGI serving
- `structlog` or standard-library structured logging

### Database and storage

- PostgreSQL in production
- SQLite only for local development and unit tests
- S3-compatible object storage for invoice PDFs
- Persistent database backups

### Quality tools

- pytest
- pytest-cov
- Ruff
- Black
- MyPy or Pyright
- pre-commit
- Playwright for browser smoke tests

### Infrastructure

- Docker and Docker Compose for local development
- Small AWS EC2 or Lightsail instance initially
- Nginx or an AWS-managed HTTPS proxy
- Meta WhatsApp Cloud API directly, avoiding a provider markup where possible
- Windows Task Scheduler for nightly local synchronization

## 4. Repository Structure

```text
showroom-platform/
├── app/
│   ├── __init__.py                 # Flask application factory
│   ├── config.py                   # Environment-based configuration
│   ├── extensions.py               # SQLAlchemy and other extensions
│   ├── common/
│   │   ├── errors.py               # Domain exceptions and error handlers
│   │   ├── logging.py              # PII-safe logging setup
│   │   ├── security.py             # Auth, permissions and security helpers
│   │   └── validation.py           # Shared validation helpers
│   ├── auth/
│   │   ├── routes.py
│   │   ├── service.py
│   │   └── schemas.py
│   ├── customers/
│   │   ├── models.py
│   │   ├── routes.py
│   │   ├── service.py
│   │   └── schemas.py
│   ├── invoices/
│   │   ├── models.py
│   │   ├── routes.py
│   │   ├── service.py
│   │   └── schemas.py
│   ├── tally/
│   │   ├── models.py
│   │   ├── client.py                # Tally XML/HTTP adapter
│   │   ├── parser.py                # Pure Tally response parsing
│   │   ├── service.py
│   │   └── routes.py
│   ├── whatsapp/
│   │   ├── client.py                # Meta API client
│   │   ├── templates.py
│   │   ├── service.py
│   │   ├── webhooks.py
│   │   └── routes.py
│   ├── campaigns/
│   │   ├── models.py
│   │   ├── routes.py
│   │   ├── service.py
│   │   └── schemas.py
│   └── jobs/
│       ├── models.py
│       ├── service.py
│       └── routes.py
├── connector/
│   ├── cli.py                       # `python -m connector sync`
│   ├── tally_client.py              # Local Tally connection
│   ├── exporter.py                  # Daily invoice extraction
│   ├── uploader.py                  # Secure AWS upload
│   └── config.py
├── migrations/
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
├── scripts/
│   ├── create_admin.py
│   └── validate_tally_sample.py
├── Dockerfile
├── docker-compose.yml
├── pyproject.toml
├── .env.example
└── README.md
```

## 5. Core Database Tables

### `customers`

- `id` UUID primary key
- `name`
- `phone_e164` unique normalized phone number
- `email`, optional
- `preferred_language`, optional
- `marketing_consent` boolean
- `consent_source`
- `consent_at`
- `opted_out_at`
- `tally_ledger_name`, optional
- `created_at`, `updated_at`

### `invoices`

- `id` UUID primary key
- `customer_id` foreign key
- `tally_company`
- `tally_voucher_number`
- `tally_reference` unique idempotency key
- `invoice_number`
- `invoice_date`
- `total_amount`
- `pdf_object_key`
- `sync_status`: `received`, `validated`, `sent`, `failed`
- `created_at`, `updated_at`

Create a unique constraint on the Tally company plus voucher/reference. This prevents duplicate invoices during retries.

### `message_logs`

- `id`
- `customer_id`
- `invoice_id`, nullable
- `campaign_id`, nullable
- `message_type`: `invoice` or `marketing`
- `whatsapp_message_id`
- `status`: `queued`, `sent`, `delivered`, `read`, `failed`
- `failure_code`, `failure_reason`
- `sent_at`, `delivered_at`, `read_at`

Do not store access tokens or full sensitive API payloads in this table.

### `campaigns`

- `id`
- `name`
- `template_name`
- `status`: `draft`, `scheduled`, `running`, `paused`, `completed`, `failed`
- `scheduled_at`
- `created_by`
- `started_at`, `completed_at`

### `consent_events`

- `id`
- `customer_id`
- `event_type`: `opt_in`, `opt_out`
- `source`: `billing_screen`, `manual`, `whatsapp_reply`
- `recorded_at`

### `sync_runs` and `sync_items`

Track every nightly run:

- Date range requested
- Start/end time
- Number found
- Number imported
- Number skipped as duplicate
- Number failed
- Error details

This makes the process auditable and allows a failed item to be retried safely.

## 6. Tally Integration Design

### Discovery before coding

Obtain a sample export from the real TallyPrime company and confirm:

- Invoice number field
- Voucher number and date
- Customer ledger name
- Mobile-number field
- Total amount
- Item lines, only if needed
- GST fields, only if needed
- Invoice PDF/export availability
- Cancelled invoice representation

The phone number must be stored in a structured Tally field. Do not parse mobile numbers from free-form narration.

### Connector responsibilities

The local utility should:

1. Accept a date range, defaulting to the previous business day or current day.
2. Connect to Tally's local XML/HTTP interface.
3. Request sales invoices for the range.
4. Parse the Tally response into internal DTOs.
5. Normalize each number to E.164 format, for example `+919876543210`.
6. Validate required fields.
7. Upload invoice metadata to AWS over HTTPS.
8. Upload or reference the invoice PDF.
9. Save a local sync report.
10. Exit with a non-zero status if any item failed.

### Local command interface

```bash
python -m connector sync --from 2026-09-24 --to 2026-09-24
python -m connector health-check
```

The connector should not contain marketing logic. It only extracts and securely transfers data.

### Tally connection safety

- Keep Tally's local port private.
- Never expose Tally port 9000 to the public internet.
- Use a connector token for AWS authentication.
- Use HTTPS for connector-to-AWS communication.
- Rotate the connector token if the computer is replaced.
- Use request timeouts and bounded retries.
- Use idempotency keys for every invoice.

### PDF strategy

Choose one strategy after testing the actual Tally setup:

1. Configure Tally to export each day's invoice PDFs to a known local folder and let the connector match PDFs by invoice number.
2. Use a Tally-supported export route if it reliably produces PDFs.
3. Generate a PDF from structured invoice data only if the generated document is approved for business/GST use.

Do not assume that invoice metadata automatically includes a PDF. Validate this with a real Tally invoice first.

## 7. AWS API Design

### Connector endpoint

```http
POST /api/v1/sync-runs
Authorization: Bearer <connector-token>
Idempotency-Key: <run-id>
Content-Type: application/json
```

The payload contains the run date and invoice records. The server validates every record and returns per-item results.

### Invoice endpoints

```http
GET  /api/v1/invoices?status=failed
POST /api/v1/invoices/{invoice_id}/send
GET  /api/v1/invoices/{invoice_id}/status
```

### Campaign endpoints

```http
POST /api/v1/campaigns
POST /api/v1/campaigns/{campaign_id}/preview
POST /api/v1/campaigns/{campaign_id}/start
POST /api/v1/campaigns/{campaign_id}/pause
GET  /api/v1/campaigns/{campaign_id}/report
```

### WhatsApp webhook

```http
GET  /webhooks/whatsapp
POST /webhooks/whatsapp
```

The GET route verifies Meta's webhook challenge. The POST route validates event structure and updates message delivery status. Webhook processing must be idempotent because providers can retry events.

## 8. WhatsApp Integration

Use the official Meta WhatsApp Cloud API.

Required configuration values must be environment variables:

```text
WHATSAPP_ACCESS_TOKEN
WHATSAPP_PHONE_NUMBER_ID
WHATSAPP_BUSINESS_ACCOUNT_ID
WHATSAPP_VERIFY_TOKEN
WHATSAPP_API_VERSION
```

The `WhatsAppClient` should expose small methods:

```python
class WhatsAppClient:
    def send_template(...): ...
    def upload_document(...): ...
    def send_invoice_document(...): ...
    def validate_webhook(...): ...
```

The client must:

- Use timeouts.
- Retry only safe transient failures.
- Never retry permanent validation failures.
- Redact tokens from logs.
- Record provider message IDs.
- Handle rate limits.
- Keep Meta-specific code out of business services.

### Consent rule

Marketing recipients must satisfy all conditions:

```python
customer.marketing_consent is True
customer.opted_out_at is None
customer.phone_e164 is not None
```

An opt-out message such as `STOP` or `UNSUBSCRIBE` must immediately disable future marketing messages.

## 9. Invoice Sending Flow

```text
Nightly sync receives invoice
    |
    v
Validate customer and phone number
    |
    v
Create/find customer using normalized number
    |
    v
Insert invoice using unique Tally reference
    |
    v
Store PDF securely
    |
    v
Create invoice message job
    |
    v
Send approved utility template plus PDF
    |
    v
Store WhatsApp message ID
    |
    v
Webhook changes status to delivered/read/failed
```

If the number is missing or invalid, mark the invoice as `needs_attention`; do not attempt to send it.

## 10. Marketing Campaign Flow

1. Admin selects an approved WhatsApp marketing template.
2. Application selects only opted-in customers.
3. Application excludes opted-out, invalid and duplicate numbers.
4. Admin previews recipient count and message.
5. Admin schedules or starts the campaign.
6. A bounded worker sends messages in batches.
7. Delivery status is updated through webhooks.
8. Campaign can be paused.
9. A report shows sent, delivered, read and failed counts.

The system must re-check consent immediately before sending. A customer who opts out after campaign creation must still be excluded.

## 11. Job Processing for MVP

Avoid an unnecessarily complex Celery cluster initially.

Start with a database-backed job table and a separate worker process:

```bash
python -m app.jobs.worker
```

Jobs include:

- `send_invoice`
- `send_campaign_message`
- `process_tally_sync`
- `retry_failed_message`

Use row locking or an atomic status transition so two workers cannot send the same message. If traffic grows, move the job backend to Redis and Celery/RQ without changing the service interfaces.

## 12. Security Requirements

- HTTPS everywhere outside local Tally communication.
- Passwords hashed with Argon2 or bcrypt.
- Role-based access for admin, billing and marketing users.
- Secure, HttpOnly, SameSite cookies.
- CSRF protection for browser forms.
- Rate limiting on login, webhooks and public endpoints.
- Strict CORS configuration.
- Security headers through Flask-Talisman or equivalent.
- Validate JSON using Pydantic schemas.
- Parameterized database queries through SQLAlchemy.
- Never log full phone numbers, access tokens or invoice PDFs.
- Signed/expiring invoice download URLs.
- Encrypted backups.
- Audit logs for campaign starts, opt-ins, opt-outs and resends.
- Do not expose Tally's port to AWS or the public internet.

## 13. Testing Strategy

### Unit tests

Test pure functions and service rules:

- Phone normalization
- Invalid number rejection
- Tally XML parsing
- Invoice idempotency
- Consent filtering
- Opt-out handling
- Campaign recipient selection
- Retry classification
- Date-range handling

### Integration tests

Use test doubles for:

- Tally XML responses
- Meta WhatsApp API
- Object storage
- PostgreSQL

Test:

- Successful nightly sync
- Repeated nightly sync does not duplicate invoices
- Partial sync failure can be retried
- Missing phone number is reported
- WhatsApp permanent error is not endlessly retried
- Webhook delivery events are idempotent

### Browser smoke tests

Use Playwright against a test environment to verify:

- Admin login
- Invoice status screen
- Sync result screen
- Campaign preview
- Campaign pause
- Failed message retry
- Keyboard-accessible controls

External Meta calls must be mocked in automated tests. Use one controlled test number for manual sandbox verification.

## 14. Configuration

Use `.env.example` for names only. Never commit real credentials.

```text
FLASK_ENV=production
SECRET_KEY=
DATABASE_URL=
S3_ENDPOINT_URL=
S3_BUCKET=
S3_ACCESS_KEY_ID=
S3_SECRET_ACCESS_KEY=
CONNECTOR_TOKEN_HASH=
WHATSAPP_ACCESS_TOKEN=
WHATSAPP_PHONE_NUMBER_ID=
WHATSAPP_VERIFY_TOKEN=
WHATSAPP_API_VERSION=
```

Use separate development, staging and production credentials.

## 15. Deployment Plan

### Local development

```bash
python -m venv .venv
.venv\Scripts\activate
pip install -e ".[dev]"
flask --app app run --debug
```

### AWS deployment

1. Provision a small EC2 or Lightsail server.
2. Install Docker and configure a firewall.
3. Run the Flask web container with Gunicorn.
4. Run the job worker as a separate container/process.
5. Use PostgreSQL with persistent storage.
6. Configure HTTPS and a domain.
7. Configure the Meta webhook URL.
8. Configure encrypted daily backups.
9. Install the connector on the Tally Windows computer.
10. Add a Windows Task Scheduler job for nightly execution.
11. Run a controlled test using one invoice and one WhatsApp number.
12. Enable production campaigns only after delivery and opt-out testing passes.

Example nightly Windows command:

```text
Program: C:\showroom-connector\.venv\Scripts\python.exe
Arguments: -m connector sync --from yesterday --to yesterday
Start in: C:\showroom-connector
```

Write a local log file and a server-side sync report for every run.

## 16. Implementation Phases

### Phase 1: Discovery and proof of integration

- Obtain real Tally sample exports.
- Confirm phone-number location.
- Confirm invoice PDF process.
- Create Meta developer/business configuration.
- Test one utility template.
- Document actual payloads.

### Phase 2: Foundation

- Create Flask application factory.
- Add configuration, database and migrations.
- Add authentication and roles.
- Add structured logging and error handling.
- Add health and readiness endpoints.

### Phase 3: Nightly Tally sync

- Build Tally parser.
- Build connector CLI.
- Add idempotency and sync reports.
- Add customer and invoice persistence.
- Test repeated and failed syncs.

### Phase 4: Invoice WhatsApp delivery

- Store invoice PDFs securely.
- Implement Meta client.
- Implement approved invoice template.
- Add send queue and retries.
- Add webhook status updates.
- Add resend and failure screens.

### Phase 5: Marketing

- Add consent capture and audit events.
- Add opt-out handling.
- Add customer filtering.
- Add campaign preview and pause.
- Add batch sending and reports.

### Phase 6: Production hardening

- Configure backups and monitoring.
- Run security review.
- Run browser smoke tests.
- Test restore from backup.
- Train staff.
- Start with one campaign and a small recipient group.

## 17. Success Criteria

The first release is complete when:

- A staff member can create a normal Tally invoice without changing the Tally workflow.
- The nightly sync imports all invoices in the selected date range.
- Running the sync twice does not duplicate records or messages.
- A valid customer number receives the correct invoice.
- Missing or invalid numbers are clearly reported.
- WhatsApp delivery statuses appear in the dashboard.
- Failed messages can be retried safely.
- Only opted-in customers receive marketing messages.
- STOP/UNSUBSCRIBE prevents future marketing messages.
- Tally is never exposed publicly.
- Backups and restore procedures are tested.

## 18. Cost-Control Decisions

- Do not implement real-time polling.
- Do not replace Tally billing or stock in version one.
- Use one nightly local sync utility.
- Use direct Meta Cloud API where practical.
- Use one AWS server initially.
- Keep the web application responsive instead of building mobile apps.
- Start with PostgreSQL and simple database-backed jobs.
- Add Redis/Celery only when volume requires it.
- Keep invoice and campaign services separate so each can be tested and changed independently.

## 19. Main Risks to Resolve First

1. Whether Tally exposes the customer mobile number in a reliable structured field.
2. Whether the invoice PDF can be exported or matched automatically.
3. Whether the selected WhatsApp number can be registered for the Cloud API.
4. WhatsApp template approval and current message pricing.
5. Customer marketing consent process.
6. Handling cancelled, edited or returned Tally invoices.
7. Backup and recovery of invoice documents and message history.

The first development task should therefore be a small integration proof: read a real Tally invoice, extract the number and invoice reference, upload a test record to a local Flask endpoint, and send one controlled WhatsApp test message. Only after that proof works should the full dashboard and campaign module be built.

## 20. Thirty-Day Implementation Calendar

This schedule assumes approximately **3–4 focused hours per day**, five days per week. Each day has one primary outcome. Do not start the next day's work until the checkpoint for the current day passes.

### Daily working method

Use this routine every day:

- **20 minutes:** review the previous day's result and define today's acceptance check.
- **2 hours 15 minutes:** implement or configure the day's primary task.
- **45 minutes:** write or run tests.
- **20–30 minutes:** clean up code, update `.env.example`, commit changes and record blockers.

Keep real API keys and real customer data out of the repository. Use fake customer data until the controlled integration test phase.

### Week 1 — Confirm the external systems

#### Day 1 — Freeze the MVP scope

**Tasks:**

- Confirm that billing and stock stay in TallyPrime.
- Confirm that synchronization happens manually or automatically once per night.
- Confirm the first release only sends invoices and manages consent/campaigns.
- Write a short list of excluded features: mobile app, replacement billing, multi-branch, advanced CRM.
- Create a project task board with `todo`, `doing`, `blocked`, and `done`.

**Deliverable:** approved MVP checklist.

**Checkpoint:** every required first-release feature has one acceptance statement.

#### Day 2 — Prepare the development machine

**Tasks:**

- Install Python 3.12+, Git and a code editor.
- Create the project directory and virtual environment.
- Create `pyproject.toml`.
- Add Flask, SQLAlchemy, Alembic, Pydantic, HTTPX, pytest, Ruff and Black.
- Add `.gitignore` and `.env.example`.

**Deliverable:** a clean project that can install dependencies.

**Checkpoint:** the virtual environment activates and `pytest`, `ruff` and `black` run successfully.

#### Day 3 — Collect a real Tally sample

**Tasks:**

- Create or identify one safe test invoice in Tally.
- Export the invoice/company data using the intended Tally XML/HTTP route.
- Save a redacted sample under `tests/fixtures/`.
- Identify the invoice number, voucher number, date, customer name, phone field and total.
- Record how the invoice PDF is produced.

**Deliverable:** redacted Tally sample and field-mapping notes.

**Checkpoint:** the mobile number is available in a structured field; if not, stop and fix the Tally billing data-entry process before coding further.

#### Day 4 — Configure Meta and WhatsApp test access

**Tasks:**

- Create or verify the Meta Business account.
- Register the dedicated WhatsApp API number.
- Create the WhatsApp Business API application.
- Record the phone number ID, business account ID and API version in a password manager.
- Request one utility invoice template and one marketing template.

**Deliverable:** Meta test configuration and template submission.

**Checkpoint:** credentials exist outside source control and the webhook requirements are understood.

#### Day 5 — Write the integration proof plan

**Tasks:**

- Define the exact test: one Tally invoice, one test customer, one invoice message.
- Define expected inputs and outputs.
- Define failure cases: missing number, duplicate invoice, invalid token and WhatsApp rejection.
- Draw the final data flow.
- Create Git repository and make the first commit.

**Deliverable:** integration proof checklist.

**Checkpoint:** do not continue if the Tally field mapping or WhatsApp account is still unknown.

### Week 2 — Build the Flask foundation

#### Day 6 — Create the Flask application factory

**Tasks:**

- Create `app/__init__.py`.
- Add `create_app(config_object=None)`.
- Add separate development, test and production configuration.
- Add `/health` and `/ready` endpoints.
- Add a basic test client test.

**Deliverable:** Flask server starts through the application factory.

**Checkpoint:** `pytest` passes and `/health` returns HTTP 200.

#### Day 7 — Add database configuration

**Tasks:**

- Add SQLAlchemy extension setup.
- Add Alembic migration configuration.
- Add local SQLite configuration and PostgreSQL configuration.
- Add a simple database connection test.
- Add a safe startup error if `DATABASE_URL` is missing in production.

**Deliverable:** database connection and first migration.

**Checkpoint:** migration can be applied to a clean local database.

#### Day 8 — Add logging and error handling

**Tasks:**

- Add structured application logging.
- Add centralized JSON error responses for API routes.
- Add request correlation IDs.
- Redact access tokens and full phone numbers from logs.
- Add tests for validation and server errors.

**Deliverable:** consistent errors and PII-safe logs.

**Checkpoint:** intentionally invalid requests return useful errors without stack traces or secrets.

#### Day 9 — Add authentication and roles

**Tasks:**

- Create user model and password hashing.
- Add login/logout.
- Add admin and marketing roles.
- Add secure session cookie configuration.
- Add login rate limiting.
- Add an admin creation script.

**Deliverable:** protected dashboard route.

**Checkpoint:** unauthenticated users cannot access invoice or campaign screens.

#### Day 10 — Add base web screens

**Tasks:**

- Add a simple responsive layout.
- Add dashboard navigation.
- Add login page.
- Add sync-run summary placeholder.
- Add invoice and campaign placeholder pages.
- Add accessible labels, keyboard focus and clear error messages.

**Deliverable:** navigable skeleton application.

**Checkpoint:** Playwright smoke test can log in and open the dashboard.

### Week 3 — Implement Tally nightly synchronization

#### Day 11 — Create Tally data models and DTOs

**Tasks:**

- Define typed invoice and customer DTOs.
- Define `SyncRun` and `SyncItem` models.
- Define invoice status values.
- Add database constraints for Tally company plus voucher/reference.
- Add migrations.

**Deliverable:** persistence model for sync data.

**Checkpoint:** duplicate Tally references are rejected by the database.

#### Day 12 — Implement phone normalization

**Tasks:**

- Implement E.164 normalization for Indian numbers.
- Reject short, long or malformed values.
- Add tests for `9876543210`, `+919876543210`, spaces, dashes and invalid values.
- Never log the full number in tests or production logs.

**Deliverable:** reusable phone validation module.

**Checkpoint:** valid numbers normalize to one canonical value and invalid numbers are reported clearly.

#### Day 13 — Implement Tally XML parser

**Tasks:**

- Parse the saved redacted XML fixture.
- Extract invoice number, voucher reference, date, customer name, phone and total.
- Handle missing optional fields.
- Raise typed parsing errors for missing required fields.
- Add unit tests for valid and malformed XML.

**Deliverable:** pure, tested Tally parser.

**Checkpoint:** parser tests pass without connecting to Tally.

#### Day 14 — Implement the local Tally client

**Tasks:**

- Build the local HTTP/XML client with timeout handling.
- Add configurable Tally URL and company name.
- Add bounded retries for temporary connection failures.
- Add `health-check` command.
- Test using a mocked Tally server.

**Deliverable:** local client can retrieve a date-range response.

**Checkpoint:** a closed Tally instance produces a clear recoverable error.

#### Day 15 — Build the nightly connector CLI

**Tasks:**

- Add `python -m connector sync --from DATE --to DATE`.
- Read invoices from Tally.
- Build a sync report.
- Add dry-run mode.
- Return exit code 0 only when all required items succeed.
- Add tests for empty days and partial failures.

**Deliverable:** local connector runs against fixtures without AWS.

**Checkpoint:** running the same fixture twice produces zero duplicate invoices.

### Week 4 — Connect AWS application and invoice records

#### Day 16 — Build the sync API

**Tasks:**

- Add authenticated `POST /api/v1/sync-runs`.
- Validate connector payloads with Pydantic.
- Create sync-run and item records.
- Return per-invoice success/failure results.
- Add request size and rate limits.

**Deliverable:** connector can upload a dry-run payload to Flask.

**Checkpoint:** invalid connector tokens and malformed payloads are rejected.

#### Day 17 — Add customer and invoice services

**Tasks:**

- Implement create-or-update customer logic.
- Use normalized phone number as the matching key, with Tally ledger reference as supporting data.
- Insert invoices using idempotency keys.
- Store `needs_attention` for missing/invalid phone numbers.
- Add service-level tests.

**Deliverable:** uploaded records appear in the database.

**Checkpoint:** repeated upload is safe and does not create duplicate invoices.

#### Day 18 — Implement invoice listing and detail screens

**Tasks:**

- Add invoice search by date, number, customer and status.
- Add sync-run summary screen.
- Add missing-number and failed-item views.
- Add invoice detail page.
- Add pagination for future growth.

**Deliverable:** staff can inspect the results of the nightly sync.

**Checkpoint:** a staff member can identify all invoices requiring attention.

#### Day 19 — Implement PDF storage path

**Tasks:**

- Select the tested Tally PDF strategy.
- Add local PDF discovery by invoice number if using an export folder.
- Validate file type and size.
- Upload to S3-compatible storage or local development storage.
- Generate expiring download URLs.

**Deliverable:** one invoice PDF is securely associated with one invoice record.

**Checkpoint:** a user cannot download another invoice by changing a URL.

#### Day 20 — Complete the first end-to-end dry run

**Tasks:**

- Create one test bill in Tally.
- Run the connector locally.
- Upload the sync data to Flask.
- Verify customer, invoice, status and PDF.
- Repeat the run and verify no duplicates.
- Record every failure and fix only the MVP blockers.

**Deliverable:** complete Tally-to-AWS invoice record flow.

**Checkpoint:** do not begin WhatsApp sending until this flow is repeatable and idempotent.

### Week 5 — Implement WhatsApp invoice delivery

#### Day 21 — Build the WhatsApp API client

**Tasks:**

- Implement typed request/response models.
- Add document upload and template send methods.
- Add timeouts and safe retry classification.
- Add redacted request logging.
- Mock Meta responses in tests.

**Deliverable:** isolated, tested WhatsApp client.

**Checkpoint:** tests prove permanent Meta errors are not retried indefinitely.

#### Day 22 — Add invoice sending service

**Tasks:**

- Validate invoice status and phone number before sending.
- Send the approved utility template and PDF.
- Create a `message_log` before the request.
- Save the provider message ID after success.
- Mark failures with actionable reasons.

**Deliverable:** invoice can be sent from a service call.

**Checkpoint:** duplicate send is blocked unless an explicit authorized resend is requested.

#### Day 23 — Add delivery webhook

**Tasks:**

- Implement Meta GET verification endpoint.
- Implement POST event parsing.
- Match provider message IDs.
- Update sent, delivered, read and failed states.
- Make webhook processing idempotent.

**Deliverable:** delivery status changes automatically in the database.

**Checkpoint:** replaying the same webhook event does not corrupt timestamps or create records.

#### Day 24 — Add invoice send controls

**Tasks:**

- Add “Send invoice” and authorized “Resend” actions.
- Add status badges.
- Add failure reason display.
- Add confirmation before resend.
- Add audit log entries.

**Deliverable:** admin can safely retry a failed invoice.

**Checkpoint:** a user cannot resend invoices without the required permission.

#### Day 25 — Perform controlled WhatsApp test

**Tasks:**

- Use one approved test template and one controlled number.
- Send one real invoice PDF.
- Verify received content, number, invoice and PDF.
- Verify delivery/read webhook events.
- Test an invalid number and a rejected message.

**Deliverable:** verified real WhatsApp invoice delivery.

**Checkpoint:** obtain business-owner approval before sending to real customers.

### Week 6 — Consent, campaigns and production readiness

#### Day 26 — Implement consent and opt-out

**Tasks:**

- Add consent fields and consent event history.
- Add billing/customer consent screen.
- Add manual opt-out action.
- Parse STOP/UNSUBSCRIBE webhook messages.
- Add tests proving opted-out customers are excluded.

**Deliverable:** auditable consent system.

**Checkpoint:** an opted-out customer cannot be selected for marketing.

#### Day 27 — Build campaign recipient selection

**Tasks:**

- Add campaign and recipient snapshot models.
- Add customer search and filters.
- Preview count before sending.
- Re-check consent immediately before each send.
- Remove duplicate numbers from a campaign.

**Deliverable:** campaign draft and recipient preview.

**Checkpoint:** recipient count is explainable and all recipients have valid consent.

#### Day 28 — Build campaign sending and pause controls

**Tasks:**

- Add approved template selection.
- Add batch sending with a safe rate limit.
- Add pause and resume states.
- Add campaign message logs.
- Add delivery report.

**Deliverable:** one small campaign can be run safely.

**Checkpoint:** pause prevents new messages from being sent.

#### Day 29 — Deploy and schedule nightly sync

**Tasks:**

- Build Docker image.
- Deploy Flask app and worker to AWS.
- Configure PostgreSQL, storage, HTTPS and firewall.
- Configure Meta webhook URL.
- Install connector on the Tally Windows computer.
- Configure Windows Task Scheduler for a nightly run.

**Deliverable:** production-like staging environment.

**Checkpoint:** the connector makes outbound HTTPS calls and no Tally port is publicly exposed.

#### Day 30 — Production acceptance and handover

**Tasks:**

- Run the complete staged flow with test data.
- Test backup and restore.
- Run Ruff, Black check, MyPy/Pyright and pytest.
- Run Playwright smoke tests.
- Test Tally closed, AWS unavailable, Meta unavailable and duplicate sync cases.
- Write staff operating steps for nightly sync and failed records.
- Start with one real invoice and a very small approved campaign.

**Deliverable:** production acceptance checklist and handover.

**Checkpoint:** launch only when all success criteria in Section 17 pass.

## 21. Daily Definition of Done

A day is complete only when:

- The planned code/configuration is committed.
- At least one happy-path test exists.
- At least one relevant failure/edge-case test exists.
- The application still starts from a clean environment.
- No secret or customer PII was committed.
- The day's checkpoint passed.
- The next blocker is written down clearly.

## 22. If a Day Takes Longer Than Expected

Do not extend the day's scope indefinitely. Use this order:

1. Fix the smallest blocker required for the day's checkpoint.
2. Move optional polish to the backlog.
3. Add a failing test that captures the remaining problem.
4. Continue the next day only if the dependency is stable.
5. If Tally field mapping, PDF export or WhatsApp access is blocked, stop coding dependent features and resolve that external dependency first.

The schedule is intentionally sequential because Tally field mapping, PDF handling and WhatsApp approval are prerequisites for reliable implementation.
