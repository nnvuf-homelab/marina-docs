# Harbormaster

MARINA's read-only Model Context Protocol gateway. It is a thin client of the
Lighthouse API and never connects directly to PostgreSQL, Qdrant, object
storage, Trawler, the shell, or Kubernetes.

## Tools

| Tool | Lighthouse call | Bounds |
| --- | --- | --- |
| `marina_list_collections` | `GET /v1/collections` | 100 collections |
| `marina_search` | `POST /v1/collections/{id}/search` | 1-10 results, 2,000 chars each |
| `marina_get_source` | `GET /v1/documents/{id}/chunks/{chunk}` | 8,000 chars |

All tools are annotated read-only. `src/policy.ts` allows only these three
routes; any other method or path, including collection creation, upload,
retry, delete, and questions, is refused before a network call. Each MCP
result is capped at 64 KB.

PDF search/source results include optional `page_number` (one-based physical
page) alongside page-relative `lines`. Non-PDF output is unchanged. No PDF
download tool or original-file route is added to the read-only allow-list.

Lighthouse scopes every result to the calling identity, so Harbormaster sees
only collections that identity can read. Each tool call writes one JSON audit
line to stderr with the tool, a credential fingerprint (`session:<sha256
prefix>`), the collection or document touched, the outcome, and the duration.
Queries and credentials are never logged.

## Run locally (stdio)

```bash
npm ci
npm test
npm run build
```

The homelab workstation uses one user-level VS Code configuration
(`MCP: Open User Configuration`) to start the server against production;
there are no duplicate root or repository-local configurations. VS Code prompts for
your Authentik session cookie (`name=value`, from the browser's cookies for
`https://marina.nnvuf.uk`) and stores it as a password input, not in the file.
The cookie expires with your Authentik session; restart the server with a new
one when tools report "supply a current session".

Copilot CLI-backed sessions use a separate `~/.copilot/mcp-config.json` because
they do not read VS Code's configuration. On the Windows homelab workstation,
that entry runs the local `~/.copilot/harbormaster.ps1` launcher and permits only
the three read-only tools. Run the launcher with `-SetCookie` to enter a fresh
cookie through a hidden prompt; it stores the cookie encrypted with Windows
CurrentUser DPAPI and decrypts it only for the server process. Restart the MCP
server after changing the cookie. Use `-ClearCookie` and stop the server to
remove the saved credential and its running copy. It cannot reuse VS Code's
cached password input. Neither
configuration contains the cookie itself. User-level configurations apply
across workspaces; enable Harbormaster only in trusted sessions.

| Variable | Purpose |
| --- | --- |
| `LIGHTHOUSE_URL` | Lighthouse origin. HTTPS required except loopback or `*.svc.cluster.local`. Defaults to `http://knowledge-api.marina-core.svc.cluster.local:8080`. |
| `LIGHTHOUSE_SESSION_COOKIE` | Authentik session cookie for forward-auth Lighthouse. |
| `LIGHTHOUSE_ORIGIN` | Exact HTTPS origin Lighthouse allows for search (`https://chartroom.nnvuf.uk`). |
| `LIGHTHOUSE_TOKEN` | Bearer token for a local Lighthouse in `dev` or `oidc` mode. Mutually exclusive with the cookie. |
| `HEALTH_PORT` | Health endpoint port; `0` disables it for stdio use. |

## Cluster workload

The `marina-mcp` workload carries no Lighthouse credential and no backing-store
credential. Without a credential it serves only `/healthz` and `/ready`.
Public remote MCP deployment requires a dedicated Authentik OAuth
application with per-user delegated identity (see Lighthouse
`EXPANSION-PLAN.md`, Milestone 7).

## Streamable HTTP

An opt-in, stateless Streamable HTTP endpoint is available at `/mcp`.
Stdio remains the default. Start it locally with:

```powershell
$env:MCP_TRANSPORT = "http"
$env:HEALTH_PORT = "8091"
$env:LIGHTHOUSE_URL = "https://marina.nnvuf.uk"
$env:LIGHTHOUSE_ORIGIN = "https://chartroom.nnvuf.uk"
Remove-Item Env:\LIGHTHOUSE_SESSION_COOKIE, Env:\LIGHTHOUSE_TOKEN -ErrorAction SilentlyContinue
npm start
```

