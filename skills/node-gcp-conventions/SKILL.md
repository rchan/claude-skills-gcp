---
name: node-gcp-conventions
description: Conventions for Node.js (Express + Nunjucks) web apps on Google App Engine (or Cloud Run) that use Firestore through firebase-admin, or PostgreSQL/MySQL on Cloud SQL, with Firebase Auth, Stripe webhooks and per-customer workspaces. Use when creating or changing routes, services, middleware, Firestore or SQL reads/writes, migrations, indexes, webhooks, cron endpoints or deployment in a Node project on Google Cloud.
---

# Node + Google Cloud app conventions

Distilled from a production multi-tenant Node.js app (Express, Nunjucks, Firestore, Stripe) on App Engine, generalised so it applies to similar apps. Check the existing code first and match it where it differs. This skill covers the server side only: front-end build tooling (Tailwind, bundlers) and page templates are separate concerns and are not covered here.

For the Go equivalent see the `go-gcp-conventions` skill. The architecture is the same (thin handlers, one place that touches Firestore, secrets out of config), but the data model differs: this app is multi-tenant.

## Stack and layout

- Express 5, Nunjucks (`autoescape: true`), `firebase-admin` for Firestore and Auth, `express-session` stored in Firestore (`@google-cloud/connect-firestore`), `helmet`, `compression`, `express-rate-limit`. Runtime `nodejs24` on App Engine standard; `engines.node` pinned in `package.json`.
- Layout: `server.js` (wiring only), `routes/` (one router per feature), `services/` (all Firestore and third-party access), `middleware/`, `config/app-config.js`, `utils/` (pure helpers such as formatting, CSV, PDF), `views/`, `scripts/` (migrations, one-off fixes, security tests).
- `services/firebase-admin.js` initialises the SDK exactly once (guarded by `admin.apps.length`), sets `db.settings({ ignoreUndefinedProperties: true })`, and exports `{ admin, db, auth }`. Everything else imports from it. Never call `initializeApp` elsewhere.
- Services are classes exported as a single instance (`module.exports = new CustomerService()`), with JSDoc on each method. Routes never call `db` directly. If two services need each other, `require` inside the method to avoid a circular import at load time (and comment why).
- Mount order in `server.js` matters and is commented where it matters: webhook routes before body parsing (Stripe needs the raw body), public routes (`/pay`, `/portal`, `/admin`) before the catch-all `/` router that applies `requireAuth` + `checkWorkspace` to everything.

## Data model

- **Multi-tenant by workspace.** Business data lives in subcollections under the tenant: `workspaces/{workspaceId}/customers|orders|items|...`. Top-level collections are only for things that are not tenant data (`users`, `workspaces`, `sessions`, `adminAccessLog`, verification codes).
- **Every service method takes `workspaceId` as its first argument** and builds its path from it. The workspace ID comes from the server-side session through `checkWorkspace` (`req.workspace.id`), never from the request body, query or URL. `checkWorkspace` re-verifies on each request that the user still has access to the session's workspace.
- Documents are returned as `{ id: doc.id, ...doc.data() }`, and `null` when `!doc.exists`. The ID is not stored as a field.
- Field names are camelCase. Every document gets `createdAt` and `updatedAt` as `admin.firestore.Timestamp.now()` (or `Timestamp.fromDate(date)` for user-supplied dates). Do not store dates as strings or JS `Date` (a few older writes do; convert with `.toDate()` when reading).
- Normalise input in the service: defaults for optional strings (`|| ''`), booleans from form strings (`!== 'false'`), `parseInt(...) || default`, nested objects such as `address: { line1, line2, city, ... }`.
- **Soft delete first.** Deleting sets `archived: true`. A second delete hard-deletes only if nothing references the document (`isCustomerUsed`), otherwise throws a user-readable error. Queries treat a missing `archived` field as false by filtering in memory, so old documents without the field still work.
- Remove a field with `admin.firestore.FieldValue.delete()`. Append to an array with `FieldValue.arrayUnion(...)`.
- **Sequential numbers use a transaction.** Order and invoice numbers come from a counter on the workspace document, incremented in `db.runTransaction`, with the year stored alongside so it resets each calendar year. First use seeds the counter from the highest existing number. Never compute "last number + 1" outside a transaction.
- Use `db.batch()` for multi-document writes that must succeed together.

