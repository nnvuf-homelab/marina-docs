# Chartroom

MARINA's document library and evidence desk, backed by Lighthouse's `/v1` API.
Create collections, upload source documents, follow indexing status, retry failed
jobs, explicitly confirm deletion, and ask questions with inspectable versioned
source passages. There is no representative corpus or unsupported graph view.
Successful answers are labelled **What your sources say**: Lighthouse selects
exact source excerpts rather than composing a summary. Newlines are preserved;
fenced code blocks receive code formatting, while prose and source passages
remain plain text. Raw HTML and other Markdown are not interpreted.
Use the citations to check context and relevance; a quotation is not a guarantee
that a document is true or that the selected passage completely answers a question.

PDF citations show physical page numbers and page-relative extracted line ranges.
Opening a source passage fetches the original through the same authenticated
Lighthouse client, then offers a temporary PDF link at `#page=N`. Blob URLs are
revoked when the source/workspace closes; credentials never appear in the link.
PDF viewing/page fragments depend on the browser's PDF viewer. Deploy the
PDF-capable Lighthouse backend and migration before this UI.

## Workspace destinations

- **Conversations** is for private, source-first questions and saved chat history.
- **Research** is for knowledge-building threads, approved public connector
  queries, evidence review, and reviewed knowledge updates. Manual-source runs
  remain available under **Advanced: manual-source research run**.
- **Library** is for uploaded and published collection documents, source search,
  and temporary one-off questions that are not saved to conversation history.
- **Activity** and **Settings** provide job recovery and connector/workspace
  configuration.

All destinations use the selected collection. Research and Conversations lists
are filtered by mode on the server before pagination, keeping their totals and
page boundaries accurate as saved history grows. Knowledge-building chat in
Research requires the conversation and learning capabilities to be enabled by
Lighthouse; Advanced manual-source runs remain a separate workflow.

## Development

```sh
npm ci
npm run generate:api
npm run lint
npm test
npm run build
npm run dev
```

Run Lighthouse on `localhost:8080`. Vite proxies `/api` to that address and strips
the prefix. `VITE_LIGHTHOUSE_URL` defaults to `/api`; set it before starting Vite
or building to use a different API base URL (cross-origin APIs must allow your
origin). Do not put tokens in environment files or build arguments.

Vite and local Compose default to explicit **token** mode. Paste a user-supplied
development bearer token into the labeled token entry form. Tokens live only in
React memory, never local/session storage, URLs, or cookies, and are cleared on
disconnect, unauthorized responses, or reload. This local UI is not a login
service; production does not require manual tokens or an OAuth callback.

## Production browser session

The production image defaults to `CHARTROOM_AUTH_MODE=session`. On startup,
Nginx serves the non-secret runtime configuration at `/auth-config.json`:
`{"authMode":"session"}`. The startup hook validates `session` or `token` before
envsubst; no build argument or Fleet auth environment override is needed.
Vite's static `public/auth-config.json` provides `{"authMode":"token"}` for local
development; local Compose explicitly sets `CHARTROOM_AUTH_MODE=token`.
Missing, unavailable or invalid configuration fails closed with a reload/error
screen, not a manual-token fallback.

Protect the Chartroom public page with Traefik's existing Authentik forward-auth
middleware. A top-level visit performs login through that middleware; Chartroom
does not implement a callback, OIDC client setup, or session/token exchange.
In session mode the UI automatically requests `/api/v1/collections`, sends
`credentials: "same-origin"`, and never emits a bearer token. It always uses the
same-origin `/api` proxy regardless of `VITE_LIGHTHOUSE_URL`. The client also
rejects external API URLs and refuses fetch redirects in session mode, preventing
API requests from becoming third-party login navigations. Cookies remain
HTTP-only and Authentik-managed; JavaScript never reads them.

Lighthouse must use its `forwardauth` runtime mode (production handoff:
`MARINA_AUTH_MODE=forwardauth`), independently verify browser cookies against its
fixed trusted Authentik outpost, and reject forged request identity headers.
Its unsafe-request CSRF check requires the exact Chartroom public Origin.
Nginx copies the browser's Cookie and original Origin; it never synthesizes an
Origin when the browser omits one. Missing/wrong Origin must be rejected by
Lighthouse. Neither this repo nor its configuration endpoint provisions backend
authentication, trusted outpost URLs, or ingress middleware.

On API 401, Chartroom unmounts the workspace, aborts outstanding requests, and
clears collections, documents, upload selection, questions, answers and passages.
**Sign in again** makes a top-level navigation back to the protected page so
Authentik can authenticate again. HTTP 403/503 and other dependency failures stay
visible with explicit recovery; they do not clear private state or trigger login
redirects. **Clear workspace** drops private UI state without claiming to log
out of Authentik; **Reopen workspace** revisits the protected page. To end an
Authentik session, use the provider's own account/session controls.

