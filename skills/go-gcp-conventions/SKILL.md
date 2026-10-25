---
name: go-gcp-conventions
description: Conventions for Go web apps on Google App Engine (or Cloud Run) that use Firestore, or PostgreSQL/MySQL on Cloud SQL, with Cloud Storage and Secret Manager (stdlib net/http, html/template, no framework). Use when creating or changing handlers, Firestore or SQL reads/writes, migrations, admin areas, caching, redirects, cron jobs or deployment in a Go project on Google Cloud.
---

# Go + Google Cloud app conventions

Distilled from a production Go content site on App Engine with Firestore, Cloud Storage and Secret Manager, generalised so it applies to similar apps. Follow these when working in a Go project on Google Cloud, and check the existing code first: where a project already does something differently, match the project. Function and type names below (`server`, `pageCache`, `requireAdmin`, `loadSecretOrEnv`) describe patterns to reproduce, not a library to import.

## Stack and layout

- Standard library only for HTTP: `http.NewServeMux`, `html/template`. No router or framework.
- Cloud libraries: `cloud.google.com/go/firestore`, `cloud.google.com/go/storage`, `cloud.google.com/go/secretmanager`, `google.golang.org/api/iterator`.
- Runtime is App Engine standard (`app.yaml`, `runtime: go1xx`, `instance_class: F1`). Dev loop is `air` (`.air.toml`, rebuild on `go`/`html` changes).
- Layout: package `main` at the repo root holds handlers, split by concern (`main.go`, `admin.go`, `media.go`, `redirects.go`). Shared Firestore types go in `internal/models`. Background integrations go in `services/`. Both can import `models`; nothing can import `main`, which is why shared types never live there.
- One `server` struct holds every dependency (Firestore client, GCS client, parsed templates, page cache, admin config, redirect table). Handlers are methods on `*server`. No globals except constants and read-only config.
- Constants (TTLs, limits, base URL) go in one `const` block at the top of `main.go`, not as magic numbers inside handlers.

## Firestore data

- **Document ID is the natural key** (for posts, the slug). Set `post.Slug = doc.Ref.ID` after decoding; do not store the ID as a field.
- **Field naming: tag every field explicitly.** Untagged Go structs store PascalCase field names, which is an accident of Go (only exported, capitalised fields are stored, and the stored name defaults to the Go name), not a convention. Use `firestore:"..."` tags with camelCase if Node or other clients may ever read the data, otherwise snake_case, and say which in a comment. Never mix styles within a collection. Never rename fields of a live collection without migrating every document. Raw writes (REST, scripts, `map[string]interface{}`) must use the exact stored names.
- Decode with `doc.DataTo(&v)`. On a decode error, log it and `continue` in list loops, and return 500 for a single-document page.
- Writes from forms build a `map[string]interface{}` with explicit stored field names and call `.Set`. Convert `template.HTML` fields with `string(...)` before writing.
- Times are `time.Time` in the struct and Firestore timestamps in storage. Never store dates as strings.
- Shared document types that more than one package reads or writes live in `internal/models`, and the other package uses a type alias (`type RateData = models.Rate`) rather than a parallel struct, so the two can't drift.
- Iterate with `iter := ...Documents(ctx)`, `defer iter.Stop()`, and loop on `iter.Next()` until `iterator.Done`. Wrap non-Done errors with context (`fmt.Errorf("read redirects: %w", err)`).
- `GetAll()` reads every document and bills for each one. Fine for tiny collections, but do not use it just to count a large one.

### Queries and indexes

- An equality filter combined with `OrderBy` on another field needs a **composite index**. Document the required index in the comment on the function that needs it (`// requires a composite index (Category asc, Pinned asc, PublishedAt desc)`).
- Prefer filtering in Go (`visiblePosts`, skipping `Hidden` and the current slug) over adding a Firestore filter that needs another index. Accept that a filtered item can use up a `Limit` slot, and say so in a comment.
- A `!=` filter on the document ID forces ordering by `__name__` first. Avoid it when the query must order by something else.
- Fetch a small buffer (`limit+1`) when you will exclude the current item client-side.
- Independent homepage queries run concurrently with a `sync.WaitGroup`, each goroutine writing a distinct field of one result struct. On a query error, log and return what was collected (or nil). A failing widget must not take the page down.