### Queries and indexes

- Composite indexes are tracked in `firestore.indexes.json` and deployed with `firebase deploy --only firestore:indexes` (`firebase.json` points at it). Whenever a query combines `where` and `orderBy` on different fields, add the index to that file in the same change.
- Always `.limit()` list queries that can grow (the default is 100). Order on the server, filter on the server where an index exists, and filter in memory only for flags that old documents may lack.
- Avoid N+1 reads inside loops (a `getCustomers` that queries orders per customer). For a new list view, fetch the related documents once with a single query and group them in memory.

## Routes and middleware

- A router starts with `router.use(requireAuth); router.use(checkWorkspace);` unless it is deliberately public. Public routes authenticate by an unguessable token in the URL (`/pay/:token`) and say so in a comment.
- `requireAuth` distinguishes API from page requests (`req.xhr`, `Accept: application/json`, JSON content type): JSON callers get `401 { error }`, browsers get a redirect to login with the original path in `redirect`.
- Handler shape: `try { validate → service call → res.json({ success: true, ... }) / res.render(...) } catch (error) { console.error('<Action> error:', error); res.status(500).json({ error: 'Failed to <action>' }) }`. Log the real error, return a generic message. Business-rule errors thrown by a service on purpose (`'This customer cannot be deleted because...'`) may be returned with a 400.
- Validate types, not just presence (`typeof body.name !== 'string'`), then `trim()` and `slice(0, N)` free text before it reaches a service. Body size limits are set globally (`512kb`).
- JSON responses use `{ success: true, ... }` or `{ error: '...' }`. Page failures send a short plain message or render `404.html` / `500.html`. In production the error page receives `{}`, never the error object.
- Templates get `title` and `workspace` explicitly. Shared filters (`currency`, `date`, `dateISO`) live in `server.js` and handle both Firestore Timestamps (`.toDate()`) and plain dates.

## Security patterns

- Helmet with a **per-request CSP nonce** (`res.locals.nonce`), created before helmet runs. Sessions: `httpOnly`, `secure` in production, `trust proxy` set for App Engine, 7-day lifetime, stored in Firestore, `saveUninitialized: false`.
- `express-rate-limit` on login, magic-link and other guessable endpoints.
- Compare secrets with `crypto.timingSafeEqual` (check lengths first). An unset secret **disables** the feature (magic login, cron endpoints return 500) rather than leaving it open.
- Admin-only pages answer **404, not 403**, to non-admins (allowlist of emails from env). Sensitive admin entry points write an audit record (`adminAccessLog`: ip, user agent, success, timestamp) on every attempt, fire-and-forget with a `.catch` that logs.
- Stripe webhook: `express.raw({ type: 'application/json' })` on that route only, verify with `stripe.webhooks.constructEvent` and the webhook secret, return 400 on failure. Take trusted values (workspace and order IDs) from the event's `metadata` / `client_reference_id`, never from the browser.
- Cron endpoints (`/webhook/process-recurring`, `/webhook/process-trial-expiry`) are POST-only and require an `x-api-key` header equal to `CRON_SECRET`. An external scheduler calls them.
- Throttle background writes that happen on every request (`lastActiveAt` is updated at most every 15 minutes, fire-and-forget).

## Secrets and config

- Read every secret from `process.env` and have **no fallback value in code**. Missing required config should throw at startup or disable the feature, not silently use a default.
- Do not put secret values in `app.yaml`. Use Secret Manager (as the `go-gcp-conventions` skill does: store the resource path in `app.yaml`, read the value at startup) or a deploy-time secret. `app.yaml` may hold non-secret settings only (`NODE_ENV`, `APP_DOMAIN`, feature flags).
- Keep separate environments as separate files (`app.yaml` / `app-live.yaml`, `cloudbuild.yaml` / `cloudbuild-live.yaml`) and per-environment Firebase projects. Service-account key files, `.env` and `cookies.txt` are gitignored and never committed.
- Use `process.env.X || default` only for harmless defaults (port, domain).

