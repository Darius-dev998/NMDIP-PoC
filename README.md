# National Malaria Data Integration Platform (NMDIP)

**Interoperability · Data quality · Surveillance · Analytics**

NMDIP is a Ministry of Health–style proof of concept for connecting facility EMRs, laboratory systems and a national malaria repository. It demonstrates a role-protected ASP.NET Core API, PostgreSQL persistence, validation/transformation/synchronization workflows, audit events, governed FHIR R4 resources (Patient, Observation, DiagnosticReport, Organization, CapabilityStatement) with versioned terminology bindings, a delivery observability outbox, and a React operations dashboard.

> **Demonstration boundary:** the dashboard is intentionally populated with synthetic, fixed demonstration data. Its “Run sync” action updates the local demo view only. It does not submit cases to the API or to DHIS2. The API and database schema are a separate integration foundation and must be configured before handling real health data.

## Architecture

```text
React + TypeScript (Vite)
        │ bearer JWT / JSON API
        ▼
ASP.NET Core 8 Web API
  Controllers → Integration service → EF Core
        │
        ▼
Supabase PostgreSQL (server-side credentials only)
        │
        └── future DHIS2 adapter / tracked entities / events / data elements
```

The frontend includes an offline demonstration mode and an optional **Connect API** sign-in flow. When connected, patient list, patient registry counts (based on the API response, capped at 500), registration, laboratory-result reads and entry, and saved-case reads use the API. A patient's displayed case result is the latest associated laboratory result; it remains Pending when no test result has been recorded. The browser restores a still-valid API session after refresh by checking `GET /api/v1/auth/me`. Dashboard metrics and most monitoring screens remain synthetic. Without an API connection, those module views use clearly labeled demo records. Patient review status is not implemented. Saved case reports are not a live DHIS2 feed; the API does not yet include a production DHIS2 connector or durable hosted queue. Do not interpret demo UI activity as live integration monitoring.

Patient registration provides a searchable ISO 3166-1 country list. For compatibility with the current schema, the selected country is stored in the existing patient `County` field; split country and administrative-area fields in a reviewed schema migration before collecting production data.

## Project structure

```text
src/                         React dashboard and interactive demo modules
backend/src/Nmdip.Api/       ASP.NET Core 8 API
Database/Supabase/           Supabase schema script
```

## Run the frontend

Requirements: Node.js 20.19+ or 22.12+.

```powershell
npm install
npm run dev
```

Open the Vite URL (normally `http://localhost:5173`). The demo includes overview, patient registry, laboratory, integration pipeline, NMDR repository, data quality, operations, audit, CSV validation and FHIR preview screens. Without an API connection, patient registration and search are local demo interactions. CSV import validates the required patient ID, county, facility, age and result columns and reports row errors in the browser; approval is a local demonstration only and does not persist imported records.

To connect the patient registry and registration form to a running API, start the API in Development on port 5129, then use **Connect API** in the application header. The frontend signs in using JWT and loads/creates patient records through the API. For deployment, set `VITE_API_BASE_URL` to the API's `/api/v1` URL at frontend build time. The dashboard indicators, charts, operations view and CSV validation remain clearly labeled synthetic/demo functionality.

## Configure the API and Supabase

1. Create a Supabase project. Supabase creates and manages its PostgreSQL database; you do not need to issue `CREATE DATABASE`.
2. In Supabase **SQL Editor → New query**, paste the contents of [`Database/Supabase/001_schema.sql`](Database/Supabase/001_schema.sql), then select **Run**. The final queries list the eight expected tables and confirm that RLS is enabled.
3. In Supabase, click **Connect** (top of the project dashboard), choose the **URI** format, and copy the PostgreSQL URI. A URI such as `postgresql://postgres:[YOUR-PASSWORD]@db.<project-ref>.supabase.co:5432/postgres` is a template: `[YOUR-PASSWORD]` is a placeholder, not your actual database password. Use the **Session pooler** option if your network cannot connect to the direct database host.
4. Run the local setup script. Paste the URI at its hidden prompt. If it contains `[YOUR-PASSWORD]`, the script prompts separately for the actual database password, then saves the assembled connection string to the ignored .NET User Secrets store and creates random local JWT/bootstrap secrets. It displays the bootstrap key once; keep it private for the next step:

