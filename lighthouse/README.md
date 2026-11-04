# Lighthouse

MARINA's authenticated document backend. Python 3.10+; one image, four roles:

> **Production deployment is blocked pending a separate runtime handoff.**
> The staged Fleet release now wires named ConfigMap/Secret runtime references
> and declares a temporary production suspension until dependencies are provisioned.
> Fleet's pre-install/pre-upgrade initialization hook blocks rollout on failure.
> Shipyard preserves Fleet-owned overrides across image promotion. Infrastructure/Fleet
> must first provision backing stores,
> wire named runtime ConfigMaps and local-only credential Secrets, execute the
> migration/storage initialization gate, configure the four `forwardauth`
> settings, and select/install embedding and generation models. No production
> rollout has occurred; production authentication/readiness stay fail-closed.

| Role | Mounted domain routes | Required runtime dependencies |
| --- | --- | --- |
| `knowledge-api` | Collections, documents, sources, search, questions | PostgreSQL, private original storage, Qdrant, Ollama embeddings/generation, auth |
| `knowledge-worker` | None; leased background ingestion/deletion | PostgreSQL, private original storage, Qdrant, Ollama embeddings, auth configuration |
| `agentic-rag` | Questions only | PostgreSQL, Qdrant, Ollama embeddings/generation, auth |
| `knowledge-learning` | None; scaffold only | None |

Every role preserves `/healthz`, `/ready`, `/v1/about`. Liveness does not check
dependencies. Readiness returns 503 with boolean checks for missing configuration,
unapplied migration, unavailable dependencies, absent exact model tags, invalid
vector dimensions, or a stalled worker. Learning reports `capability: scaffold`;
it is not a learning implementation. Runtime requires PostgreSQL; SQLite is only
used by deterministic tests.

Learning is optional in Compose (`docker compose --profile learning up -d`);
the default MVP startup runs only the other three roles.

Infrastructure owns live cluster storage, authentication, secrets, ingress and
runtime settings. This repository owns backend code, migrations and **local**
Compose. Cluster manifests belong to Infrastructure, not this repository.
See the [production runbook](PRODUCTION-RUNBOOK.md) for cross-repo configuration,
backup/restore/reindex, operations visibility, and the still-pending live gates.

`shipyard-compose.env` contains nonsecret placeholders used only for CI's
Compose interpolation check. Do not source it or use it to run the local stack;
local development still requires a private `.env` created from `.env.example`.

## Internal telemetry

`MARINA_METRICS_PORT=0` (default) disables the listener. Set it to `9090` to
serve Prometheus `GET /metrics` on a **separate internal HTTP port**, bound to
`0.0.0.0`. There is no `/metrics` route on the public application port 8080.
The listener is unauthenticated: restrict it to monitoring traffic; never add
an ingress, tunnel route, or public port mapping. Runtime wiring remains owned
by Infrastructure/Fleet and is not enabled by this application change.

| Metric | Type | Exact labels / values |
| --- | --- | --- |
| `marina_jobs` | gauge | `kind=ingest\|delete`, `status=uploading\|queued\|running\|done\|failed\|cancelled` |
| `marina_oldest_pending_job_age_seconds` | gauge | none; creation age of oldest uploading/queued/running job; 0 when empty |
| `marina_job_completions_total` | counter | `kind=ingest\|delete`, `outcome=done\|failed` |
| `marina_answers_total` | counter | `outcome=answered\|insufficient_evidence\|error` |
| `marina_answer_duration_seconds` | histogram | no application labels; `_bucket` adds `le`, plus `_sum` and `_count` |
| `marina_dependency_ready` | gauge | `dependency=configuration\|authentication\|database\|vectors\|models\|objects\|worker`, role-dependent; 0/1 |

Jobs/oldest-age gauges are emitted **only by knowledge-api**, read directly
from the database at scrape time, and include all owners without identifying
any. Other roles omit them. Dependency gauges execute the same checks as
`/ready`: API has all except `worker`; worker has all; RAG excludes `objects`
and `worker`; learning has no dependencies. Scraping performs real dependency
checks, including original-storage probes where applicable. A failed dependency
is 0; a failed DB gauge read or collection error returns HTTP 503 with a generic
body, never a successful scrape containing fabricated empty/healthy gauges.

