# MARINA production deployment

## Status and ownership

There are two environments: local development and production. This runbook describes production deployment, not a separate pilot environment.
Lighthouse and Chartroom are deployed through Fleet and unsuspended. On
2026-10-18 the live restart, isolated restore, encrypted-backup, resource-sizing,
anonymous Authentik challenge, and operator-driven Chartroom workflow gates
passed. The operator confirmed final public-session validation and accepted
Milestone 6.

- Infrastructure owns PostgreSQL, the `lighthouse-originals` original-document
  PVC, Qdrant/Ollama, Authentik forward-auth, secrets, TLS, tunnel routes, and monitoring.
- Fleet owns the runtime storage contract `MARINA_STORAGE_BACKEND=filesystem` and
  `MARINA_STORAGE_PATH=/var/lib/marina/originals`, and mounts that PVC at
  `/var/lib/marina` in the initialization hook and every Lighthouse Deployment.
- Fleet owns immutable application images, runtime references, and admission.
- Lighthouse owns migrations, backup/restore/reindex tools, and smoke evaluation.
- Chartroom owns the browser workflow. Shipyard preserves Fleet release overrides
  when generating the next image promotion.

Do not merge/push runtime changes before the local-only secrets are present.
The new PostgreSQL service is wired to Infrastructure's applications group;
a watched-branch merge will provision it. It is not an application Deployment.

## Internal application metrics (Phase 10 handoff)