## Testing

- `scripts/security/` is a black-box suite run against a live server: `node scripts/security/run-all.js http://localhost:3000` (auth, authorization/IDOR, input validation, webhooks/cron, business logic). Exit code 0 passes, 1 fails, 2 crashed. Run it against a local or test instance, never production. When you add a route, add a case for its auth check and for cross-workspace access.
- There are no unit tests. For new pure logic (tax maths, number formatting, payment status calculations), add Node's built-in runner (`node --test`) tests next to it. Small verification scripts that recompute dashboard totals from raw data are a good way to check calculations against real data.
- One-off data fixes and migrations go in `scripts/` as standalone files that are safe to re-run (idempotent), with a header comment saying what they change.

## Deploy

- `npm run deploy` builds the CSS and runs `gcloud app deploy`. `cloudbuild.yaml` runs `gcloud app deploy app.yaml` with `logging: CLOUD_LOGGING_ONLY`.
- Static files via the `/public` `static_dir` handler with `secure: always`; everything else `script: auto`.
- Deploy indexes and rules separately from the app (`firebase deploy --only firestore:indexes,firestore:rules`).
- A Serverless VPC connector is used when the app must reach a fixed-IP relay required by a third-party API. Declare it in `app.yaml` and say why in the code that depends on it.

## Cloud Run variant

The default above is App Engine. When the user asks for Cloud Run, keep every code pattern in this skill (workspace data model, services, middleware, security, webhooks) and swap only the platform pieces below. Do not generate `app.yaml` or `static_dir` handlers.

- **Deploy:** a `Dockerfile` (`node:24-slim`, `npm ci --omit=dev`, `COPY` the app, `CMD ["node", "server.js"]`) or buildpacks via `gcloud run deploy <service> --source . --region <region>`. Build the CSS in the image build or in CI, not at runtime. `server.js` must listen on `process.env.PORT` (it already does). `cloudbuild.yaml` becomes build, push to Artifact Registry, `gcloud run deploy`. Keep separate services for test and live.
- **Credentials:** attach a dedicated service account with only the roles needed (`roles/datastore.user`, `roles/storage.objectAdmin` on the media bucket, `roles/secretmanager.secretAccessor` on named secrets). `admin.credential.applicationDefault()` works with no key file. Delete the key-file branch of `firebase-admin.js` and never commit a key.
- **Secrets:** `gcloud run deploy ... --set-secrets=STRIPE_SECRET_KEY=stripe-secret-key:latest,SESSION_SECRET=session-secret:latest`. Code keeps reading `process.env.X`, so nothing else changes. Non-secret settings use `--set-env-vars`. Remove secrets from any YAML in the repo.
- **Cron:** replace the shared `x-api-key` with a Cloud Scheduler HTTP job that sends an OIDC token (`--oidc-service-account-email`, `--oidc-token-audience=<service URL>`). The endpoint verifies it with `google-auth-library` (`new OAuth2Client().verifyIdToken({ idToken, audience })`) and checks the token's email is the scheduler service account. Keep the 500-if-unconfigured behaviour. The Stripe webhook path is unchanged.
- **Storage buckets:** create buckets explicitly (`--uniform-bucket-level-access`). With uniform access, per-object public ACLs fail, so grant `roles/storage.objectViewer` to `allUsers` on a public bucket, or return signed URLs for private files.
- **Sessions:** the Firestore session store works unchanged and is the right choice, since instances are stateless and scale up and down. Keep `app.set('trust proxy', 1)` and `cookie.secure` in production.
- **State and scaling:** instances scale to zero, so in-memory rate limiters (`express-rate-limit` default store), caches and counters are per instance and reset on cold start. For limits that must hold across instances, use a shared store (Firestore or Redis). `--min-instances=1` avoids cold starts at a cost. Background timers (`setInterval`) only run while CPU is allocated, so use `--no-cpu-throttling` or Cloud Scheduler instead.
- **Fire-and-forget writes:** a request's un-awaited promises (`lastActiveAt`, `adminAccessLog`) can be cut off once the response is sent when CPU is throttled. On Cloud Run, `await` writes you cannot afford to lose, or deploy with `--no-cpu-throttling`.
- **Networking:** HTTPS is enforced by the platform. For a fixed outbound IP (a third-party API that requires a fixed IP), use Direct VPC egress or a Serverless VPC connector configured on the service instead of `vpc_access_connector` in `app.yaml`. Custom domains use a domain mapping or load balancer.
- **Logs:** `console.log`/`console.error` go to Cloud Logging as before.