Completions increment only after successful transaction commit, including
terminal lease exhaustion and permanent failures; automatic retries, stale
leases, cancellations and rolled-back commits do not increment them. A manual
retry can produce another terminal transition for the same job. Answers count
completed service calls, including errors and early insufficient-evidence
returns; rejected authentication/validation before the call is not an answer
attempt. Duration includes those calls' errors, in seconds, with finite buckets
`0.1, 0.5, 1, 2.5, 5, 10, 30, 60, 120, 300` (plus `+Inf`).

Registries/counters are instance-local and thread-safe, reset on restart, and
include Prometheus client's `_created` metadata. They are operational signals,
not a durable audit ledger: a crash between DB commit and increment can lose
an event. No owner, collection/document ID, filename/title, query, content or
credential labels are exported; default Python/process collectors are absent.
The lifespan starts/stops the listener and drains active scrapes before DB
disposal. Use one app process per network namespace when enabled (not multiple
Uvicorn workers sharing port 9090); an occupied port fails startup explicitly.
Sum event-counter rates across role instances, but do **not** sum database
gauges across API replicas querying the same database: select one API scrape
target, or aggregate those gauges with `max` rather than `sum`.

## Local startup (PowerShell)

Install Docker Desktop with its Linux engine running. From this directory:

```powershell
if (!(Test-Path .env)) { Copy-Item .env.example .env }
# Edit .env: fill every required secret and exact installed model tag.
# For LOCAL development only: MARINA_AUTH_MODE=dev and an externally generated
# MARINA_DEV_TOKEN. Alternatively configure the OIDC or forwardauth settings below.
docker compose config --quiet
docker compose up -d --build postgres qdrant ollama
```

Required secrets: `POSTGRES_PASSWORD`, `QDRANT_API_KEY`. Use a URL-safe
PostgreSQL password (or percent-encode reserved URL characters). Values are
never generated into source or supplied as defaults.

### Original document storage

Compose stores originals on the `marina-originals` named volume, mounted at
`/var/lib/marina` in `initialize`, `knowledge-api`, `knowledge-worker`, and
`agentic-rag` with `MARINA_STORAGE_BACKEND=filesystem` and
`MARINA_STORAGE_PATH=/var/lib/marina/originals`. The image creates
`/var/lib/marina` owned by UID/GID 65532 with mode 0700, so a new named volume
inherits that ownership; initialization creates the private `originals`
directory. Nothing changes permissions of existing data recursively.

| Variable | Values | Notes |
| --- | --- | --- |
| `MARINA_STORAGE_BACKEND` | `filesystem`, `s3` | Application default is `s3` for existing callers; Compose and production set `filesystem`. |
| `MARINA_STORAGE_PATH` | absolute path | Required for `filesystem`. Its parent must exist; the final directory is created 0700. |
| `MARINA_S3_ENDPOINT`, `MARINA_S3_ACCESS_KEY`, `MARINA_S3_SECRET_KEY`, `MARINA_S3_BUCKET`, `MARINA_S3_REGION` | | S3 mode, and the source of `copy-originals`. |