## Handlers

- Handlers are small: check method and path, try the cache, fetch, render to a buffer, cache, write. Exact-path routes begin with `if r.URL.Path != "/x/" { http.NotFound(...) }` because `"/"` patterns match subtrees.
- Log the real error with `log.Printf("<what> error: %v", err)` and send a generic `http.Error(w, "internal server error", 500)`. Never send err text to the client.
- Render templates into a `bytes.Buffer` via `ExecuteTemplate(&buf, "base", data)` so a template failure returns a clean 500 instead of a half-written page. Set `Content-Type` explicitly (`text/html; charset=utf-8`).
- Templates: one `template.New("base.html").Funcs(funcMap).ParseFiles("templates/base.html", "templates/<page>.html")` per page, parsed once at startup and stored on the server. Parse failure is `log.Fatalf` at startup. Admin pages use a separate `admin_base.html` layout.
- Sitewide values a template needs (analytics ID, build version) are exposed through `funcMap`, not copied into every page's data struct.
- Rich-text fields stored as HTML are typed `template.HTML` so they render unescaped. Only admin-authored or code-authored content may use it. Strip tags (`stripHTMLTags`) before putting such a field in an attribute such as `<meta content>`.
- A static-asset cache-buster (`buildVersion` = process start time, appended as `?v=`) is needed because `app.yaml` gives `/static` a long `expiration`.

## In-memory page cache

- `pageCache`: `map[string]cacheEntry` guarded by `sync.RWMutex`, with a fixed TTL (1h), keyed by route (`"home"`, `"category/<category>"`, `"<category>/<slug>"`, `"sitemap.xml"`).
- Methods: `get`, `set`, `invalidate`, `count`, `clear`. It is per-instance memory, so other instances stay stale until the TTL expires. That is accepted; say so rather than pretending it is shared.
- Any write path must invalidate every key that could show the changed data. Keep that in one helper (`invalidatePostCache`) and call it from each save. When a field such as the category can change, invalidate all candidate keys, not just the current one.
- Pages with no Firestore data do not need the cache (the contact page renders fresh so template edits show immediately). Note why in the handler comment.

## Admin area

- All admin routes sit under one unguessable prefix and are wrapped with `s.requireAdmin(...)`. Register specific patterns (`/prefix/dashboard`) alongside the subtree pattern (`/prefix/`); the most specific wins.
- Auth: PIN login, HMAC-SHA256 signed session cookie (`email|expiry`, base64url), `HttpOnly`, `SameSite=Strict`, `Secure` when the request is HTTPS (check `r.TLS` or `X-Forwarded-Proto`), cookie `Path` scoped to the admin prefix. Sessions slide: `requireAdmin` reissues the cookie on every request. Compare PINs with `subtle.ConstantTimeCompare` and signatures with `hmac.Equal`.
- Brute-force limit: per-client counter (`X-Forwarded-For` first hop, else `RemoteAddr`) with lockout (5 attempts, 15 min).
- CSRF: token = HMAC of `"csrf|"+email`, derived from identity, not the cookie value, so it survives the sliding-session cookie refresh. Every state-changing admin handler must be POST-only and call `validCSRF` (form field `csrf_token` or header `X-CSRF-Token`) before doing anything.
- Admin action handlers share a shape: `noStore(w)`, require POST, check CSRF, do the work, log `"admin <action>: OK — ..."` or `FAILED`, then `303 See Other` back to the dashboard with a status query param (`?cache_cleared=1&cache_flushed=3`) that the dashboard template turns into a banner. Put the redirect-with-querystring boilerplate in one helper rather than copy-pasting it across handlers.
- Validate user-supplied slugs with a regexp (`^[a-zA-Z0-9][a-zA-Z0-9-]{0,199}$`) before using them as document IDs or object paths. Fall back to a safe default (`"misc"`) rather than trusting input.
- Set `Cache-Control: no-store` on every admin response.

## Secrets and config