```powershell
.\scripts\setup-local-supabase.ps1
```

5. Start the API in Development and open Swagger:

```powershell
dotnet run --project backend/src/Nmdip.Api
```

Swagger is available at `http://localhost:5129/swagger`. Check `http://localhost:5129/health/ready`; it returns Ready when the API can connect to the database. The frontend's development proxy is configured for port 5129. Start the frontend in another terminal with `npm run dev`, then use **Connect API** in the dashboard.

6. In Swagger, call `POST /api/v1/auth/bootstrap` once to create your first Administrator. Use the bootstrap key printed by the setup script and choose your own administrator name, email, and unique password of at least 12 characters. After a successful response, remove the one-time bootstrap key:

```powershell
dotnet user-secrets remove "Bootstrap:Key" --project backend/src/Nmdip.Api
```

Sign into the application using **Connect API**. Administrators can add other role accounts via `POST /api/v1/users`.

Do not use the checked-in Development JWT placeholder outside a private local machine; the setup script replaces it in local User Secrets. Production settings should be supplied through the hosting provider's secret manager:

```text
ConnectionStrings__Nmdip=<Supabase PostgreSQL connection string>
Jwt__SigningKey=<random secret of at least 32 bytes>
Jwt__Issuer=NMDIP
Jwt__Audience=NMDIP.Client
Cors__AllowedOrigins__0=https://<your-frontend-domain>
Bootstrap__Key=<one-time random secret of at least 32 characters>
```

Use a server-side Supabase/PostgreSQL connection only. Do not put the database password, secret/service-role key or JWT signing key in Vite variables or browser code. The schema enables RLS and intentionally defines no anonymous/authenticated browser policies; database access is through the trusted API. The setup script writes credentials only to the local .NET User Secrets file under the current Windows user profile.

The Supabase project URL and publishable key are intended for public client configuration, but the current NMDIP app does not use the Supabase browser SDK. It authenticates to the ASP.NET API using the NMDIP JWT, and the API connects to PostgreSQL using its server-side connection string. Do not use the publishable key as a PostgreSQL password, and never expose a `service_role`/secret key.

### First administrator

With the database schema applied and `Bootstrap__Key` configured, make one request to `POST /api/v1/auth/bootstrap`:

```json
{
  "bootstrapKey": "<the configured one-time key>",
  "fullName": "National Data Administrator",
  "email": "admin@example.org",
  "password": "<unique password of at least 12 characters>"
}
```

This creates the first Administrator only, stores the password as PBKDF2-SHA256, and refuses bootstrap once any user exists. Remove `Bootstrap__Key` after initialization. Administrators can create role accounts through `POST /api/v1/users`. Roles are `Administrator`, `Data Manager`, `Lab Officer` and `Clinician`.

## API surface

All routes are versioned under `/api/v1`. Clinical, audit and operations endpoints require a bearer token; see `POST /auth/login`. Read/write endpoints enforce role claims as documented by their controller attributes.

| Area | Routes |
|---|---|
| Authentication and user roles | `POST /auth/bootstrap`, `POST /auth/login`, `POST /auth/change-password`, `GET/POST /users` |
| Patient registry | `GET/POST /patients`, `GET/PUT/DELETE /patients/{id}` |
| Laboratory | `GET/POST /labresults`, `GET/PUT /labresults/{id}` |
| Integration hub | `POST /integration/validate`, `POST /integration/sync`, `POST /integration/retry?failedId=...`, `GET /integration/status`, `GET /integration/transactions`, `GET /integration/failed` |
| NMDR case repository | `GET/POST /cases`, `GET /cases/{id}` |
| FHIR resources | `GET /fhir/metadata`, `GET /fhir/patient/{id}`, `GET /fhir/patient` (search), `GET /fhir/observation/{id}`, `GET /fhir/diagnostic-report/{id}`, `GET /fhir/organization/{id}` — `application/fhir+json` content negotiation |
| Operations and audit | `GET /operations/status`, `GET /quality/summary`, `GET /quality/issues`, `GET /audit?search=...&since=...` |

