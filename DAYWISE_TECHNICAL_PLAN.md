# Day-Wise Technical Implementation Plan

## Working Assumption

- One developer
- 3–4 hours per day
- Python 3.12+
- Flask
- PostgreSQL
- SQLAlchemy and Alembic
- Official Meta WhatsApp Cloud API
- Existing TallyPrime installation
- Nightly synchronization from the Tally computer
- AWS deployment

## Project Structure to Build

```text
showroom-platform/
├── app/
│   ├── __init__.py
│   ├── config.py
│   ├── extensions.py
│   ├── common/
│   ├── auth/
│   ├── customers/
│   ├── invoices/
│   ├── tally/
│   ├── whatsapp/
│   ├── campaigns/
│   └── jobs/
├── connector/
├── migrations/
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
├── scripts/
├── Dockerfile
├── docker-compose.yml
├── pyproject.toml
└── .env.example
```

---

## Day 1 — Python Project Setup

- Install Python 3.12+.
- Create the project directory.
- Create and activate a virtual environment.
- Create `pyproject.toml`.
- Install Flask, SQLAlchemy, Alembic, Pydantic, HTTPX, pytest, Ruff, Black and MyPy.
- Create the project folders.
- Add `.gitignore`.
- Add `.env.example`.
- Add a minimal `README.md` with local startup commands.
- Initialize Git.

## Day 2 — Flask Application Factory

- Create `app/__init__.py`.
- Implement `create_app()`.
- Add development, test and production configuration classes.
- Add Flask health endpoint: `GET /health`.
- Add readiness endpoint: `GET /ready`.
- Add the first Flask test client test.
- Configure Flask secret key from environment variables.

## Day 3 — Extensions and Database

- Create `app/extensions.py`.
- Configure SQLAlchemy.
- Configure Flask-Migrate/Alembic.
- Add local SQLite configuration.
- Add PostgreSQL configuration.
- Create the first migration.
- Add database connection tests.

## Day 4 — Logging and Error Handling

- Create `app/common/errors.py`.
- Create `app/common/logging.py`.
- Add structured application logging.
- Add request correlation IDs.
- Add centralized 400, 401, 403, 404 and 500 handlers.
- Redact API tokens and phone numbers from logs.
- Add tests for invalid requests and server errors.

## Day 5 — User Authentication

- Create the user model.
- Add password hashing with Argon2 or bcrypt.
- Add login and logout routes.
- Add secure session cookies.
- Add admin and marketing roles.
- Add authentication decorators/helpers.
- Add login tests.

## Day 6 — Authentication Interface

- Create login page.
- Create base layout.
- Add protected dashboard route.
- Add logout action.
- Add unauthorized and forbidden pages.
- Add login rate limiting.
- Add Playwright login smoke test.

## Day 7 — Customer Model

- Create the `customers` table.
- Add UUID primary key.
- Add name, phone, email and language fields.
- Add marketing consent fields.
- Add opt-out fields.
- Add Tally ledger reference.
- Add timestamps.
- Create migration.
- Add model tests.

## Day 8 — Phone Number Module

- Create phone normalization module.
- Convert Indian numbers to E.164 format.
- Support values with spaces, dashes and country code.
- Reject invalid numbers.
- Add unit tests for valid numbers.
- Add unit tests for malformed, empty and duplicate numbers.

## Day 9 — Invoice Model

- Create the `invoices` table.
- Add customer relationship.
- Add Tally company and voucher fields.
- Add unique Tally reference.
- Add invoice number, date and amount.
- Add PDF object key.
- Add invoice status values.
- Create migration.
- Add duplicate-invoice tests.

## Day 10 — Sync Models

- Create `sync_runs` table.
- Create `sync_items` table.
- Add sync status values.
- Add error message and retry fields.
- Add sync date range fields.
- Add database relationships.
- Add migrations.
- Add tests for successful and failed sync records.

## Day 11 — Tally XML Fixtures

- Add redacted Tally XML response under `tests/fixtures/`.
- Add fixtures for a valid invoice.
- Add fixture for a missing phone number.
- Add fixture for malformed XML.
- Add fixture for a cancelled invoice.
- Document the XML field paths in code comments.