- Never commit secret values to `app.yaml`. `app.yaml` holds only the *resource path* (`ADMIN_PIN_SECRET: projects/<p>/secrets/admin-pin/versions/latest`). `loadSecretOrEnv(ctx, "<X>_SECRET", "<X>")` reads from Secret Manager when that path is set and otherwise falls back to a plain env var for local dev.
- If a required secret is missing in dev, warn and degrade (random per-process session secret, login disabled). If one is configured but cannot be read, `log.Fatalf` so a bad deploy fails loudly.
- `GOOGLE_CLOUD_PROJECT` comes from the environment. Warn if unset. Credentials are Application Default Credentials locally (`gcloud auth application-default login`) and automatic on App Engine. Key files (`*-key.json`, `service-account-key.json`) are gitignored.
- Use `envOrDefault(key, fallback)` for optional config. `PORT` defaults to 8080.

## Cloud Storage uploads

- Default App Engine bucket: `<project>.appspot.com`. Objects go under `uploads/<slug>/<unixnano>-<rand>.<ext>`.
- The App Engine filesystem is read-only at runtime, so user uploads must go to GCS rather than `static/`.
- Cap the body with `http.MaxBytesReader` (8 MiB) before `ParseMultipartForm`. Sniff the type from the first 512 bytes, never from the client's filename or header, and allowlist content types. Go's `http.DetectContentType` does not recognise AVIF, so check the ISO-BMFF `ftyp` brand manually.
- Writer settings: `ContentType`, `CacheControl = "public, max-age=2592000"`, `PredefinedACL = "publicRead"`. Close the writer and check its error. A failed `Close` means the upload failed.
- Respond with JSON `{"url": "https://storage.googleapis.com/<bucket>/<object>"}`.

## Redirects from Firestore