Lighthouse implements the [stable metrics interface](README.md#internal-telemetry).
This does not enable production scraping or change runtime configuration.
Infrastructure/Fleet must separately set `MARINA_METRICS_PORT=9090` on each
desired role and authorize private monitoring access. Default `0` binds no
listener. Keep 9090 internal and unauthenticated, monitoring-only; do not route
it through public Traefik/Cloudflare or add `/metrics` to the public 8080 app.
Use one app process per pod/network namespace, not Uvicorn multi-worker mode.

Scrape `http://<private-role-target>:9090/metrics` with a timeout that covers
the existing readiness checks (the same external probes as `/ready`, including
storage write probes). Validate all of:

- The public application `/metrics` remains 404 and protected API routes
  retain their existing authentication behavior.
- `marina_dependency_ready{dependency}` is 0/1 for the role's existing checks.
  An unavailable dependency reports 0; inability to collect database gauges
  returns HTTP 503, not healthy zeros. Monitor scrape `up` as well as readiness.
- Only API targets emit `marina_jobs{kind,status}` and
  `marina_oldest_pending_job_age_seconds`. Select one API replica, or use `max`
  across replicas sharing the database; never sum their duplicate gauges.
  Pending means uploading, queued or running, aged from creation (including
  retry wait); no pending jobs reports 0.
- Sum rates of `marina_job_completions_total{kind,outcome}` across worker
  instances; outcomes are exactly `done|failed`, and failures are terminal only.
  Counters represent committed transitions, not DB status counts. Cancelled,
  stale, rolled-back and automatic-retry transitions are excluded.
- Sum `marina_answers_total{outcome}` rates across API/RAG instances; outcomes
  are exactly `answered|insufficient_evidence|error`. Use
  `marina_answer_duration_seconds_bucket` (`le`), `_sum` and `_count` for latency.
  These process-local counters/histograms reset on restart; they are not an
  exact durable event ledger across crashes.

All label vocabularies and histogram buckets are fixed in the README. No
owner/document/query/title/content/token labels or default process collectors
are exported. Listener bind failure blocks startup; lifespan shutdown closes
the socket and joins in-flight scrapes before disposing database resources.
Disable only this feature by restoring `MARINA_METRICS_PORT=0` and removing its
private scrape target in a separately authorized runtime change; application
HTTP/authentication behavior is unchanged.

## 1. Provision dependencies before admission

1. Reserve capacity and approve the release image digests. Production API and worker
   each request 250m CPU/256Mi and limit 2 CPU/1Gi. PostgreSQL requests
   250m/256Mi and limits 2 CPU/1Gi, with a retained 10Gi local-path RWO PVC.
   These are starting budgets, not measured cluster sizing.
   App readiness allows 15 seconds for dependency checks; a 135-second termination
   grace covers the default model timeout and worker shutdown wait. Interrupted
   jobs still rely on lease recovery, not on assuming shutdown always completes.
2. Verify Vault is unsealed and External Secrets is Ready. It synchronizes
   registry pull credentials and Origin TLS into `marina-core` and `marina-ui`,
   and both Lighthouse database Secrets into `marina-core`. Do not run
   `Infrastructure/scripts/setup-secrets.sh` as the steady-state writer; it is
   for explicitly authorized bootstrap/recovery only. Existing Cloudflare/Traefik
   owns the edge path; no public backing-store ports.
3. Original documents use filesystem storage on Infrastructure's retained
   `lighthouse-originals` PVC (20Gi, local-path, ReadWriteOnce, Flux prune
   disabled). No S3 endpoint or credentials are required. Fleet mounts it at
   `/var/lib/marina` with `fsGroup: 65532` (the image's non-root UID); the
   initialization hook creates the private `originals` child directory (0700)
   and refuses a missing parent, symbolic links, other-user permissions, or an
   unwritable root. Local-path binds the PVC to one node, so the API, worker,
   and hook schedule there; Deployments use `Recreate`, and losing that node
   means restoring from an off-node backup. The volume is not itself a backup.
   Lighthouse's S3 mode remains for compatibility only and is not this rollout.
4. Existing Qdrant in `ai` is internal and has no configured API key. The `ai`
   namespace default-denies ingress, and K3s *does* enforce NetworkPolicy via its
   embedded kube-router controller, so `marina-core` needs explicit allow rules
   for Qdrant (6333/6334) and Ollama (11434) — see
   `Infrastructure/cluster/applications/ai-netpol.yaml`. Without them the pods
   fail readiness with `connection refused`. Those rules are namespace-scoped
   reachability, not authentication: accept that trust boundary explicitly or add
   authentication before sensitive data admission.
   Use the dedicated `marina_chunks` collection, never another app's collection.
5. Choose a reachable Ollama Service (GPU or external Mac bridge), explicitly
   install `nomic-embed-text:v1.5` (768 dimensions) and `qwen2.5:7b`, and verify
   exact tags/digests. Local CPU quality acceptance does not prove cluster GPU
   latency or memory capacity. Do not substitute an unevaluated family/tag.
6. Browser authentication uses `MARINA_AUTH_MODE=forwardauth` with the
   **existing** Authentik `homelab-forward-auth` proxy provider and embedded
   outpost. No OAuth provider, callback URI, client secret, or manually pasted
   bearer token is needed for browser use, and there is no auth Secret for
   Lighthouse. Infrastructure derives the required runtime ConfigMap values
   from `CLOUDFLARE_DOMAIN`:

   | Variable | Value |
   | --- | --- |
   | `MARINA_AUTH_MODE` | `forwardauth` |
   | `MARINA_FORWARDAUTH_URL` | `http://authentik-server.auth.svc.cluster.local/outpost.goauthentik.io/auth/traefik` |
   | `MARINA_FORWARDAUTH_EXTERNAL_URL` | `https://marina.<domain>` |
   | `MARINA_FORWARDAUTH_OWNER_NAMESPACE` | `https://auth.<domain>/application/o/homelab-forward-auth/` |
   | `MARINA_FORWARDAUTH_BROWSER_ORIGIN` | `https://chartroom.<domain>` |

   Lighthouse re-verifies the browser `Cookie` against that fixed internal
   outpost URL on every protected request and never trusts incoming
   `X-authentik-*` or `X-Forwarded-*` headers, so a request reaching the
   Service directly cannot forge an identity. The owner is the namespace plus the
   outpost's `X-authentik-uid`; usernames/e-mails are not used. The owner
   namespace is a fixed identifier: never change it after data is admitted.
   Review which users/groups the `homelab-forward-auth` application admits:
   every admitted user can create their own Lighthouse collections. Do not reuse
   Kubernetes control-plane OIDC or run `setup-k8s-oidc.sh` for this
   application. Local `dev` mode is prohibited on public routes.
   Optional per-user MCP OAuth bearer access is disabled unless all three
   additional Lighthouse settings are provided: `MARINA_MCP_OIDC_ISSUER`,
   `MARINA_MCP_OIDC_AUDIENCE` (the exact HTTPS MCP resource URL), and
   `MARINA_MCP_OIDC_JWKS_URL`. Configure a dedicated Authentik OAuth provider
   and map its signed `marina_uid` claim to the same stable UID returned by the
   forward-auth outpost; tokens must include the `marina.read` scope. The token
   audience is matched exactly. These bearer tokens are limited to collection
   listing, collection search, and source-chunk retrieval; all other routes,
   including writes, remain cookie-authenticated. Before enabling this in
   production, Infrastructure must explicitly extend the runtime ConfigMap and
   its provisioning workflow to carry the three non-secret values, after
   reviewing the Authentik mapping and client registration. No OAuth client
   secret is needed by Lighthouse.
7. Fill the nonsecret `LIGHTHOUSE_OLLAMA_URL`, model names, embedding dimensions,
   and `CLOUDFLARE_DOMAIN` in Infrastructure's local `.env`.
   Provision against the exact approved kubectl context:

   ```sh
   bash scripts/setup-lighthouse.sh --provision-runtime --context <approved-context>
   ```

   Run from Infrastructure. The command requires the selected context to match
   and pins every provisioning API call to that explicit context. It writes
   only the namespace and `lighthouse-runtime` ConfigMap; it does not read or
   write credentials and needs no Vault CLI login, port-forward, or password inputs.
   External Secrets generates `lighthouse-postgres` and `lighthouse-credentials`
   from the existing Vault values. Verify synchronization before enabling Fleet:

   ```sh
   kubectl --context <approved-context> wait --for=condition=Ready clustersecretstore/vault-homelab --timeout=120s
   kubectl --context <approved-context> -n marina-core wait --for=condition=Ready externalsecret/vault-lighthouse-postgres externalsecret/vault-lighthouse-credentials --timeout=360s
   kubectl --context <approved-context> -n marina-core wait --for=condition=Ready externalsecret/vault-ghcr-pull externalsecret/vault-cloudflare-origin-tls --timeout=360s
   kubectl --context <approved-context> -n marina-ui wait --for=condition=Ready externalsecret/vault-ghcr-pull externalsecret/vault-cloudflare-origin-tls --timeout=360s
   ```

   The ExternalSecrets must exist first; namespace discovery and refresh can take
   five minutes. Inspect `kubectl get clusterexternalsecrets` and namespace
   ExternalSecret conditions if they are missing. Do not print Secret contents.
   Changing a Vault password does not rotate an initialized database password;
   coordinate database and client rotation separately. Never reset a bound PVC.

To merge nonsecret template settings into the local `.env` and prompt for
missing inputs, run from Infrastructure:

```sh
python3 scripts/generate-env.py
```

On Windows use `python` instead of `python3` if needed. This standard-library
Python tool works without sourcing/executing the shell `.env`. It preserves
existing values and vault inputs and does not change cluster resources.
The current template omits Vault-managed application passwords; the generator
does not create their Vault entries or rotate existing credentials.
`--non-interactive` leaves unresolved external inputs explicitly reported by name.
New files use restrictive Unix permissions; on Windows, verify the local file's
NTFS ACL restricts access to your account. Keep the entire `.env` out of Git.

The script provisions `lighthouse-runtime`; External Secrets owns
`lighthouse-credentials` and `lighthouse-postgres`. Both consume the same
application password at `secret/homelab/LIGHTHOUSE_POSTGRES_PASSWORD`
(field `value`), with URL percent-encoding applied by the database URL template.
The administrator password is stored separately at
`secret/homelab/LIGHTHOUSE_POSTGRES_ADMIN_PASSWORD`; never use the same value.
PostgreSQL bootstraps with a separate administrator password and creates the
`marina` application role as a database owner without superuser/role/database
creation privileges. App containers never receive the administrator password.
PostgreSQL's role-bootstrap script runs only on an empty PVC; existing databases need a
separately reviewed role migration, not a PVC reset.

## 2. Promote and deploy through Flux

The Fleet Helm release owns initialization. Its pre-install/pre-upgrade Job
runs Alembic migration and storage initialization using the same immutable image
and runtime references as the application, mounts the same originals PVC,
and refuses to complete when PostgreSQL references originals missing from the
selected storage (`Original storage migration required`); never satisfy that by
pointing at an empty volume. It requires `forwardauth` mode with a valid
configuration (`Settings.from_env().auth_error()` is empty) and completes
before Helm starts the new workloads. A failed Job blocks the release; inspect
its logs and fix the dependency before retrying. Successful hooks are removed;
failed hooks remain until the next Helm attempt replaces them.

There is no separate initialization image or context in `.env`. Legacy
`LIGHTHOUSE_PILOT_IMAGE` / `LIGHTHOUSE_PILOT_CONTEXT` entries in an existing local
file are unused and may be removed. The explicit provisioning `--context`
argument is still required; Flux itself uses its cluster-local credentials.

For upgrades, old application pods may still be running while the hook executes.
Use backward-compatible migrations. For incompatible changes, stop writers and
arrange a maintenance window with a verified backup before deploying. Helm
rollback does not undo database migrations or original/vector initialization.

Review the immutable images in both Fleet charts: currently committed digests
are **not asserted to contain this implementation**. Promote signed, scanned
images through Shipyard and inspect the promotion diff. Report-only scanner
findings require operator review; they are not automatically admission-blocking.
Do not let a promotion reset `suspend`, runtime overrides, initialization, or
rollback policy. The app workflows pin Shipyard's policy-preserving commit
`93c181cb495edc159b7a693a9b6d88566c1adf91`, rather than the older `v0.3.0`.
Its generator leaves the Fleet initialization template intact; future tooling
updates must keep both the hook template and release overrides.

Production delivery is: merge app changes to `main` (or push an approved release
tag), wait for Shipyard build/test/signing and the Fleet promotion PR, review
and merge that PR, then let Flux reconcile. A local commit or feature-branch
push does not publish/deploy the application. `workflow_dispatch` currently
validates but does not publish. Infrastructure's watched `main` push deploys
backing services independently; verify Vault-synchronized secrets before that push.

Change Fleet's `spec.suspend` to false only after secrets, backing services,
model/forward-auth configuration, and backup preparation pass, and approved images are
promoted. Flux then runs the initialization hook before application rollout.
Suspension is a temporary production safety hold, not another environment.
Start just knowledge-api and knowledge-worker; agentic-rag and learning remain
at zero replicas because they are not needed for the current UI/API path.
This reduces duplicate model capacity and keeps the single UI/API path.

Users sign in once through Authentik forward-auth at the edge. Chartroom's
same-origin API calls carry the Authentik session cookie and its exact
`https://chartroom.<domain>` `Origin`; Lighthouse independently re-verifies the
cookie with the outpost and keeps its ownership checks authoritative. A
cookie-free request to the public Lighthouse route receives the edge's Authentik
login redirect (or 401): that is the expected authentication challenge, not an
outage. Writes whose `Origin` is missing or differs from the Chartroom origin
return 403 (CSRF protection), including writes sent from the Lighthouse origin.
Resume Chartroom only after Lighthouse is ready.

Configure public routes `marina.nnvuf.uk` and `chartroom.nnvuf.uk` through
Cloudflare Tunnel -> Traefik -> Ingress -> ClusterIP. Validate TLS and external
reachability separately; rendering manifests proves neither.

## 3. Operational visibility and sizing

- `/healthz`: process liveness only; never use it as a dependency gate.
- `/ready`: role-specific checks for database migration, a write/fsync/delete
  probe of original storage (API/worker),
  vector dimensions, model inventory, authentication, and worker activity.
  In `forwardauth` mode the authentication check validates configuration and
  sends an anonymous probe to the outpost for `/v1/collections`; a login
  challenge (302/303/307/401/403) is healthy, while 404, 5xx, 2xx, or an
  unreachable outpost is not. It sends no cookie and reveals no identity.
  A dependency failure returns 503 and boolean checks, not a healthy fallback.
- Authenticated `/v1/operations`: only the caller's document/job status counts
  and oldest pending job age; no titles, IDs, evidence, or private error text.
  Investigate failed jobs immediately, queue age above 180s, or any unready role.
- Existing Promtail/Loki captures structured `job_failed`, `job_lease_exhausted`,
  `dependency_request_failed`, and `readiness_check_failed` logs. Use the
  `marina-core` namespace filter. Do not put private evidence in log labels.
- Inspect restart count, RSS, CPU throttling, PostgreSQL and originals PVC
  free space, and model memory
  during production verification. Keep at least 20% PVC free space; choose alerting thresholds
  after measuring the cluster. The authenticated operations endpoint is an
  operator signal, **not** a newly configured Prometheus scrape target.

Record measurements and failures with model/image digests. Backing-store failure
must be visible, recover after restoration, and never produce invented answers.
Do not intentionally interrupt production dependencies without a maintenance plan.

## 4. Consistent backup and isolated restore

Quiesce **all** API writers and workers, block ingress, and drain pending jobs
first. The acknowledgment flags do not enforce this for you. Do not copy live
PVC contents as a database backup. Pin the PostgreSQL client major to at least
the server major; the application image includes the PostgreSQL client tools.

Run an administrative container/Job with the runtime environment and a private
mounted backup destination:

```sh
marina-admin backup --acknowledge-quiesced --output /backup/marina.zip
```

This creates a new archive exclusively, checks there are no pending jobs, uses
`pg_dump` custom format with no owner/ACL restoration, and includes exact
original bytes plus SHA-256 hashes. Partial backups are removed on failure.
No vectors or credentials are included. Metadata still contains private owner
identities and document text: encrypt the archive, restrict destination ACLs,
copy it off-node, and configure retention before resuming service.
Do not commit or expose the archive. The archive hashes detect corruption,
not malicious tampering; trust and protect the backup source.

Restore into a **new empty database and a new empty original-storage
directory** (a fresh PVC, or an empty `MARINA_STORAGE_PATH` on one) using
separate target runtime settings, with API/worker traffic stopped. Keep the
archive off-node; a node-local PVC does not survive node loss:

```sh
marina-admin restore --acknowledge-empty-target --input /backup/marina.zip
marina-admin init-storage
marina-admin reindex --acknowledge-quiesced
```

Use a fresh Qdrant collection name in every restored role's settings. Restore
validates the manifest (original keys, member names, hashes) before writes and
refuses tables in any non-system schema or any existing original. Each original
is written atomically and read back, then every metadata reference is verified.
`pg_restore` is transactional; original writes are not. If one fails, treat the
isolated target as incomplete, remove only its explicitly identified resources,
and restore into another empty target.
Never redirect live traffic to a partial restore.

Reindex validates original hashes, unpublishes ready documents, and queues
normal deterministic ingestion without destroying originals or existing chunks.
Resume the target worker, wait for every expected document to be ready, verify
metadata/original counts, exact citations and vector counts, then test questions.
Preserve failed/deleting records for deliberate remediation; rebuild only queues
ready documents. Older citation IDs remain stable with unchanged chunking.
Embedding changes require rebuilding *all* documents into a fresh collection.
Owner identity is `MARINA_FORWARDAUTH_OWNER_NAMESPACE|<Authentik user UID>` in
`forwardauth` mode (`issuer|sub` in `oidc` mode). Keep that namespace and the
Authentik users stable when restoring; a different namespace, a recreated
Authentik user (new UID), or a switch from `oidc` does not transfer collection
ownership. Owners from earlier `oidc` deployments are **not** mapped
automatically. To migrate them, quiesce writers, take and verify a backup, then
in a reviewed maintenance window update `collections.owner` from the old
`issuer|sub` to the new `namespace|uid` for each confirmed user (the UID is shown
on the user in the Authentik admin interface), verify ownership through
`/v1/collections` as each user, and keep the backup until accepted. Local `dev`
documents do not become a production user's documents merely by switching
authentication mode. Start production verification with a new non-sensitive
collection owned by its real Authentik user.

The local harness `evaluation/restore_rehearsal.py --acknowledge-local --output ...`
creates disposable databases, original-storage directories (sibling
directories of `MARINA_STORAGE_PATH`, or buckets in S3 mode) and vector
collections, exercises real backup, restore, reindex, exact source retrieval and
source-first answering, and removes its isolated resources. It requires local
database creation privileges; do not run it with production administrator settings.

## 5. Public admission gates

Local rehearsal passed on 2026-10-09: 169 backend/evaluation tests (one expected
SQLite-only skip), 27 frontend tests, two Chromium scenarios, 36 Shipyard tests,
and four Infrastructure guard tests. Helm/Kustomize rendering, Bash syntax,
contracts, and builds passed. Real isolated backup/restore/reindex and
non-superuser PostgreSQL migration passed. The candidate API process restart
preserved exact sources. An isolated unreachable S3 endpoint and missing model
returned 503 while liveness stayed healthy; valid configuration recovered exact
answers. Shared services were not interrupted. These are local rehearsals,
not public Authentik or cluster failure-recovery evidence.

Sign in to Chartroom as a non-sensitive test user and copy the complete `Cookie`
request-header value from an authenticated
`https://marina.nnvuf.uk/v1/session` request that returned 200. Windows
PowerShell's `Read-Host` can truncate long Authentik cookies. Copy the value to
the clipboard, set it without echoing with
`$env:MARINA_EVALUATION_COOKIE = (Get-Clipboard -Raw).Trim()`, and immediately
run `Set-Clipboard -Value " " -ErrorAction Stop` to replace it with a single
space (some Windows PowerShell versions reject an empty string). This does not
remove clipboard history or synchronized copies; clear those separately if
enabled. Never put the cookie in arguments, files, reports, chat,
or tickets; clear the environment variable after the run:

```sh
python evaluation/production.py --api-url https://marina.nnvuf.uk --origin https://chartroom.nnvuf.uk --login-prefix https://auth.nnvuf.uk/ --output evaluation-report-production-public.json
```

The smoke requires cookie mode (bearer tokens are refused) and an exact HTTPS
Chartroom `--origin` (or `MARINA_EVALUATION_ORIGIN`). Cookie-free, forged
`X-authentik-*`/forwarded-identity, and invalid-bearer requests must receive the
login redirect or 401; any other status fails. Optional `--direct-api-url` (a
loopback port-forward to the API Service) must return 401 for forged identity
without a session. With the session it checks `/v1/session`, that forged headers
do not change the owner, 403 for foreign/missing write `Origin`, readiness,
failed owner jobs, and queue age above 180s. It never follows redirects, so the
cookie cannot leak to another host. It uploads only a non-sensitive
fixture, checks exact source answers/abstention and source retrieval, then deletes
the document and waits for cleanup. Empty fixture collections remain.
Repeat the real Chartroom UI flow through the authenticated public browser
session; local controlled 401/503 tests do not prove public Authentik integration.

A successful local rehearsal is not a successful public smoke. Live application
restart, isolated restore, encrypted-backup, dependency recovery, image
promotion/scanning, resource measurement, anonymous Authentik rejection, and the
operator-driven UI workflow were recorded on 2026-10-18. The operator accepted
Milestone 6 after final public-session validation. Repeat the cookie-based
authorization, forged-header, CSRF, private-object, queue, citation, and cleanup
checks for future production candidates.

## 6. Rollback

Block ingress and stop writers before any data recovery. Suspend the Fleet
releases through reviewed Git; suspension alone does not scale down pods.
Revert to the prior compatible immutable images and runtime ConfigMap values.
Do not remove the PostgreSQL or `lighthouse-originals` PVCs, or drop
collections, to roll back code. Images older than filesystem storage cannot read
originals from the PVC; roll back only to a filesystem-capable image.

Phase 9 introduces additive migration `0002`, a nullable `chunks.page_number`
column. Before promotion, take a fresh consistent backup and rehearse upgrade
from `0001`, text/PDF ingestion, original retrieval and deletion in isolated
targets. The initialization hook must apply `0002` before new API/workers serve;
new readiness requires it. Promote Lighthouse before PDF-capable Chartroom and
Harbormaster. Existing documents need no reindex and non-PDF citations are unchanged.
Do not alter Infrastructure/Fleet or run a live migration as part of local
application development.

Pre-Phase-9 images hard-code readiness to `0001` and therefore are **not a
ready-compatible rollback** on `0002`, even though the column is additive.
Prepare a reviewed compatibility image that accepts the newer schema, or keep
traffic blocked and perform the isolated recovery procedure below. A Helm image
rollback does not undo `0002`. Never automatically downgrade a database.
If old code cannot read the schema, keep traffic blocked and restore a pre-change
backup to isolated targets, validate, then switch all roles consistently.
Keep the original volumes and backup until the recovery is accepted.