## Day 12 — Tally XML Parser

- Create `app/tally/parser.py`.
- Create typed invoice DTOs.
- Parse invoice number.
- Parse voucher number.
- Parse invoice date.
- Parse customer name.
- Parse phone number.
- Parse total amount.
- Parse cancellation status.
- Add parser unit tests.

## Day 13 — Tally HTTP Client

- Create `app/tally/client.py`.
- Configure Tally local URL.
- Build XML request payload.
- Add HTTP timeout.
- Add connection error handling.
- Add bounded retry for temporary errors.
- Add response status validation.
- Mock Tally HTTP responses in tests.

## Day 14 — Connector Configuration

- Create `connector/config.py`.
- Load Tally URL from environment variables.
- Load AWS API URL from environment variables.
- Load connector token from environment variables.
- Add date and timezone configuration.
- Add safe configuration validation.
- Add `connector health-check` command.

## Day 15 — Connector CLI

- Create `connector/cli.py`.
- Implement `python -m connector sync`.
- Add `--from` date argument.
- Add `--to` date argument.
- Add default previous-day behavior.
- Add `--dry-run` mode.
- Add console and file logging.
- Add CLI tests.

## Day 16 — Connector Invoice Exporter

- Create `connector/exporter.py`.
- Request invoices for a date range.
- Parse Tally XML through the parser.
- Normalize phone numbers.
- Validate required invoice fields.
- Create a per-invoice export result.
- Continue processing after one invalid invoice.
- Return a final summary.

## Day 17 — AWS Sync API

- Create connector authentication middleware.
- Add `POST /api/v1/sync-runs`.
- Validate connector request body with Pydantic.
- Validate the connector token.
- Add request size limits.
- Create sync-run records.
- Return per-invoice results.
- Add API tests.

## Day 18 — Customer and Invoice Import Service

- Create customer create-or-update service.
- Match customers using normalized phone number.
- Create invoices using Tally idempotency key.
- Handle duplicate invoices as skipped records.
- Store invalid records as attention items.
- Add transaction boundaries.
- Add service tests.

## Day 19 — Connector Uploader

- Create `connector/uploader.py`.
- Upload sync data over HTTPS.
- Add authorization header.
- Add idempotency key for sync runs.
- Add request timeout.
- Add bounded retries.
- Save local JSON sync report.
- Return non-zero exit code when required items fail.

## Day 20 — End-to-End Tally Sync

- Run the connector against Tally or the Tally mock server.
- Upload one real-shaped invoice.
- Verify customer creation.
- Verify invoice creation.
- Run the same sync twice.
- Confirm no duplicate customer or invoice is created.
- Test missing and invalid phone numbers.
- Test Tally connection failure.

## Day 21 — Invoice and Sync Dashboard

- Add invoice list route.
- Add invoice search by number and customer.
- Add invoice date filter.
- Add invoice status filter.
- Add sync-run list route.
- Add sync-run detail route.
- Add failed-item display.
- Add pagination.
- Add browser smoke tests.

## Day 22 — Invoice PDF Storage

- Select local development storage and production object storage.
- Validate PDF file extension and MIME type.
- Validate maximum PDF size.
- Prevent path traversal.
- Upload invoice PDF.
- Store object key in the invoice record.
- Generate expiring download URL.
- Add storage tests.

## Day 23 — Tally PDF Matching

- Implement local invoice PDF folder configuration.
- Match PDFs by invoice number or voucher number.
- Handle missing PDF files.
- Handle duplicate PDF matches.
- Add PDF upload to the sync process.
- Add tests for matching, missing and duplicate files.

## Day 24 — WhatsApp Client

- Create `app/whatsapp/client.py`.
- Configure Meta API URL and version.
- Configure access token and phone number ID.
- Implement template message request.
- Implement document upload request.
- Implement document message request.
- Add timeout handling.
- Add retry classification.
- Mock Meta responses in tests.

## Day 25 — WhatsApp Message Logs

- Create `message_logs` table.
- Add invoice relationship.
- Add campaign relationship field.
- Add message type.
- Add provider message ID.
- Add sent, delivered, read and failed states.
- Add failure code and reason.
- Create migration.