- A `redirects` collection (`OldPath`, `NewPath`, `StatusCode`, `Disabled`) is loaded wholesale into an in-memory table. The load happens at startup, from an admin reload button, and on a 10-minute background ticker so other instances catch up.
- Build the new map first and swap it in under the write lock. On a Firestore error, keep the old table.
- Validate on load, skip and log (don't fail): path must start with `/` and not `//`, target must be a site path or the site's own absolute URL (no open redirects), status in {301, 302, 307, 308}, no self-redirects. Collapse chains (A→B→C becomes A→C) up to a hop limit and drop loops.
- Normalise paths (lowercase, trim trailing slash) on both load and lookup.
- Middleware only handles GET/HEAD, runs ahead of the mux, and overrides a live page at the same path.
- Apex/www canonicalisation is a separate outer middleware (`redirectWWW`) with a 301.

## Cron and background jobs

- App Engine cron (`cron.yaml`) hits a normal route. That route **must** check `r.Header.Get("X-Appengine-Cron") == "true"` and return 403 otherwise (App Engine strips this header from external requests).
- Provide a separate admin-authenticated route (POST + CSRF) for the same job, so it can be run manually. The admin version also flushes the whole page cache. The cron version only invalidates the keys it affected.
- The job code itself lives in `services/`, takes `(ctx, *firestore.Client)`, and uses an `http.Client` with an explicit timeout. Writing all documents in one batch with fixed document IDs makes re-runs overwrite rather than duplicate. Never change an ID that is live in production.
- When an upstream API fails, fall back to last-known benchmark values and log a WARNING, so the collection is never left without an entry. Say in comments which values are live and which are benchmark.

## Deploy

- `gcloud app deploy app.yaml --project=<id>`. `cloudbuild.yaml` runs the same with the `gcr.io/cloud-builders/gcloud` image and `logging: CLOUD_LOGGING_ONLY`.
- Static files: `static_dir` handler with `expiration: "30d"` and `secure: always`; everything else `script: auto`, `secure: always`.
- Deploying composite indexes is a separate step (`firebase deploy --only firestore:indexes` or `gcloud firestore indexes composite create`). A query that needs a missing index fails at runtime with an error containing a creation link, so test every new query against a real project before deploying.

## Cloud Run variant

The default above is App Engine. When the user asks for Cloud Run, keep every code pattern in this skill (handlers, Firestore access, admin auth and CSRF, redirects, caching, uploads) and swap only the platform pieces below. Do not generate `app.yaml`, `cron.yaml` or `static_dir` handlers.

- **Deploy:** a multi-stage `Dockerfile` (`golang` build stage with `CGO_ENABLED=0`, then `gcr.io/distroless/static`), or buildpacks via `gcloud run deploy <service> --source . --region <region>`. The binary reads relative paths (`templates/`, `static/`), so `COPY` those directories next to it and set `WORKDIR`. `PORT` is injected, and `":"+port` already works. `cloudbuild.yaml` becomes build, push to Artifact Registry, `gcloud run deploy`.
- **Credentials:** attach a dedicated service account to the service. Grant it `roles/datastore.user`, `roles/storage.objectAdmin` on the media bucket only, and `roles/secretmanager.secretAccessor` on the specific secrets. Application Default Credentials work automatically, so `firestore.NewClient` and `storage.NewClient` need no change.
- **Secrets:** inject them as environment variables with `gcloud run deploy ... --set-secrets=ADMIN_PIN=admin-pin:latest,ADMIN_SESSION_SECRET=admin-session-secret:latest`. `loadSecretOrEnv` already falls back to the plain variables (`ADMIN_PIN`, `ADMIN_SESSION_SECRET`), so just don't set the `*_SECRET` path variables and the Secret Manager client code is never used. Non-secret settings use `--set-env-vars`.
- **Cron:** replace `cron.yaml` and the `X-Appengine-Cron` check with a Cloud Scheduler HTTP job that sends an OIDC token (`--oidc-service-account-email`, `--oidc-token-audience=<service URL>`). A public site cannot rely on IAM to protect the endpoint, so the handler must verify the bearer token: `idtoken.Validate(ctx, token, audience)` from `google.golang.org/api/idtoken`, and check the token's email is the scheduler service account. Return 403 otherwise. Keep the separate admin POST + CSRF route unchanged.
- **Media bucket:** there is no default bucket. Create one (`gcloud storage buckets create gs://<project>-media --location=<region> --uniform-bucket-level-access`) and read its name from an env var instead of `projectID + ".appspot.com"`. With uniform bucket-level access, `PredefinedACL = "publicRead"` fails, so remove it and grant `allUsers` the `roles/storage.objectViewer` role on the bucket (or serve through a CDN) instead.
- **Static files:** already served by `http.FileServer`. Set `Cache-Control` in code to replace the `expiration: "30d"` that `app.yaml` provided, and keep the `buildVersion` cache-buster.
- **Background work:** by default Cloud Run only gives CPU during a request. `refreshRedirectsLoop` and any other goroutine ticker will stall between requests. Either deploy with `--no-cpu-throttling`, or replace the ticker with a lazy refresh (reload when `loadedAt` is older than the interval, checked on request).
- **Scaling and state:** instances scale to zero by default, so the page cache, redirect table and login limiter are per instance and start empty after a cold start. Set `--min-instances=1` if cold starts matter and accept that cost. Concurrency (default 80) makes the `sync.RWMutex` guards on shared maps essential.
- **Networking:** HTTPS is enforced by the platform (no `secure: always`). Custom domains use a domain mapping or a load balancer, and the apex/www redirect middleware still applies. A fixed outbound IP or private VPC access needs Direct VPC egress or a VPC connector configured on the service.
- **Logs:** stdout/stderr go to Cloud Logging as before. Nothing to change.

## SQL database variant (Cloud SQL: PostgreSQL or MySQL)

**Not derived from a production project.** The source app uses Firestore only, so this section is general practice applied to the same architecture and has not been battle-tested. Verify driver APIs against current documentation, and replace guesses with what a real SQL project does. When the user asks for PostgreSQL or MySQL, keep every other pattern in this skill (handlers, cache, admin auth and CSRF, redirects, uploads, cron, deploy) and swap only the data layer below.

### Choosing

- Firestore suits document-shaped, read-heavy, per-tenant content with few relations (an article site, a CMS). SQL suits relational data, joins, reporting, strict constraints and ledgers (invoices, payments, stock).
- Default to PostgreSQL. Use MySQL when the user asks or an existing system requires it. Using both Firestore and SQL in one app is fine (content in Firestore, ledger in SQL), but never write the same fact to both without a clear owner for each.

### Packages and layout

- Standard `database/sql` with a driver: PostgreSQL `github.com/jackc/pgx/v5/stdlib` (or `pgxpool` directly), MySQL `github.com/go-sql-driver/mysql`. Prefer plain SQL, optionally generated with `sqlc`, over an ORM, so every query is visible.
- Put all SQL in a `store/` (or `repository/`) package with one type per aggregate. Handlers call services/stores, never `*sql.DB`. Put the `*sql.DB` on the `server` struct beside `fsClient`, closed with `defer db.Close()`.
- Schema changes are versioned migrations in `migrations/` (`golang-migrate` or `goose`), forward-only, committed with the code that needs them. Run them as a separate deploy step, not on app startup when more than one instance can start at once.

### Connecting to Cloud SQL

- Prefer the Cloud SQL Go Connector (`cloud.google.com/go/cloudsqlconn`) with IAM or password auth, or the Unix socket at `/cloudsql/<project>:<region>:<instance>`. On Cloud Run enable it with `--add-cloudsql-instances`; on App Engine use `beta_settings.cloud_sql_instances` in `app.yaml`. Grant the service account `roles/cloudsql.client`. Locally use the Cloud SQL Auth Proxy or a Docker container.
- The password is a secret: Secret Manager (`--set-secrets`), read through the existing `loadSecretOrEnv`. Never put a DSN with a password in the repo.
- **Size the pool against the database.** Each instance opens its own pool, so `SetMaxOpenConns × max instances` must stay below the instance's `max_connections`. Use small pools (5 to 10), `SetMaxIdleConns`, and `SetConnMaxLifetime` (about 30 minutes) so stale connections are recycled. Cap `--max-instances` accordingly.

### Queries and data

- **Parameterised queries only.** Placeholders are `$1, $2` on PostgreSQL and `?` on MySQL. Never build SQL with `fmt.Sprintf` or string concatenation of user input. Dynamic `ORDER BY` columns come from an allowlist.
- Pass `r.Context()` (or a derived context with a timeout) to every call: `QueryContext`, `QueryRowContext`, `ExecContext`. Always `defer rows.Close()` and check `rows.Err()`.
- Transactions: `tx, err := db.BeginTx(ctx, nil)`, `defer tx.Rollback()` immediately, `tx.Commit()` last. Sequential numbers (invoice numbers) use a counter row read with `SELECT ... FOR UPDATE` inside a transaction, or a database sequence. Never read-then-write outside one.
- Types: store time as UTC (`timestamptz` on PostgreSQL; `DATETIME` or `TIMESTAMP` on MySQL with the connection parameter `parseTime=true&loc=UTC`). Money is integer minor units (pence) or `NUMERIC`/`DECIMAL`, never float. Use `sql.NullString`/`sql.NullTime` or pointer fields for nullable columns. MySQL tables use `utf8mb4`.
- Columns are `snake_case`, with `id`, `created_at`, `updated_at` on every table. Map to the JSON/template names in the Go struct tags. IDs: `bigint generated always as identity` (PostgreSQL) or `BIGINT AUTO_INCREMENT` (MySQL), or UUIDs where IDs are exposed publicly.
- Multi-tenant apps put `tenant_id` (NOT NULL, foreign key) on every business table, first in every composite index, and in the `WHERE` of every query. The tenant comes from the session, never the request.
- Constraints belong in the schema (NOT NULL, UNIQUE, CHECK, foreign keys), not only in Go. Translate violations into friendly errors: unique violation is `*pgconn.PgError` code `23505` on PostgreSQL and `*mysql.MySQLError` number `1062` on MySQL.
- Add an index for every foreign key and every filter/sort used by a list query, and check slow queries with `EXPLAIN`. Paginate with keyset (`WHERE (created_at, id) < ($1, $2) ORDER BY ... LIMIT n`) rather than large `OFFSET`s. Avoid N+1: join or fetch with `WHERE id = ANY($1)` / `IN (...)`.
- Soft delete with `archived_at`/`deleted_at` where records are referenced (invoices, customers), and hard delete only when nothing references the row.

### PostgreSQL and MySQL differences to watch

| Topic | PostgreSQL | MySQL (InnoDB) |
|---|---|---|
| Placeholders | `$1` | `?` |
| Insert and return ID | `INSERT ... RETURNING id` | `res.LastInsertId()` |
| Upsert | `ON CONFLICT (...) DO UPDATE` | `ON DUPLICATE KEY UPDATE` |
| JSON column | `jsonb` (indexable) | `JSON` |
| Default isolation | READ COMMITTED | REPEATABLE READ |
| Text comparison | case-sensitive by default | case-insensitive by default collation |
| Identifier quoting | `"name"` | `` `name` `` |

### Sessions, cache and the rest

- The in-memory page cache, signed admin sessions and redirect table are unchanged. If redirects move to SQL, load them wholesale into the same swapped-in map.
- Keep the Cloud Run rules: pools and caches are per instance, and background tickers need CPU allocated.

### Testing

- Pure helpers are tested as before. Store code is tested against a real database in a container (`testcontainers-go`, or `docker compose` locally), applying the real migrations to a fresh schema per test run, and skipped when `TEST_DATABASE_URL` is unset. Avoid mocking `database/sql`: mocks don't catch wrong SQL.
- Back up Cloud SQL (automated backups and point-in-time recovery enabled), and test restoring one before relying on it.

## Testing

- `go vet . && go test -race -count=1 .` before committing. Tests sit beside the code (`admin_test.go`, `redirects_test.go`, ...) in package `main`, and `.air.toml` already excludes `_test.go` from rebuilds.
- Table-driven tests with `t.Run` for pure helpers: slug validation, redirect path normalising and target validation, truncation, tag stripping, image sniffing, token signing.
- Build a `&server{...}` with only the fields the handler touches (`admin`, `cache`, `redirects`) and drive it with `httptest`. Handlers that need Firestore are not unit-tested this way.
- Security behaviour is tested as rejections: tampered, expired and wrong-secret session tokens; missing or foreign CSRF tokens; GET on POST-only admin routes; open-redirect targets; non-image uploads. A rejected request must leave state (the cache) untouched.
- Write the assertion at the level the code controls. For example, assert that an mp4 is not reported as AVIF and is not in the allowlist, rather than the exact string the standard library returns.
- Firestore-backed code (redirect loading, list queries, the sitemap, the admin save, sync jobs) is best tested against the Firestore emulator, which needs Java (`gcloud emulators firestore start`, then `FIRESTORE_EMULATOR_HOST=127.0.0.1:8686`). Skip those tests when the variable is unset, use a `demo-` project ID, clear collections before and after each test, and seed with the stored field names (never via a model struct that carries a `Slug`/ID field, which would store it). The emulator does not enforce composite indexes.

## Code style

- Comments explain why, not what, and are long where the reason is non-obvious (cache tradeoffs, index requirements, security choices). Doc comments start with the function name.
- Errors: log with a stable prefix (`"redirects: ..."`, `"admin upload ..."`), wrap with `%w` when returning, and skip bad items instead of failing a whole load.
- Pass `r.Context()` into Firestore calls so cancelled requests stop. Use `context.Background()` only for startup and long-lived loops.
- Keep the in-progress files you were not asked to touch out of commits. Check `git status` before staging.

## Pitfalls to avoid

- Unit tests that only cover pure helpers and HTTP behaviour leave Firestore queries and indexes unchecked until runtime. Add emulator tests for the queries that matter.
- Describing required composite indexes only in comments is fragile. Track them in `firestore.indexes.json` and deploy them with the code.
- Counting documents with `GetAll()` reads every document. Use an aggregation count query (`.NewAggregationQuery().WithCount(...)`) for larger collections.
- Copy-pasting the redirect/CSRF boilerplate across admin handlers invites mistakes. Use one helper.
- Do not hardcode an admin email or any identity default in code. Read it from configuration.