Configure an HTTP-capable MCP client with URL `http://127.0.0.1:8091/mcp`
and a `Cookie` header supplied from its secure credential facility. Every
request needs the calling user's own Authentik session cookie. No server
credential is used, and credentials are never shared across requests.
Lighthouse validates the cookie when a knowledge tool is invoked; an invalid
or expired cookie returns a tool error. Initialization alone does not prove
that a cookie is valid. Bearer authentication is not supported on this endpoint.

Cookie-mode HTTP binding is intentionally restricted to loopback.
Requests with a different Host
or a foreign Origin are refused; request bodies are capped at 32 KB.
The 2025 protocol is stateless: POST requests work without session IDs,
and GET/DELETE session operations return 405. No legacy standalone SSE
endpoint is provided. The SDK also supports its newer HTTP protocol.
Health routes share the HTTP port. The cluster deployment remains health-only
because HTTP mode is not enabled there.

### Authentik OAuth mode (opt-in)

Set `MCP_HTTP_AUTH=oauth` with `MCP_TRANSPORT=http`, `MCP_OAUTH_ISSUER`
and `MCP_OAUTH_JWKS_URL` from the dedicated Authentik provider's discovery
document. `MCP_HTTP_ORIGIN` identifies the HTTP resource; its `/mcp` URL
is the required token audience. For TLS-terminated deployment, configure an
HTTPS public origin and `MCP_HTTP_BIND=0.0.0.0`; loopback remains the default.
Do not set a server cookie or bearer credential.

OAuth mode accepts only per-request bearer JWTs, validates issuer, exact
audience, signature, expiry, subject, `marina.read` scope and `marina_uid`,
and advertises protected-resource metadata with OAuth challenges. Cookie
authentication remains separate. Discovery and challenges request
`marina.read offline_access` so compatible clients can obtain and securely
store a refresh token from Authentik, then silently renew access tokens.
Harbormaster never receives or stores refresh tokens; `offline_access` is not
required to execute read tools with an otherwise valid access token. Existing
clients need a fresh consent/login to obtain the additional scope.

Cookie HTTP mode remains loopback-only. Stdio credentials and behavior are unchanged.
Lighthouse must enable its optional MCP OIDC verifier and remain on the
private internal service URL so its browser ingress does not intercept bearer
requests. Both services verify the same logical resource token independently.

See [the Authentik setup and rollout runbook](../Infrastructure/docs/runbooks/marina-mcp-oauth.md).
The provider and public OAuth runtime are deployed. Interactive login has been
reported successful; real-client public search and exact source retrieval also
passed for the synthetic page-2 PDF fixture. Real `akadmin` and non-admin
`nnvu.fit` browser and native MCP sessions passed symmetric isolation checks:
own collection listing/search/exact-source succeeded, foreign collections were
absent, and known foreign search/source IDs were refused without source content.
The synthetic documents await owner-authenticated cleanup.
The same VS Code connection passed search/source reads after waiting
past its five-minute token expiry without signing in again; read-only Authentik
metadata confirmed new access expiry and refresh rotation. Silent renewal is
accepted for the tested `akadmin` client, not asserted for `nnvu.fit` renewal or
other clients. The two-user isolation acceptance is independently established.

## Milestone 7 acceptance

The stdio scope was accepted on 2026-11-02. A real MCP JSON-RPC client
using the local encrypted-cookie launcher listed four production collections,
searched an existing collection, retrieved a source with matching citation
IDs and nonempty text, and received a refusal for a delete-shaped tool call.
No production data was changed. Lighthouse remains responsible for owner
scoping; acceptance used one operator identity.

Local VS Code chat access is confirmed. Existing CLI-backed chat access is
not claimed: the launcher works independently, but the chat connection still
requires separate troubleshooting. Public remote OAuth is deployed and its
real two-user acceptance is recorded above.
Release tagging and Fleet promotion are separate from stdio acceptance.

## Future work

MARINA v1 is accepted for the agreed scope. Remaining work is tracked in the
[cross-repository future-work list](../lighthouse/EXPANSION-PLAN.md#7-future-work-and-deferred-scope):

- Delete both retained synthetic isolation documents through their owning
  Chartroom accounts and commit the remaining acceptance records.
- Optionally verify post-expiry renewal as `nnvu.fit` and additional MCP hosts;
  do not repeat or relabel the already-passed two-user isolation checks.
- Revisit client compatibility and current dependency/base-image findings as
  maintenance work, with fresh evidence.
- Backup automation, independent-host recovery and external notifications
  remain deferred or intentionally unconfigured until requested.

No broader crawling, write-capable MCP or learning-agent scope is implied.