## SQL database variant (Cloud SQL: PostgreSQL or MySQL)

**Not derived from a production project.** The source app uses Firestore only, so this section is general practice applied to the same architecture and has not been battle-tested. Verify library APIs against current documentation, and replace guesses with what a real SQL project does. When the user asks for PostgreSQL or MySQL, keep every other pattern in this skill (services, middleware, security, webhooks, cron, deploy) and swap only the data layer below.

### Choosing

- Firestore suits document-shaped, per-tenant data with simple access patterns. SQL suits relational data, joins, reporting, strict constraints and ledgers. Reports, payments and document numbering are the kind of data that benefits from SQL's constraints and transactions.
- Default to PostgreSQL. Use MySQL when the user asks or an existing system requires it. Using both Firestore and SQL in one app is possible, but never write the same fact to both without a clear owner for each.

### Packages and layout

- PostgreSQL: `pg` (node-postgres) with a `Pool`. MySQL: `mysql2/promise` with a pool. Prefer plain parameterised SQL, or a thin query builder (Kysely, Knex), over a heavy ORM, so every query stays visible.
- Replace `services/firebase-admin.js` with `services/db.js` that creates the pool once and exports `query` and a `withTransaction(fn)` helper. Services keep the same shape (class instances taking `workspaceId` first) and are the only code that touches the pool. Routes never run SQL.
- Schema changes are versioned migrations (`node-pg-migrate`, Knex migrations or similar) in `migrations/`, forward-only, committed with the code that needs them, and run as a separate deploy step, not at app start when several instances can boot at once.

### Connecting to Cloud SQL

- Use the Cloud SQL Node connector (`@google-cloud/cloud-sql-connector`) or the Unix socket at `/cloudsql/<project>:<region>:<instance>`. On Cloud Run enable it with `--add-cloudsql-instances`; on App Engine use `beta_settings.cloud_sql_instances` in `app.yaml`. Grant the service account `roles/cloudsql.client`. Locally use the Cloud SQL Auth Proxy or a Docker container.
- The password is a secret: Secret Manager (`--set-secrets=DB_PASSWORD=...`), read from `process.env`. Never put it in `app.yaml`, a connection string in the repo, or a code fallback.
- **Size the pool against the database.** Each instance has its own pool, so `pool max × max instances` must stay below the database's `max_connections`. Use small pools (5 to 10), set `idleTimeoutMillis`/`maxLifetimeSeconds` where available, and cap `--max-instances` accordingly.
- Sessions: replace the Firestore session store with `connect-pg-simple` (PostgreSQL) or `express-mysql-session` (MySQL), keeping the same cookie settings.

### Queries and data