Upload UTF-8 plain text, Markdown (`.txt`, `.md`, `.markdown`) or text-layer PDF
(`.pdf`) up to 5 MiB. PDFs support up to 100 physical pages; scanned/image-only,
blank-page, encrypted and malformed PDFs fail with an actionable backend error.
OCR is not included. Text files must be nonempty and NUL-free, with lines no longer
than 4,000 characters. Lighthouse owns authoritative validation. Failed ingestion
jobs can be retried after automatic retries are exhausted; deletion cleanup
errors expose retry as well. A still-active automatic retry may return 409.
Only ready
documents contribute to answers. Active documents are polled every three seconds,
up to 60 consecutive checks; terminal jobs are not polled. An error or the time
limit pauses polling until **Check status again** / **Refresh**. Switching
collection or disconnecting aborts outstanding requests and drops the old corpus,
answer, and passage state.

## Live authenticated browser acceptance

Authentication is already implemented through Authentik and the same-origin
session proxy. The live acceptance suite tests that existing boundary; it does
not add a second login system. Unlike `test:session`, it never intercepts or
mocks API responses, cookies, or Authentik.

Run the anonymous public challenge check without credentials:

```powershell
npm run test:live
```

By default this runs one real Chromium challenge check and explicitly **skips**
the authenticated acceptance case. A passing anonymous check is not acceptance
of the new answer-quality backend.

After the reviewed Lighthouse answer-quality image has been promoted, run the
complete check from a Windows desktop:

```powershell
$env:CHARTROOM_LIVE_LOGIN = '1'
try { npm run test:live }
finally { Remove-Item Env:\CHARTROOM_LIVE_LOGIN }
```

A fresh headed Chromium window opens. Sign in to Authentik yourself within five
minutes (including MFA if required). Never send credentials or cookies to the
assistant. The test uses only that ephemeral browser context, with tracing,
screenshots and video disabled; it never exports browser storage or login state.
Automatic failure-page snapshots are disabled and test output is not retained.
The browser closes when the test finishes.

The authenticated case checks real session-mode configuration, no bearer token
entry/headers, the exact Origin on writes, CSRF rejection of missing/foreign
Origin, upload/indexing, complete multipart answers, two ordered numbered steps,
exact rendered shell blocks and source citations, unsupported-part abstention,
document deletion and private workspace clearing. It uses a synthetic fixture,
not personal documents. Only its new document is removed, with explicit cleanup
failure if deletion cannot be confirmed. An empty acceptance collection remains
because Lighthouse has no collection-delete endpoint; its ID is printed.
An interrupted/killed run may require manual fixture cleanup.

Defaults are `https://chartroom.nnvuf.uk` and `https://auth.nnvuf.uk`.
`CHARTROOM_LIVE_URL` and `CHARTROOM_LIVE_AUTH_ORIGIN` can select other explicitly
authorized HTTPS origins. The suite does not merge promotions, deploy images,
change auth configuration, or log out other Authentik sessions. Two-user
isolation, real session expiry/renewal, and PDF viewer behavior remain separate
acceptance tasks.

Validation on 2026-11-03: **both live Chromium tests passed**, including manual
operator Authentik login, against the promoted Lighthouse answer-quality image.
The synthetic document was confirmed deleted; empty acceptance collection
`25a6892a-dc99-4b10-9e6c-ecceffc364dc` remains. Both application HelmReleases
were Ready. No browser session state was exported.
Lint/build, 49 unit tests and both mocked session browser regressions also passed.

### Two-human isolation acceptance

With the approved `akadmin` and `nnvu.fit` accounts already able to sign in:

```powershell
$env:CHARTROOM_LIVE_ISOLATION = '1'
try { npm run test:live -- --grep 'two real human' }
finally { Remove-Item Env:\CHARTROOM_LIVE_ISOLATION }
```

Sign in as the account named in the console for each separate browser window.
The test verifies each username through Authentik, creates a synthetic document
for each owner, checks own list/search/exact-source access, and requires symmetric
404 refusals for the other owner's known collection/document/chunk IDs.
It then deletes both documents and confirms terminal deletion. Empty collections
remain. No cookies or tokens are exported; the live configuration disables
screenshots, failure snapshots, traces and videos.

For a coordinated follow-on MCP check only, explicitly set
`CHARTROOM_LIVE_KEEP_ISOLATION=1` as well. A successful run then retains its two
synthetic documents and prints only their fixture IDs/markers, never credentials.
Failed runs attempt both cleanups. Remove that environment variable after the
run and delete the retained documents through their owning Chartroom sessions
after MCP acceptance. Abrupt interruption may require manual cleanup.
This harness validates the browser boundary, **not** per-user OAuth MCP isolation.

## API contract generation

Lighthouse exports `../lighthouse/openapi.json`. Generated definitions are tracked
in `src/api.generated.ts` and the API client references those schema types.
Regenerate after backend contract changes:

```sh
npm run generate:api
npm run generate:api -- path/to/openapi.json
```

The optional path is resolved from the Chartroom repository root. Generation fails
if the schema is missing or invalid; no placeholder definitions are used.
TypeScript 5.9 is used to satisfy openapi-typescript's supported peer range.

## Local container workflow

With Lighthouse reachable on the host at port 8080:

```sh
docker compose up --build
```