## Day 26 — Invoice WhatsApp Service

- Create invoice sending service.
- Validate invoice status before sending.
- Validate customer phone number.
- Validate PDF availability.
- Send approved invoice template.
- Attach invoice PDF.
- Store provider message ID.
- Mark send failures.
- Add service tests.

## Day 27 — WhatsApp Webhook

- Add webhook verification route.
- Add webhook event route.
- Validate Meta webhook payload.
- Match provider message IDs.
- Update delivery status.
- Store delivery timestamps.
- Ignore duplicate webhook events safely.
- Add webhook tests.

## Day 28 — Invoice Send Interface

- Add invoice detail send button.
- Add resend action for failed messages.
- Add resend authorization check.
- Add confirmation dialog.
- Display sent, delivered, read and failed status.
- Display safe failure reason.
- Add audit event for sending and resending.
- Add Playwright tests.

## Day 29 — Real Invoice Delivery Test

- Configure Meta webhook URL in the test environment.
- Use one approved invoice template.
- Use one controlled WhatsApp test number.
- Send one invoice PDF.
- Verify the recipient number.
- Verify the invoice PDF.
- Verify delivery webhook.
- Verify read webhook.
- Test invalid-number failure.

## Day 30 — Consent Data Model

- Add `consent_events` table.
- Add opt-in and opt-out event types.
- Add consent source.
- Add consent timestamp.
- Add customer consent service.
- Add manual opt-in route.
- Add manual opt-out route.
- Add consent history query.
- Add migration and tests.

## Day 31 — Consent Interface

- Add marketing consent checkbox to customer screen.
- Add current consent status display.
- Add opt-out action.
- Add consent history display.
- Add permission checks.
- Add audit logging.
- Add browser tests.

## Day 32 — WhatsApp Opt-Out Processing

- Parse inbound WhatsApp text messages.
- Normalize STOP and UNSUBSCRIBE values.
- Match the sender phone number.
- Set marketing consent to false.
- Set opt-out timestamp.
- Create opt-out event.
- Prevent future marketing sends.
- Add tests for supported and unsupported text values.

## Day 33 — Campaign Models

- Create `campaigns` table.
- Add campaign status values.
- Add template name.
- Add creator and schedule fields.
- Create campaign recipient snapshot table.
- Add campaign message relationships.
- Create migration.
- Add model tests.

## Day 34 — Customer Segmentation

- Add customer search service.
- Filter by marketing consent.
- Filter by purchase date.
- Filter by product or invoice attributes if available.
- Exclude opted-out customers.
- Exclude invalid numbers.
- Remove duplicate numbers.
- Add recipient-count tests.

## Day 35 — Campaign Creation Interface

- Add create campaign route.
- Add campaign name input.
- Add approved template selector.
- Add customer filter controls.
- Add recipient preview.
- Add campaign save action.
- Add validation errors.
- Add Playwright tests.

## Day 36 — Campaign Sending Service

- Implement campaign recipient snapshot.
- Re-check consent before every send.
- Send approved marketing template.
- Create message log per recipient.
- Add batch size configuration.
- Add safe rate limiting.
- Add transient failure retry.
- Add permanent failure handling.

## Day 37 — Campaign Controls

- Add campaign start action.
- Add pause action.
- Add resume action.
- Add status transitions.
- Prevent two workers from processing one recipient.
- Add campaign progress count.
- Add campaign failure count.
- Add service tests.

## Day 38 — Campaign Reports

- Add campaign list page.
- Add campaign detail page.
- Add recipient counts.
- Add sent count.
- Add delivered count.
- Add read count.
- Add failed count.
- Add export of failure reasons.
- Add browser tests.

## Day 39 — Background Worker

- Create database-backed jobs table.
- Add invoice message jobs.
- Add campaign message jobs.
- Add job status and retry count.
- Add worker process.
- Add atomic job claiming.
- Add graceful shutdown.
- Add worker tests.

## Day 40 — Full Local Integration

- Start Flask application.
- Start database.
- Start worker.
- Start mock Tally server.
- Run the connector.
- Import invoices.
- Process invoice jobs.
- Process webhook events.
- Create and send a test campaign.
- Verify all database statuses.