The filesystem adapter accepts only generated keys
(`collections/<id>/documents/<id>/original`), refuses traversal and symbolic
links (including in the root's ancestors), refuses a root other users can
access, writes 0600 files atomically (temporary file, fsync, rename, directory
fsync), deletes idempotently, and readiness performs a real write/fsync/delete
probe. Failures are explicit dependency errors, never an empty fallback.

**Stacks created before filesystem storage keep their originals in MinIO.**
Switching such a stack to the default Compose file does not copy, discard, or
overwrite anything: `initialize` checks every original that PostgreSQL metadata
references and fails with `Original storage migration required` when any is
absent, so the API and worker never start against an empty volume. Keep using
the opt-in S3 mode until you migrate:

```powershell
# .env must still contain the existing MINIO_ROOT_USER / MINIO_ROOT_PASSWORD.
docker compose -f docker-compose.yml -f docker-compose.s3.yml up -d --build
```

Explicit offline migration to filesystem storage:

1. Stop `knowledge-api`, `agentic-rag`, and `knowledge-worker` after pending
   jobs drain, and take a backup (`marina-admin backup ...`, see
   `PRODUCTION-RUNBOOK.md`) to a private location.
2. Copy and verify the originals. The command requires drained jobs, verifies
   each source SHA-256, never overwrites a differing target, reads back every
   copy, and finally checks every metadata reference:

   ```powershell
   docker compose -f docker-compose.yml -f docker-compose.s3.yml run --rm --no-deps `
     -e MARINA_STORAGE_BACKEND=filesystem --entrypoint marina-admin `
     initialize copy-originals --acknowledge-quiesced
   ```

3. Start the default (filesystem) stack: `docker compose up -d`. Keep the
   `minio-data` volume and backup until uploads, sources, and answers are
   verified; remove MinIO only deliberately afterwards.

Required model configuration: `MARINA_EMBEDDING_MODEL`,
`MARINA_GENERATION_MODEL`, `MARINA_EMBEDDING_DIMENSIONS`. Use **exact** tags as
reported by `ollama list`; the configured dimension must match the embedding
model. Hardware/model choices are deliberately operator decisions, not a claim
of evaluated quality. Example candidates to evaluate are `nomic-embed-text:v1.5`
(768 dimensions) and `qwen2.5:3b`, but neither is configured or downloaded
automatically. Install your chosen models explicitly:

```powershell
docker compose exec ollama ollama pull <chosen-exact-embedding-tag>
docker compose exec ollama ollama pull <chosen-exact-generation-tag>
docker compose exec ollama ollama list
docker compose up -d --build
Invoke-RestMethod http://127.0.0.1:8080/ready
```

The one-shot `initialize` service applies Alembic migrations under a PostgreSQL
advisory lock, initializes private original storage, verifies that every
original referenced by metadata exists there (failing immediately without
retry otherwise), creates a dimension-checked Qdrant collection,
and retries dependency startup for at most 30 attempts, with two-second waits
between attempts plus dependency operation timeouts. It does **not** download
models. Application roles never auto-migrate. Rerun initialization explicitly
after schema changes: `docker compose run --rm initialize`.

API ports 8080–8083, PostgreSQL 5432, Qdrant 6333, Ollama 11434 (and MinIO
9000/9001 in S3 mode) bind only `127.0.0.1`. Named volumes preserve data across
restarts and `docker compose down`; **`down -v` destroys local data**, including
every original. MinIO root credentials are a local S3-mode convenience only. Do
not expose local backing ports externally.

The optional local MinIO image is built from upstream release
`RELEASE.2025-04-22T22-12-26Z`, pinned to source commit
`0d7408fc9969caf07de6a8c3a84f9fbb10a6739e` in `docker/minio.Dockerfile`.
The formerly referenced public registry image is no longer pullable.
The first local build requires access to upstream source and Go dependencies;
the compiled image runs as a non-root user. This development image is not a
production storage rollout.

For local API calls, export your externally supplied token to the client/UI.
Chartroom should proxy to `http://127.0.0.1:8080`; all frontend workflows use
that one origin. Direct browser cross-origin requests are intentionally not
enabled. Never put tokens in URLs or committed frontend settings.

## Authentication

Production uses `MARINA_AUTH_MODE=forwardauth`: Lighthouse reuses the existing
Authentik `homelab-forward-auth` proxy provider and the browser's Authentik
session cookie. No OAuth provider or bearer token is needed for browser use. An
optional per-user OAuth bearer path can be enabled separately for read-only MCP
clients; it does not replace cookie authentication:

- `MARINA_FORWARDAUTH_URL`: trusted embedded-outpost endpoint, for example
  `http://authentik-server.auth.svc.cluster.local/outpost.goauthentik.io/auth/traefik`
  (http/https, no credentials, query, or fragment).
- `MARINA_FORWARDAUTH_EXTERNAL_URL`: Lighthouse's public HTTPS origin, for example
  `https://marina.<domain>`; its host is the fixed `X-Forwarded-Host`.
- `MARINA_FORWARDAUTH_OWNER_NAMESPACE`: stable HTTPS owner namespace, exactly
  `https://auth.<domain>/application/o/homelab-forward-auth/`.
- `MARINA_FORWARDAUTH_BROWSER_ORIGIN`: exact HTTPS Chartroom origin, for example
  `https://chartroom.<domain>`; the only `Origin` accepted for writes.
- Optional `MARINA_FORWARDAUTH_TIMEOUT` (seconds, default 5).

For every protected request Lighthouse itself sends only the incoming `Cookie`
header to the configured outpost with a fresh HTTP client (no redirects, no
cookie jar, no proxy environment, no `Authorization` forwarding) and fixed
`X-Forwarded-Proto: https`, `X-Forwarded-Host` from the external URL, and
`X-Forwarded-Uri` from the request path. Incoming `X-authentik-*` and
`X-Forwarded-*` headers are never trusted. A 2xx outpost answer must carry one
valid `X-authentik-uid`; the owner is `<namespace>|<uid>`, so renaming a user or
changing their e-mail does not change ownership. Missing/oversized cookies and
outpost denials (redirect/401/403) return 401; an unreachable outpost or a
malformed success returns 503 with a sanitized log entry. Cookies and sessions
are never cached or logged. Unsafe methods (POST/PUT/PATCH/DELETE) additionally
require `Origin` to equal the browser origin and reject `Sec-Fetch-Site:
cross-site` with 403. `/ready` checks the configuration and makes an anonymous
probe expecting the outpost's challenge (302/303/307/401/403); 404, 5xx or 2xx
mean not ready. `/healthz` and `/ready` never use cookies or expose identities.
Authenticated `GET /v1/session` returns `{"authenticated": true}` with no PII.

### Optional MCP OAuth bearer authentication

Leave all three settings empty to keep bearer authentication disabled in
`forwardauth` mode. To opt in, set all three:

- `MARINA_MCP_OIDC_ISSUER`: exact HTTPS issuer for the Authentik OAuth provider.
- `MARINA_MCP_OIDC_AUDIENCE`: the exact HTTPS URL of the MCP resource, for
  example `https://marina.<domain>/mcp`. Tokens with an audience array, an
  additional audience, or any other value are rejected.
- `MARINA_MCP_OIDC_JWKS_URL`: HTTPS JWKS URL for that issuer.

The signed JWT must use RS256 or ES256 and contain `exp`, `iat`, `sub`, `iss`,
`aud`, `scope`, and `marina_uid`. The `scope` must include `marina.read`.
Configure Authentik's trusted claim mapping so `marina_uid` is the same stable
user UID returned as `X-authentik-uid` by the existing forward-auth outpost.
Lighthouse uses that UID with the existing owner namespace, so the browser and
MCP client see the same collections. Key-service outages return 503; invalid
tokens return 401.

Bearer tokens are accepted only on `GET /v1/collections`,
`POST /v1/collections/{uuid}/search`, and
`GET /v1/documents/{uuid}/chunks/{uuid}`. All other bearer-authenticated routes,
including every write, return 403. A request containing both a bearer token and
a session cookie returns 401; an invalid bearer never falls back to its cookie.
Cookie-authenticated browser routes retain their existing outpost check and
exact-Origin CSRF protection.

Identities from earlier `oidc` deployments (`issuer|sub`) are not mapped
automatically; see the runbook's backup/restore owner migration.

Bearer `MARINA_AUTH_MODE=oidc` remains supported and fails closed until all are supplied:

- `MARINA_OIDC_ISSUER`: exact HTTPS issuer, including Authentik's trailing slash.
- `MARINA_OIDC_AUDIENCE`: API audience/client ID issued for Lighthouse.
- `MARINA_OIDC_JWKS_URL`: HTTPS signing-key endpoint from the trusted provider.

RS256/ES256 JWT signature, issuer, audience, subject, issued-at and expiration
are validated against JWKS (bounded cached key refresh). Owner identity is
issuer + subject; every resource access is checked against collection ownership.
Forwarded identity headers and unsigned tokens are **not** trusted. Valid tokens
whose issuer/audience do not match are rejected. Unknown resources and another
owner's resources both return 404.

Explicit `MARINA_AUTH_MODE=dev` accepts only the externally supplied
`MARINA_DEV_TOKEN`; no token is embedded or generated by the server. The default
local identity is `dev:local-user`, optionally set with `MARINA_DEV_SUBJECT`.
Dev mode must never be used on a production/public endpoint.

## Versioned UI contract

Authentication (the Authentik session cookie in `forwardauth` mode, an optional
read-only MCP bearer token in that mode, or a bearer token in `oidc`/`dev` mode)
is required on domain routes. `openapi.json` is the
knowledge-api contract, including the shared question implementation.
Regenerate reproducibly without dependencies/services or secrets:

```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -e ".[dev]"
.\.venv\Scripts\marina-admin.exe openapi --output openapi.json
```

| Method / path | Input / output |
| --- | --- |
| POST `/v1/collections` | `{name}` → 201 `{id,name}` |
| GET `/v1/collections` | `{items:[{id,name}]}` |
| GET `/v1/session` | `{authenticated:true}`; no identity data |
| POST `/v1/collections/{id}/documents` | Multipart **`file`** → 201 Document; exact duplicate → 200 existing Document |
| GET `/v1/collections/{id}/documents` | `{items:[Document]}`; includes deleting, excludes deleted |
| GET `/v1/documents/{id}` | Document, including terminal deleted tombstone |
| POST `/v1/documents/{id}/retry` | Document; 409 unless exhausted failed ingestion/cleanup job |
| DELETE `/v1/documents/{id}` | 202 Document with `deleting`; repeated request is idempotent |
| POST `/v1/collections/{id}/search` | `{query,limit?:1..20}` → `{items:[Citation]}` |
| POST `/v1/collections/{id}/questions` | `{question}` → `{answer,status,citations:[Citation]}` |
| GET `/v1/documents/{id}/chunks/{chunk_id}?version=1` | Citation; version optional, defaults to published version |
| GET `/v1/documents/{id}/original?version=1` | Owner-only original PDF bytes; version required; ready PDFs only |

`Document = {id,collection_id,title,status,error:string|null,created_at:string,
active_version:number|null}`. Status is `queued`, `processing`, `ready`, `failed`,
`deleting`, or `deleted`. `created_at` is ISO-8601 UTC.

`Citation = {document_id,version:number,chunk_id,title,text,line_start,line_end,page_number?:number}`.
Lines are one-based, inclusive, in decoded UTF-8 text with normalized newlines.
PDF citations additionally carry the one-based physical page number (not a printed
page label), and lines are relative to that page's extracted text. No chunk spans
pages. Non-PDF responses omit `page_number` and retain their existing fields/IDs.
Question status is `answered` or `insufficient_evidence`. Errors use
`{detail:string,request_id?:string}`; `X-Request-ID` is returned on responses.
Dependency/model outages are 503, not successful empty/generated answers.

## Persistence and failure behavior

- Maximum original size defaults to 5 MiB (`MARINA_MAX_UPLOAD_BYTES`). Only
  `.md`, `.markdown`, `.txt`, and text-layer `.pdf` are supported.
  Text/Markdown accept UTF-8/BOM; empty/NUL/invalid UTF-8 is rejected.
  Chunks preserve headings and deterministic line ranges: maximum 40 lines and
  4,000 characters. Overlong individual lines are rejected, not silently
  truncated. Exact raw-original SHA-256 deduplicates **within a collection**;
  filenames do not affect identity. Across collections, uploads are independent.
- PDFs are admitted by extension/header and parsed only by the worker, in a
  separate process with a **20-second wall-time limit**, **256 MiB process-memory
  limit**, **100-page limit**, and **1 MiB extracted UTF-8 text limit**. PDF input
  remains capped at 5 MiB even if the generic upload limit is raised. One parser
  runs per worker process; the parser receives no application environment
  credentials. Linux uses `RLIMIT_AS`; Windows uses a process-memory Job Object.
  Failure to establish the limit is an explicit dependency failure, not an
  unbounded fallback. This is resource isolation, not an OS security sandbox.
  Malformed, encrypted, scanned/image-only or blank-page PDFs fail with an
  actionable document/job error without automatic repeated parsing. Remove blank
  pages, export a text-layer copy or run OCR externally before uploading.
  OCR, image extraction, layout/table reconstruction, and PDF JavaScript or
  external-resource execution are not supported. Extraction order is the text
  layer's order; visually complex PDFs may need a simpler export.
- Raw PDF originals are retained unchanged in the configured private original
  store (including filesystem PVC storage), covered by existing byte-hash
  backup/reindex/deletion behavior. The versioned original endpoint rechecks
  ownership/publication and verifies the raw hash; it returns attachment bytes
  with `Cache-Control: no-store` and `nosniff`. MCP read-only OAuth tokens cannot
  download originals; the three existing MCP routes are unchanged.
- PostgreSQL metadata/job creation precedes original storage, retaining a
  durable object key even after a process dies. Upload failures are visible;
  if no original exists, delete and upload again. An interrupted upload is
  recovered after the upload grace/lease window. Objects are never public.
- Jobs use transactional `FOR UPDATE SKIP LOCKED`, lease tokens, heartbeats,
  crash recovery, three attempts and bounded exponential delay. Exhausted
  jobs show errors; manual retry resets their budget. Configure with
  `MARINA_LEASE_SECONDS`, `MARINA_MAX_ATTEMPTS`, `MARINA_RETRY_SECONDS`.
- Chunk UUIDs are deterministic for document/version/ordinal. Qdrant upserts
  are idempotent. PostgreSQL alone publishes the ready version after successful
  indexing; failed/partial/stale index entries never reach retrieval.
- Versions store embedding tag, actual Ollama digest and dimension. Changed
  embedding identities fail closed on retrieval: restore the indexed model
  or delete/re-upload into a rebuilt index. MVP uploads are immutable version 1;
  replacement/update APIs are intentionally not implemented.
- Delete immediately marks metadata `deleting` and clears publication before
  cleanup. Worker removes vectors, originals, chunks and version metadata;
  failures remain `deleting` and can be retried after their job exhausts.
  A minimal deleted tombstone/job audit remains (IDs/status/timestamps);
  filename, object key and content hash are cleared.
- Publication and deletion lock the document around external index/storage
  writes to prevent late-ingestion resurrection. These short, bounded external
  operations hold a transaction; PostgreSQL connection/lock sizing needs
  production measurement.
- Search filters Qdrant by collection and then rechecks owner, ready status and
  published version against PostgreSQL. Source endpoints resolve stored text,
  never model-generated snippets. Deleting documents cannot resolve sources.
- Questions use the same shared service on knowledge-api and agentic-rag.
  At most five authorized passages / 16,000 evidence characters are supplied.
  Uploaded content is untrusted evidence, never a system instruction.
  Structured model output is validated; citation IDs must be from supplied
  evidence and answers need citations. Sources are rechecked after generation.
  Model timeout defaults to 90 seconds and concurrency to two per process
  (`MARINA_MODEL_TIMEOUT`, `MARINA_MODEL_CONCURRENCY`, `MARINA_CONTEXT_CHARS`).
  Logs record model/digest, retrieval count and duration, not question/document
  contents. Citation validity does not prove factual entailment.

## Development and verification

```powershell
.\.venv\Scripts\python.exe -m pytest -q
.\.venv\Scripts\python.exe -m pytest -q evaluation\test_run.py
.\.venv\Scripts\python.exe -m build
```

Tests inject deterministic object/vector/model adapters and exercise real
SQLAlchemy transactions plus Alembic migration on SQLite. PostgreSQL-specific
locking is **not** claimed by SQLite tests. To run the same suite plus concurrent
SKIP LOCKED claims on PostgreSQL, supply `MARINA_TEST_DATABASE_URL` targeting
a **disposable local test database** using `postgresql+psycopg://...`. Tests
create/drop isolated schemas; never point this at a live cluster database.
Without it the PostgreSQL concurrency test is explicitly skipped.

External adapters use actual boto3, Qdrant HTTP client and Ollama APIs; mocked
transport tests cover model errors, timeouts, dimensions and output validation.
Local Qdrant is pinned to `v1.18.2`, matching Infrastructure. The pinned client
uses the compatible collection, query-points, payload-filter and upsert APIs;
the local runtime smoke test verifies server interoperability.
Real model quality gates are separate: see [evaluation](evaluation/README.md).
Milestone 3 can be reproduced with `evaluation/run.py --retrieval-only`;
the local run achieved 20/20 supporting passages in the top five and exact
source resolution through Nginx without calling generation.
Search checks collection ownership and embedding digest/dimensions,
deduplicates ranked IDs, and excludes unpublished versions from source access.
Milestone 2's separate `evaluation/reliability.py` harness kills an actual child
worker after Qdrant writes but before database publication, verifies natural
lease recovery and exact chunk/point counts, and cleans its isolated resources.
It has passed against the real local PostgreSQL, S3, Qdrant, and Ollama services;
the running application worker is not interrupted by that harness.
The local run using `nomic-embed-text:v1.5` and `qwen2.5:3b` retrieved supporting
passages for 20/20 answerable questions, produced cited answers for 19/20,
and abstained on all 10 unsupported questions. Every citation resolved to its
stored source. This passes the automatic retrieval/abstention gates, but is
not a complete quality approval: evidence review found incorrect Flux group
names, a truncated ingress path, and an unnecessary abstention about the
optional worker. A subsequent current-source API evaluation of `qwen2.5:7b`
achieved 20/20 retrieval hits, 20/20 cited answers, and 10/10 unsupported
rejections. It fixed the ingress path and optional-worker cases, but assistant
review of all factual answers still found wrong Flux groups and an unsupported
qualification about Mac Ollama access. Neither candidate has factual-quality
approval. The harmless document-instruction probe passed and deleted its
fixture document; this is not a universal prompt-injection guarantee.
See [the Milestone 4 results](DEVELOPMENT-PLAN.md#milestone-4-add-grounded-question-answering)
for the model digest and remaining acceptance boundaries.
No arbitrary similarity threshold is used.

The previous grounded-generation implementation audited every answered draft against its cited passages
and the specific question. A rejection permits one correction and another
audit; a second rejection becomes `insufficient_evidence` without citations.
Malformed output, invalid citations, truncation, and model failures at any stage
remain explicit dependency errors. Evidence, drafts, and review reasons stay in
untrusted user data, never system instructions or ordinary logs.
The auditor analyzes evidence before issuing its verdict, but uses the same
configured model and can share its errors. Its approval does not replace
human evidence review.
The strict real-model run withheld both known bad answers and rejected all
10 unsupported questions, but answered only 16/20 supported questions (below
the unchanged 18/20 coverage diagnostic). It also wrongly abstained on an
independent sensor-routing example; a repeat probe also rejected an explicit
processing-stage list. This is a safety improvement, not accepted
answer quality; that free-form approach did not meet Milestone 4 acceptance.
A larger `qwen2.5:14b` comparison also failed acceptance: it matched 7B's 5/6
independent verifier-calibration result, still accepting an unsupported
qualification. Its corpus run stopped on HTTP 503 at the Mac Ollama question;
focused reproduction confirmed a correction timeout at the default 90-second
limit after a false verifier rejection. The partial run is not a full-set pass.
Completed questions had approximately 114-second median latency on the local
CPU runtime, with observed Ollama memory around 14 GiB. The larger candidate
was not promoted to the normal stack.
A different-family `llama3.1:8b` run completed all 30 questions, achieving
20/20 retrieval hits, 19/20 cited answers, 10/10 unsupported rejections, and
passing the four independent grounding probes. However, its auditor accepted
wrong Flux group names and an incomplete ingress path. Calibration was again
5/6, missing an unsupported qualification. Its CPU corpus median latency was
approximately 45 seconds, with approximately 6 GiB reported loaded for Llama.
This candidate is also not factually accepted or promoted; automatic gates
and independent probes do not replace reviewing every answer against citations.

### Current source-first answers

The current adapter replaces free-form drafting and self-auditing with one
source-selection call. It gives the model numbered source sentences and their
section headings. Selected IDs are validated and mapped back to the original
chunks; the server assembles the answer from exact stored text, preserving
newlines, names, quantities, and complete paths rather than rewriting them.
Chartroom labels successful answers **What your sources say**. The public API
shape is unchanged, but `answer` now contains quotations, not generated summaries.

Sentence candidates are split conservatively at punctuation followed by a
capitalized word, within existing paragraphs/chunks. They are not a linguistic
parser; unusual punctuation, abbreviations, or chunk boundaries can affect
selection granularity. Section headings are untrusted selection context and are
not rewritten into answers. A real quotation can still be irrelevant or incomplete;
exact copying guarantees provenance, not relevance, truth, or universal injection resistance.
Malformed selections, unknown IDs, token truncation, timeouts, and oversized
selected text remain explicit errors. Unsupported questions abstain with no citations.

The 2026-11-03 Foundry ingestion acceptance exposed a presentation limitation:
the deployed adapter quotes installation and login commands but does not support
arbitrary formatting instructions or numbered steps. An earlier candidate adding exact
source-line selection and server-rendered numbering produced the requested two
steps with the live model, but still omitted concrete setup instructions for
the broad question and sometimes selected unrelated page navigation for focused
questions. It failed the quality gate and was not adopted or promoted.
That failed candidate was not adopted.

The subsequent local answer-quality implementation explicitly asks for every
requested part and distinguishes quoting a command from executing source
instructions. English requests for a "numbered list" or "numbered steps" use a
bounded section selector: it copies each requested operation verbatim from the
question, validates request order and source IDs, retains section descriptions
with their code, and adds server-owned ordinal prefixes. Explicit counts such
as "two numbered steps" are validated (1-20). If any step has no selected support,
the entire answer abstains; malformed output remains an explicit dependency
error, never a partial success. Unnumbered answers retain sentence selection.
The response schema and citation ownership/page metadata are unchanged.

Local acceptance: the synthetic baseline passed 5/9 cases; the new adapter
passed all nine cases across three repeated real-model runs, plus isolated
single-/two-chunk retrieval and the real-model PDF page-2/source/original checks.
The retained SDK guide also returned installation/login commands in the requested
two steps. These checks do not prove universal completeness, relevance, or
injection resistance; a section can still contain unrelated text, and code
spanning ingestion chunks is not reconstructed. Reviewed image promotion and
authenticated Chartroom browser acceptance completed on 2026-11-03: both live
Chromium tests passed with manual Authentik login and real API/model responses,
including multipart answers, numbered commands, source inspection and cleanup.
See [the rollout evidence](EXPANSION-PLAN.md#recommended-next-application-work)
and [the repeatable quality gate](evaluation/README.md#multipart-answer-quality).

The source-first `qwen2.5:7b` full run achieved **20/20 retrieval hits, 19/20
answered questions, 10/10 unsupported rejections, and 4/4 independent probes**.
Every returned excerpt matched its cited text exactly. Assistant review found
the returned answers relevant and complete for their questions. The Flux group
question abstained instead of giving incorrect names. On 2026-10-09, the user
confirmed satisfactory testing and approved the implementation for commit,
closing local [Milestone 4](DEVELOPMENT-PLAN.md#milestone-4-add-grounded-question-answering).
Production deployment remains separately gated.

Generation now makes one model call with the default 90-second timeout, plus
retrieval/embedding work. Chartroom and the evaluator retain their conservative
600-second request budget. Run `evaluation/run.py --source-first` to check exact
quotation provenance in addition to existing acceptance metrics.

The previously failing dependency-download image build now passes without
disabling integrity verification or weakening dependencies. The fresh backend
image has passed readiness and an authenticated answer/source check through an
isolated Chartroom Nginx proxy. Normal local containers retain their earlier
images and 3B configuration until candidate quality is accepted.

Local Docker validation has exercised image builds, backing-service startup,
PostgreSQL concurrent job claims, real document ingestion/retrieval, and
Chartroom's authenticated Nginx proxy. Full app/backing-service restarts preserve
retrieval, and deletion removes actual originals and vector points.
A separate database and original-storage restore
verified metadata and all nine fixture originals byte-for-byte.
These local development-token checks do not establish production Authentik
access, public reachability, or image security-scan results. Real-model quality
is a separate acceptance gate, not implied by service readiness.
Migration rollback is not automatic: retain databases/volumes,
back up PostgreSQL and originals before upgrades, and do not downgrade a
database containing documents without a tested restore.

For the verified local handoff, the ignored `.env` files retain the isolated
Compose project names, generated development credentials, and tested model
tags. Start or stop each app from its own directory with `docker compose up -d`
or `docker compose stop`; do not use `down -v` unless discarding its local data.
Open Chartroom at `http://127.0.0.1:8084` and use `MARINA_DEV_TOKEN` from
Lighthouse's local `.env` in its token connection form. Do not publish that
development token or reuse development authentication in production.

### Injectable service interface

Offline harnesses can inject `Knowledge(settings, database, objects, vectors,
models)` into `create_app("knowledge-api", settings=settings,
knowledge=knowledge)` and drive the identical HTTP contract with `TestClient`.
Adapter interfaces are `Objects`, `Vectors`, `Models` in `marina.adapters`.
For direct service evaluation, `Knowledge.search(owner, collection_id, query,
limit)` returns `list[Citation]`; `Knowledge.question(owner, collection_id,
question)` returns `AnswerOutput`; `Knowledge.source(owner, document_id,
chunk_id, version)` returns `Citation`. Schemas live in `marina.schemas`.
All paths retain authorization and published-version checks. Fake adapters are
test-only, not runtime fallbacks; they cannot measure embedding/model quality.