Patient, laboratory, case, audit and transaction collections use bounded pagination. Clients can send `page` and `pageSize`; page sizes are limited to 100. Bulk import APIs, full-text search and per-facility authorization are not yet implemented.

Authenticated users can change their own password through the connected app's profile button or `POST /api/v1/auth/change-password`, providing their current password and a different new password of at least 12 characters. In Development, Swagger's **Authorize** button accepts the bearer token returned by `POST /api/v1/auth/login`. The app signs out after a password change; other access tokens remain valid until their normal 30-minute expiry.

## Integration flow

1. A source submits a case payload to `POST /integration/validate`.
2. Validation checks required patient/county/facility/result fields, age bounds, allowed result values and duplicate patient/date pairs.
3. `POST /integration/sync` canonicalizes result values and writes a case report, transaction record, delivery-outbox entry and audit event (with a correlation ID shared by the transaction, case report, failed-sync row and audit record) in PostgreSQL.
4. Failed sync attempts are persisted to `FailedSyncQueue` with the original JSON payload, stable idempotency key, retry timestamps and dead-letter handling. Automatic and manual retries replay that exact payload.
5. External delivery status is observable through the `DeliveryOutbox` table and the case report's `Dhis2Status` column. The case store is a DHIS2-style foundation; configure the actual DHIS2 URL, authentication, organisation-unit/data-element mappings, retry strategy and acceptance tests before deployment.
6. FHIR resources are built from persisted API data (no fixed synthetic examples) and carry profile references under `https://nmdr.health.go.ke/fhir`. Terminology bindings (LOINC, SNOMED CT, ICD-10, administrative gender) are versioned; see `docs/MAPPINGS.md`.

Example sync payload:

```json
{
  "patientId": "00000000-0000-0000-0000-000000000001",
  "county": "Kisumu",
  "facility": "Jaramogi Oginga Odinga Teaching and Referral Hospital",
  "result": "positive",
  "resultDate": "2025-06-18T10:42:18Z",
  "gender": "F",
  "age": 28,
  "sourceSystem": "Afya EMR"
}
```

## Operational and data-governance notes

- Use only synthetic data in demos and non-production environments.
- Before live use, complete a privacy impact assessment, lawful-basis and retention review, threat model, security testing, incident response/runbooks, backup/restore test and disaster recovery plan.
- Add consent/legal basis and patient-identity/linkage controls before joining data across source systems. This pilot uses a UUID patient key and does not perform probabilistic matching.
- Configure TLS, managed secrets, database least privilege, network restrictions, backups, monitoring and alerting at deployment.
- Define and version the national terminology, county/facility organisation-unit codes, test codes, DHIS2 data elements and FHIR profiles with the relevant national governance teams.
- Supabase SQL uses quoted PascalCase table/column names to match EF Core conventions. Keep future schema changes in reviewed, ordered SQL migrations.

## Build and checks

```powershell
npm run build
npm run lint
npm test
dotnet build backend/src/Nmdip.Api/Nmdip.Api.csproj
dotnet test backend/tests/Nmdip.Api.Tests/Nmdip.Api.Tests.csproj
```

## Operations and governance documentation

- [`docs/OPERATIONS.md`](docs/OPERATIONS.md) — monitoring, alerting, retry/dead-letter workflow, backup and restore verification, incident response, deployment.
- [`docs/PRIVACY.md`](docs/PRIVACY.md) — data inventory, lawful basis, retention schedule, security-testing checklist.
- [`docs/MAPPINGS.md`](docs/MAPPINGS.md) — facility, county, terminology and DHIS2 mappings.

## Tests

- Backend: xUnit + `WebApplicationFactory` against an isolated in-memory SQLite database, covering authentication, authorization, validation, persistence, retries with the original payload, duplicate submission producing one logical case, FHIR content negotiation and audit correlation IDs.
- Frontend: Vitest + Testing Library covering the API client contract.

Schema changes are versioned in `Database/Supabase/` (`001_schema.sql`, `002_governance.sql`, ...) and registered in the `SchemaVersion` table.