## Day 41 — API and Browser Security

- Add CSRF protection.
- Add secure cookie flags.
- Add security headers.
- Restrict CORS origins.
- Add API rate limits.
- Validate file downloads.
- Validate all form and JSON payloads.
- Test unauthorized routes.
- Test role restrictions.

## Day 42 — Database Reliability

- Add transaction boundaries to services.
- Add unique constraints.
- Add indexes for phone, invoice date and statuses.
- Add database connection pooling.
- Add migration rollback test.
- Add database backup command.
- Add restore test using a local database.

## Day 43 — Dockerization

- Create production Dockerfile.
- Create local `docker-compose.yml`.
- Add Flask web service.
- Add worker service.
- Add PostgreSQL service.
- Add health checks.
- Add non-root container user.
- Run the complete application through Docker.

## Day 44 — AWS Server Deployment

- Create AWS server.
- Configure security group.
- Install Docker.
- Configure environment variables securely.
- Deploy web container.
- Deploy worker container.
- Configure persistent database storage.
- Configure persistent invoice storage.
- Test health endpoint.

## Day 45 — HTTPS and Domain

- Connect domain DNS.
- Configure HTTPS certificate.
- Redirect HTTP to HTTPS.
- Configure secure webhook URL.
- Test API from the Tally computer.
- Test dashboard login remotely.
- Verify Tally is not publicly exposed.

## Day 46 — Windows Connector Installation

- Copy connector to the Tally computer.
- Create a connector virtual environment.
- Install connector dependencies.
- Configure connector environment variables.
- Run `health-check`.
- Run dry-run synchronization.
- Confirm local logs are created.
- Confirm Tally remains accessible only locally.

## Day 47 — Windows Nightly Scheduler

- Create a Windows Task Scheduler job.
- Configure the nightly execution time.
- Configure the working directory.
- Configure the Python executable.
- Configure the sync date range.
- Capture standard output and errors.
- Run the scheduled task manually.
- Verify the AWS sync-run record.

## Day 48 — Failure Recovery

- Stop Tally and run the connector.
- Disconnect the Tally computer network temporarily.
- Stop the AWS application.
- Simulate WhatsApp API timeout.
- Simulate invalid phone number.
- Simulate missing invoice PDF.
- Verify retries and error statuses.
- Re-run failed sync safely.

## Day 49 — Performance and Data Checks

- Test 300 invoice records locally.
- Test repeated synchronization.
- Test campaign recipient selection with sample data.
- Check database query times.
- Add missing indexes.
- Check memory usage of the worker.
- Check invoice PDF storage behavior.
- Verify no duplicate messages are created.

## Day 50 — Production Test and Cleanup

- Run Ruff.
- Run Black check.
- Run MyPy or Pyright.
- Run all pytest tests.
- Run Playwright smoke tests.
- Remove debug routes and test credentials.
- Review environment variables.
- Review logs for exposed PII.
- Apply all pending migrations.
- Run one controlled production invoice.
- Run one small approved marketing campaign.

---

## Commands to Run Regularly

```bash
# Activate environment
.venv\Scripts\activate

# Start local Flask server
flask --app app run --debug

# Run migrations
alembic upgrade head

# Run tests
pytest

# Run tests with coverage
pytest --cov=app --cov=connector

# Format code
black app connector tests

# Lint code
ruff check app connector tests

# Type checking
mypy app connector

# Run connector dry-run
python -m connector sync --from 2026-09-24 --to 2026-09-24 --dry-run

# Run connector health check
python -m connector health-check
```

## Technical Completion Criteria

- Tally invoices can be read for a selected date range.
- Customer mobile numbers are normalized and validated.
- Duplicate Tally invoices are not inserted twice.
- Invoice PDFs are stored securely.
- Invoice messages are sent through the official WhatsApp API.
- WhatsApp delivery statuses update through webhooks.
- Marketing consent is stored and audited.
- Opted-out customers are excluded immediately.
- Campaigns can be previewed, started and paused.
- Failed jobs can be retried safely.
- The nightly connector runs through Windows Task Scheduler.
- The AWS deployment uses HTTPS.
- Tally is never exposed to the public internet.
- Tests, linting and type checks pass before launch.