Open `http://localhost:8084`. Nginx substitutes `LIGHTHOUSE_UPSTREAM` into its
runtime template. The image default is
`http://knowledge-api.marina-core.svc.cluster.local:8080`, matching the existing
Fleet service handoff when no environment is injected. Compose overrides this
with `http://host.docker.internal:8080` and adds the Linux host gateway mapping.
Override it with an HTTP(S) upstream address for
another local backend. Use an origin only (hostname/IP and optional port, no path).
A sourced startup hook normalizes trailing slashes before the image's envsubst
hook runs; the envsubst filter preserves Nginx's own `$uri` and header variables.
The image build normalizes the startup hook to LF line endings so Windows
checkouts also run under Alpine's shell.
`/api/` is proxied with the prefix stripped. Nginx disables general request-header
forwarding and explicitly passes Host, Authorization (local token compatibility),
Cookie, the original Origin, Content-Type and Accept. No client `X-authentik-*`
headers, including arbitrary future identity header names, are forwarded.
`/healthz` and `/ready` remain lightweight UI probes.
The proxy allows 6 MiB request bodies including multipart overhead; Lighthouse
enforces the tighter 5 MiB file limit.
The proxy read timeout retains a conservative 600-second budget for retrieval
and model work. Lighthouse's current source-first mode uses one selection call
with a default 90-second timeout, replacing the former four-call drafting/audit
flow. If the backend model
timeout is raised, review the proxy budget as well. The UI remains in its
loading state until completion; disconnecting or switching collections cancels
the browser request, but does not guarantee cancellation of backend generation.

Production deployments may override the image's default upstream URL. A
build-time `VITE_LIGHTHOUSE_URL` Docker
argument is available for external APIs in local token mode only; session mode
always uses same-origin `/api`. This
repository does not modify production cluster or Authentik configuration.
Production delivery remains gated on backend forward-auth configuration and
Authentik-protected ingress; the default upstream does not provision either.

`npm test` includes static proxy-contract and startup-hook checks.
`docker compose config --quiet` validates configuration, not actual Nginx
startup; verify `/ready` and an authenticated `/api/v1/collections` request
after starting the container.

`npm run test:session` starts an isolated Vite server on port 4187 and runs a
mocked Chromium browser scenario with runtime session configuration. It verifies
automatic connection without a token prompt, browser-supplied Cookie and Origin
on writes, 503 recovery without state loss, 401 workspace clearing, and top-level
sign-in navigation. It uses a synthetic HTTP-only cookie and mocked API, not the
shared local stack or real Authentik. Unit tests additionally cover runtime
configuration failure, session-mode external-URL rejection and original Nginx
header-variable contracts. These checks do not prove live outpost validation or
production ingress behavior.

## Browser acceptance (Milestone 5)

Local acceptance on 2026-10-09 passed **2 browser scenarios and 27 unit/client/proxy
tests**, plus lint, TypeScript checks, frontend/container builds, and Compose
configuration validation. The browser used an isolated current-image UI/API
against real local dependencies with `qwen2.5:7b`; normal containers and
production were not changed.

Run the current Lighthouse image, its worker and backing services, and Chartroom
locally before testing. Use the accepted source-first generation model
(`qwen2.5:7b` in local acceptance); an older backend image is not equivalent.
Browser tests target the running UI and its Nginx `/api` proxy, not a mocked API.

```powershell
npx playwright install chromium
$env:CHARTROOM_E2E_URL = 'http://127.0.0.1:8084'
# Supply the local development token or an authorized test user's token securely.
$env:CHARTROOM_E2E_TOKEN = [System.Net.NetworkCredential]::new('', (Read-Host 'Local test token' -AsSecureString)).Password
npm run test:e2e
Remove-Item Env:\CHARTROOM_E2E_TOKEN
```

The token is required and is not stored in the test files. Use a local disposable
test workspace, not production. Tests create uniquely named collections and
upload a non-sensitive beacon document. They complete ingestion, reload and
reconnect, ask a real source-first question, inspect its versioned passage,
confirm deletion, and wait for cleanup. A `finally` block deletes the uploaded
document even on assertion failure. Empty test collections remain because the
API does not expose collection deletion.

A separate browser scenario injects HTTP 503 and 401 responses to verify visible
dependency errors, recovery against the real empty collection, and clearing
private UI on unauthorized access. This is controlled UI fault injection, not
proof of an actual model outage. Unit tests additionally cover ingestion and
cleanup retries, polling failure/recovery, source-load failure/recovery, upload
limits, cancellation, and stale-response isolation.

Tracing, screenshots, and video are disabled to avoid persisting authenticated
workspace content. Playwright output directories are gitignored. Do not enable
tracing with real credentials or private documents.

The local runtime handoff has verified Nginx startup, authenticated API proxying,
missing/invalid token rejection, persisted source retrieval after full-stack
restart, and document deletion through `/api`. The ignored local `.env` retains
the running Compose project name. Open `http://127.0.0.1:8084` and use the
development token from Lighthouse's ignored local `.env`. Generated answer
quality still depends on the model; see Lighthouse's evaluation limitations.