- **Parameterised queries only.** Placeholders are `$1, $2` on PostgreSQL and `?` on MySQL. Never build SQL by concatenating or interpolating user input into a template string. Dynamic `ORDER BY` columns come from an allowlist.
- Transactions: take a client from the pool, `BEGIN`, run the statements, `COMMIT`, `ROLLBACK` in `catch`, and always release the client in `finally`. Put this once in `withTransaction`. Sequential numbers (order or invoice numbers) use a counter row read with `SELECT ... FOR UPDATE` inside a transaction, or a database sequence. Never read-then-write outside one.
- **Type gotchas:** node-postgres returns `BIGINT` and `NUMERIC` as strings, and mysql2 returns `DECIMAL` as strings by default. Convert deliberately and store money as integer minor units or `NUMERIC`/`DECIMAL`, never JavaScript floats. Store time as UTC (`timestamptz` on PostgreSQL; `DATETIME`/`TIMESTAMP` on MySQL with the connection timezone set to UTC). MySQL tables use `utf8mb4`.
- Columns are `snake_case`, with `id`, `created_at`, `updated_at` on every table. Convert to camelCase at the service boundary so routes and templates keep camelCase field names.
- Multi-tenant apps put `workspace_id` (NOT NULL, foreign key) on every business table, first in every composite index, and in the `WHERE` of every query. The workspace comes from the session via `checkWorkspace`, never from the request.
- Constraints belong in the schema (NOT NULL, UNIQUE, CHECK, foreign keys), not only in JavaScript. Translate violations into friendly errors: unique violation is error code `23505` on PostgreSQL and `ER_DUP_ENTRY` (1062) on MySQL.
- Index every foreign key and every filter/sort used by a list query, and check slow queries with `EXPLAIN`. Paginate with keyset, not large `OFFSET`s. Avoid N+1: join, or fetch related rows with `WHERE id = ANY($1)` / `IN (?)`.
- Soft delete with `archived_at`/`deleted_at` where records are referenced (orders, customers), and hard delete only when nothing references the row.
- Webhook handlers keep their idempotency: record the Stripe event or session ID in a table with a UNIQUE constraint, and let the constraint reject repeats inside the same transaction as the payment update.

### PostgreSQL and MySQL differences to watch

| Topic | PostgreSQL | MySQL (InnoDB) |
|---|---|---|
| Placeholders | `$1` | `?` |
| Insert and return ID | `INSERT ... RETURNING id` | `result.insertId` |
| Upsert | `ON CONFLICT (...) DO UPDATE` | `ON DUPLICATE KEY UPDATE` |
| JSON column | `jsonb` (indexable) | `JSON` |
| Default isolation | READ COMMITTED | REPEATABLE READ |
| Text comparison | case-sensitive by default | case-insensitive by default collation |
| Identifier quoting | `"name"` | `` `name` `` |

### Testing and operations

- Service code is tested against a real database in a container (Testcontainers, or `docker compose` locally), applying the real migrations to a fresh schema, with `node --test` and skipped when `TEST_DATABASE_URL` is unset. Avoid mocking the driver: mocks don't catch wrong SQL.
- Add cross-workspace (IDOR) cases to the security suite: a user must not be able to read or change another workspace's rows by guessing an ID.
- Enable automated Cloud SQL backups and point-in-time recovery, and test a restore before relying on it.

## Pitfalls to avoid

- **Never commit secrets.** Deploy YAML, config fallbacks, `cookies.txt` and key files must not hold secret values. If one was committed, rotate the secret first, then remove it from the repo and history.
- **Make Firestore rules match the data model.** Rules that guard top-level collections while the data lives in tenant subcollections, or that allow public read/write on verification-code collections, give a false sense of protection. The Admin SDK bypasses rules, but anything using the client SDK does not. Write rules for the real paths, or deny all client access.
- **Make payment webhooks idempotent.** A handler that adds an amount to a record without checking whether that event or session was already applied can double-count on retries. Record the session ID and skip repeats.
- **Protect state-changing requests.** Set `sameSite` explicitly on the session cookie and add CSRF tokens (or rely on `sameSite: 'strict'` with care) for cookie-authenticated POSTs.
- **Restrict CORS.** `cors()` with defaults allows any origin. Limit it to the origins that need it.
- **Do not weaken the CSP.** `'unsafe-inline'` for scripts alongside a nonce defeats most of the protection.
- **Keep one `requireAuth`.** Two middleware files with the same name and different behaviour cause security bugs.
- **Do not leak error messages.** Return `error.message` to the client only for deliberate business errors, never for unexpected ones.
- **Avoid N+1 queries,** such as one related-record query per row in a list view.
- **Add unit tests** for pure logic, not only black-box security scripts.
