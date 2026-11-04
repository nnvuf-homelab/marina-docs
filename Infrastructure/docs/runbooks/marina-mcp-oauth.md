# MARINA MCP: per-user Authentik OAuth

## Status and ownership

Harbormaster implements OAuth-protected Streamable HTTP alongside its unchanged
stdio authentication. Lighthouse optionally accepts the same resource token on
its three MCP read routes while retaining browser forward-auth and CSRF checks.
The dedicated Authentik provider and public OAuth HTTP runtime are deployed.
Interactive PKCE login and real-client search/source reads are verified.
Client-side silent renewal beyond the five-minute lifetime is verified for
the tested VS Code connection. Two-real-user browser and native MCP isolation
are verified; synthetic fixture cleanup remains outstanding.

For routine readiness, authorization failures and compatible rollback, see the
[Harbormaster operations runbook](marina-operations.md#harbormaster). This does
not change the separately tracked interactive/two-user acceptance gates.

### Provisioning record: 2026-11-02

The dedicated Authentik application is provisioned. The deployment record below
tracks subsequent public MCP activation separately from interactive acceptance.

| Setting | Applied value |
| --- | --- |
| Application slug | `marina-mcp` |
| Public client ID (not a secret) | `marina-mcp-vscode` |
| Grants | Authorization code (PKCE S256) and refresh token |
| Consent | Existing explicit-consent flow |
| PKCE | Application policy requires S256 and a valid challenge; token exchange requires a verifier |
| Signing | Existing Authentik managed asymmetric JWT certificate |
| Access-token lifetime | 5 minutes |
| Refresh-token lifetime | 7 days; rotate on every refresh (`seconds=0` threshold) |
| Scopes | `openid`, `marina.read`, `offline_access` |
| Exact callbacks | `http://127.0.0.1:33418/`, `https://vscode.dev/redirect` |
| Access | Human members of `marina-mcp-users` only; group initially empty |
| Issuer | `https://auth.nnvuf.uk/application/o/marina-mcp/` |
| JWKS | `https://auth.nnvuf.uk/application/o/marina-mcp/jwks/` |

Callbacks were verified against the installed VS Code registration prompt.
The loopback callback includes the trailing `/`: actual authorization requests
include it even though the VS Code registration popup displays the URL without
it. Authentik strict matching rejects the slashless registration. This was
corrected after inspecting the rejected requests on November 2.
Register other clients' callbacks separately after verifying their actual
values. Add only explicitly approved human accounts to `marina-mcp-users`
in Authentik; no account was automatically granted access.

On 2026-11-02, the operator explicitly approved enrollment of `akadmin`.
Membership was added and verified through the real application policy engine
with an authorization-code S256 request. This verifies account eligibility,
not an interactive login or public MCP deployment. To revoke this enrollment,
remove `akadmin` from `marina-mcp-users`.

Validation passed for public discovery and JWKS through Cloudflare, an
in-memory signed access token for an unsaved synthetic user (signature, issuer,
exact audience, UID and scope), and the real Authentik policy evaluator's group
and PKCE branches. No user grant or access-token record was created for these
checks. Interactive login and two-user application acceptance remain pending.

The idempotent provisioning source is
[`provision-marina-mcp-oauth.py`](https://github.com/nnvuf-homelab/infrastructure/blob/main/scripts/provision-marina-mcp-oauth.py).
It runs inside the existing Authentik worker Django shell, uses one database
transaction, preserves group membership, and refuses conflicting application
bindings. Re-running it is a gated live mutation:

```bash
kubectl --context homelab exec -i -n auth deploy/authentik-worker -- ak shell \
  < scripts/provision-marina-mcp-oauth.py
```

For Windows PowerShell, from the Infrastructure repository:

```powershell
$source = Get-Content -Raw .\scripts\provision-marina-mcp-oauth.py
($source -replace "`r", "") | wsl --exec kubectl --context homelab exec -i -n auth deploy/authentik-worker -- ak shell
```

No cluster Secret, API-server flag, existing browser provider, or workload
deployment is modified by this provisioning.

Infrastructure owns Authentik configuration and runtime activation. Harbormaster
and Lighthouse own their application code. No provider secret, access token, or
session cookie belongs in git. The existing Kubernetes OIDC provider and
control-plane configuration are unrelated and must not be changed.

## 1. Create an application/provider (manual, gated)

In Authentik's Admin UI create an OAuth2/OIDC application/provider pair with
application slug `marina-mcp`. Use authorization-code login with consent,
PKCE S256, and an asymmetric signing key. Use a public client for native MCP
clients; never use client-credentials grants or a shared service identity.

Register the actual callback URI supplied by each MCP client as an exact
redirect URI. Do not leave the redirect list empty or permit arbitrary redirects.
Clients must support a pre-registered OAuth client ID: automatic dynamic
registration is not implemented by Harbormaster. Register additional native
clients deliberately, with their own callback configuration.

Create an OAuth scope mapping named `MARINA MCP read identity` with scope name
`marina.read` and expression:

```python
return {
    "marina_uid": request.user.uid,
    "aud": "https://mcp.nnvuf.uk/mcp",
}
```

Attach this mapping and the standard `openid` mapping to the provider.
`request.user.uid` is Authentik's installation-specific user UID, not a
username or an OAuth provider's potentially different `sub`. Verify the claim
matches the trusted outpost `X-authentik-uid` for the same test user before
enabling Lighthouse. Restrict application access through explicit user/group
policy bindings. Do not let service accounts obtain the mapped scope. The mapping rechecks
active human status and group membership on every token issuance, including
refresh. Authentik's installed refresh implementation does not rerun application
policies; failing the claim mapping deliberately withholds the resource audience
and UID, so both MCP validators reject tokens for removed/inactive users.

The access token must be a signed JWT containing:

- `iss`: exactly the provider discovery document's issuer;
- `aud`: exactly `https://mcp.nnvuf.uk/mcp` as a single string;
- `exp`, `iat`, nonempty `sub`;
- `scope` including `marina.read`;
- `marina_uid` matching the existing browser identity.

Inspect test tokens locally without posting or logging them. If Authentik's
issued audience differs from the mapping, correct provider configuration first;
do not relax audience validation in either service.

Keep five-minute access tokens. Refresh tokens require `offline_access` and
client support. They rotate on every successful renewal and expire seven days
after issuance (sliding with rotation). Revoking refresh access or removing
membership does not immediately invalidate an already-issued access JWT before
its five-minute expiry. Revoke refresh-token records in Authentik to end renewal;
signing out of the browser alone does not revoke offline access.

## 2. Configure the applications

Read the actual provider discovery document at:

```text
https://auth.nnvuf.uk/application/o/marina-mcp/.well-known/openid-configuration
```

Copy its exact `issuer` and `jwks_uri`, rather than guessing them.
Keep HTTPS verification enabled. Lighthouse identifies its JWKS requests with
`User-Agent: MARINA-Lighthouse/1.1`; the default Python urllib user agent is
blocked by the current Cloudflare policy. The identified request was verified
from the production API pod before activation.
Harbormaster settings:

```text
MCP_TRANSPORT=http
MCP_HTTP_AUTH=oauth
MCP_HTTP_ORIGIN=https://mcp.nnvuf.uk
MCP_HTTP_BIND=0.0.0.0
HEALTH_PORT=8080
MCP_OAUTH_ISSUER=<exact discovery issuer>
MCP_OAUTH_JWKS_URL=<exact discovery jwks_uri>
LIGHTHOUSE_URL=http://knowledge-api.marina-core.svc.cluster.local:8080
```

Do not set `LIGHTHOUSE_TOKEN` or `LIGHTHOUSE_SESSION_COOKIE` in HTTP mode.
Each request supplies its own OAuth bearer token. Harbormaster validates it
before initialization or tool execution; Lighthouse independently validates
it again as the private backend of the same `.../mcp` resource. The token must
never be forwarded to an unrelated API or exchanged for a shared credential.

Lighthouse settings, alongside its existing `MARINA_AUTH_MODE=forwardauth`:

```text
MARINA_MCP_OIDC_ISSUER=<same exact discovery issuer>
MARINA_MCP_OIDC_JWKS_URL=<same exact discovery jwks_uri>
MARINA_MCP_OIDC_AUDIENCE=https://mcp.nnvuf.uk/mcp
```

All three must be configured together. Browser owner keys stay unchanged:
`MARINA_FORWARDAUTH_OWNER_NAMESPACE|marina_uid`. Bearer requests can only list
collections, search one collection, or retrieve a source chunk. Writes are
refused even if the caller bypasses Harbormaster and reaches Lighthouse.

Activate Lighthouse first, then Harbormaster. Infrastructure owns the
`applications/marina-runtime` ConfigMaps and MCP Ingress. Fleet's Lighthouse
release must import `lighthouse-mcp-oauth` after the existing runtime ConfigMap,
and Harbormaster must import `harbormaster-runtime`. Origin TLS distribution
includes `marina-mcp` through the existing Vault-backed ClusterExternalSecret.
Do not enable the Fleet chart's own ingress: it must not co-own the
Infrastructure ingress.

The existing wildcard DNS and wildcard tunnel-to-Traefik route serve the MCP
hostname without adding another tunnel. Verify external reachability after
rollout; DNS resolution alone is not evidence that the route works.

## 3. Exposure and MCP clients

Use the existing Cloudflare Tunnel -> Traefik -> ClusterIP path with origin
TLS. Keep the Harbormaster service private; do not expose a public NodePort.
The ingress must preserve the public Host. Harbormaster ignores untrusted
forwarded identity headers and checks Host and Origin against its configured
public origin.

Do not attach browser-redirect forward-auth middleware to the MCP route:
native clients need OAuth `401` challenges, not HTML login redirects.
Protect `/mcp` in the application. Permit access to protected-resource
metadata without a token:

```text
https://mcp.nnvuf.uk/.well-known/oauth-protected-resource/mcp
```

The metadata points clients at Authentik discovery. OAuth clients request
`marina.read offline_access` for resource `https://mcp.nnvuf.uk/mcp`, perform user login and
PKCE, then send `Authorization: Bearer <access token>`. Cookies are rejected
in OAuth mode. CORS for arbitrary browser origins is not enabled.

Ensure Authentik discovery, JWKS, authorization and token endpoints permit
the relevant native/server requests through Cloudflare without interactive
bot challenges. Do not disable edge security globally.

## 4. Acceptance and rollback

Before public acceptance, test with two users:

1. Verify discovery and a cookie-free `401` with `WWW-Authenticate` metadata.
2. Log in via a native MCP client using PKCE and its registered callback.
3. List, search and retrieve each user's source; confirm results match that
   user's existing Chartroom collections and do not cross owner boundaries.
4. Reject expired tokens, wrong audience/issuer, missing UID/scope, cookie plus
   bearer, forged identity headers, and write-shaped calls.
5. Confirm browser CSRF/authentication and stdio acceptance still pass.

### Deployment record: 2026-11-02

- Infrastructure revision `2f128fc` reconciled the runtime ConfigMaps, TLS-secret
  distribution and ingress. The existing wildcard edge route serves the host.
- Lighthouse `v1.1.2`, source `3bd4299`, image
  `sha256:8b3dc2ef74790d6f9b082be59882ff9262554b21eff96f86c8b82e20e289e2e0`.
  This supersedes `v1.1.0` and `v1.1.1`: live HTTPS JWKS checks identified the
  required service User-Agent, and release-scan findings required PyJWT 2.14.0.
- Harbormaster `v0.2.0`, source `aeaf0ff`, image
  `sha256:43bf30647a186a96bc71513cf37f1d5ebe4090a9a1a4c1e61034ec3a1c241833`.
- Fleet promotions #19 and #18 were merged in that order. Runtime activation
  is recorded in `e6628bc` (Lighthouse) and `ae4c7e6` (Harbormaster).
  Both HelmReleases are Ready at Fleet revision `ae4c7e6`; Lighthouse API/worker
  readiness and HTTPS key retrieval passed before Harbormaster was activated.
- Local regression tests: 94 Lighthouse auth tests, 21 Harbormaster tests;
  Harbormaster build and production dependency audit passed.
  Both release pipelines passed validation, tests, signing and provenance.
- Live public discovery returned 200; unauthenticated MCP returned 401 with
  resource metadata, without an HTML redirect. A sixty-second synthetic JWT,
  signed in memory by the dedicated provider, passed public initialization,
  listed exactly the three read tools and reached Lighthouse's independent
  verifier. The nonexistent synthetic owner saw no collections. Wrong audience,
  expired token, absent scope/UID, mixed cookie/bearer and wrong origin were
  rejected. Direct Lighthouse writes with the scoped token returned 403.
  No user, grant, access-token record or application data was created.
- Existing stdio initialized and listed three tools, but its saved local cookie
  was rejected with "supply a current session"; authenticated stdio reacceptance
  needs a current user cookie. Neither local MCP configuration nor credentials
  were replaced. Chartroom still returns its expected unauthenticated 302.
- Interactive Authentik login, callback interoperability and two-real-user
  list/search/source isolation are **not accepted** by the synthetic checks.

Release scans remain report-only and are **not clean**: Lighthouse's final
image report contains 3 Critical and 65 High OS-package findings plus 2 High
`python-multipart` findings. The patched PyJWT has no reported findings.
Harbormaster's image report contains 6 High findings in bundled Node/npm
packages; its application production dependency audit reports zero. Base-image,
multipart and bundled-tool remediation remain a separate follow-up; this
deployment must not be described as security-accepted.

### Connect VS Code

Use **MCP: Add Server** -> **HTTP** with `https://mcp.nnvuf.uk/mcp`.
When prompted for a pre-registered OAuth client, enter `marina-mcp-vscode`,
then sign in to Authentik as `akadmin` and approve the read scope. Keep the
existing stdio server as a separate entry if it is still needed. Do not copy
cookies or access tokens into the HTTP configuration.

Only the two registered callbacks listed above are currently supported.
Additional agents require their actual callback URI to be registered first.

### Silent renewal

Refresh grants and the managed `offline_access` mapping are enabled. Harbormaster
advertises `marina.read offline_access` in both metadata and challenges so
clients request the refresh scope rather than only the read scope. No shared
credentials or client secrets are added. Log in once again after updating the
server: an existing read-only grant cannot acquire offline access by refresh.
The client must securely store and replace the rotated refresh token.

The real Authentik token view was exercised inside a rolled-back transaction
with a temporary test user and refresh grant: renewal returned 200, preserved
the user UID/resource audience, retained a 300-second access lifetime, rotated
the refresh token and refused reuse of the old token. Removing the temporary
user from the access group withheld resource claims on the next renewal.
No test users or token records were retained. This verifies server-side
renewal, not the current client's token storage or a silent refresh after its
next fresh interactive login.

Deployed Harbormaster `v0.2.1` (source `9a1599c`, release pipeline
`41459966498`) through Fleet promotion #20, revision `7e0872f`. Image:
`sha256:b9ef509d94b2dab2f46489fe2181e77d7923d6540279b56465c1bc8cecf9a542`.
The HelmRelease is Ready; public metadata and the 401 challenge both advertise
`marina.read offline_access`. A refreshed token successfully executed
`marina_list_collections` through the public gateway and Lighthouse after
rollout. All 21 application tests and the build passed. Image scans retain the
same six report-only High findings in bundled Node/npm packages.

The operator reported successful interactive login after the callback fix.
At that rollout, client-side silent renewal still required a fresh login
consenting to the new scope and verification beyond the five-minute access-token
lifetime. The November 3 acceptance below closes that gate for the current client.

### Real-client acceptance follow-up: 2026-11-03

The existing `marina-mcp-vscode` connection successfully searched the synthetic
PDF fixture and retrieved its exact cited source through public MCP. Search and
source agreed on document `24ceb094-f020-4f45-ac42-3cc66f9a50b6`, chunk
`85117979-f518-52bb-942a-c601a21f3725`, version 1, physical page 2 and lines 1-4,
including the emerald-green beacon fact. No private source content was needed.

A read-only eligibility check initially found only `akadmin` enrolled in
`marina-mcp-users`. The operator subsequently requested the second human account
`nnvu.fit` for this test. It was created as an active internal, non-admin user
with only `marina-mcp-users` membership. An independent database read verified
persistence, no administrator-group membership, an unusable password and
continued active `akadmin` access. No existing account was changed.

The operator set the new user's password privately through Authentik's
administrator user controls before interactive login. No password, recovery link
or token was generated or exported by the assistant. Rollback is to remove this user's
MCP membership or deactivate it; do not remove the existing administrator.

The current client obtained five-minute access tokens with
`marina.read offline_access` without a new interactive prompt. Read-only token
metadata showed expiry at `2026-11-03T16:11:42Z`. After waiting 310 seconds,
the same connection successfully repeated search and exact source retrieval
after `16:12:20Z`, without a new sign-in, client restart or manual token exchange.
New access-token expiry was `16:17:26Z`; the previous refresh record was revoked
and a rotated replacement was active. Original authentication time was unchanged.
Neither token values nor session IDs were read or exported.

**Client-side silent renewal is accepted for this VS Code connection.**
This does not establish two-user isolation, every other MCP client's renewal,
browser-session renewal, or seven-day refresh-token expiry behavior.

After private password setup, both users completed separate real browser logins.
The live isolation harness verified their usernames through Authentik and passed
own collection listing/search/exact-source access plus symmetric 404 refusals
for the other user's known fixture collection/document/chunk IDs. No sessions
or credentials were exported.

The existing native MCP connection then passed the `akadmin` half: its fixture
was listed, the `nnvu.fit` collection was absent, own search/source matched, and
both cross-user search/source requests were refused. The `nnvu.fit` native MCP
half subsequently passed after the operator switched the native connection:
only its own fixture collection was listed, own search/source returned the exact
version-1 marker, and `akadmin` search/source both returned "Not found or not
readable by this identity" with no foreign content. Both directions of native
MCP isolation are accepted, independently of the browser checks. The earlier
post-expiry renewal acceptance applies to the tested `akadmin` connection;
post-expiry renewal as `nnvu.fit` has not been separately measured.
Synthetic fixtures remain pending owner-authenticated cleanup:

| Owner | Collection | Document | Chunk |
| --- | --- | --- | --- |
| `akadmin` | `a0181368-56e4-4691-8b30-68ae03d2b553` | `d7696670-d60d-4976-8b30-a902973cb94c` | `abb1a5cd-8018-5e34-8f3e-a2cd0450fe24` |
| `nnvu.fit` | `27c08d36-1ce5-4a79-80d4-4961231473a3` | `0fd61cd4-cceb-46c5-a03b-f37d1d4166b9` | `c655d177-26c7-5235-9e1e-382421bc886b` |

Delete both documents through their owning Chartroom sessions after MCP
acceptance and confirm terminal deletion; empty collections remain.

#### Two-user acceptance procedure

1. The operator chooses two active human accounts and explicitly approves any
   needed group enrollment. Each person signs in themselves. Do not share
   cookies, access/refresh tokens, or saved browser contexts.
2. In separate authenticated Chartroom sessions, create a small synthetic
   collection/document for each user with distinct markers, such as
   `MARINA_ISOLATION_A` and `MARINA_ISOLATION_B`. Record only fixture collection,
   document and chunk IDs. Confirm ingestion is ready.
3. Use separate VS Code profiles/windows so OAuth sessions cannot be mixed.
   Connect both to the same MCP URL/client registration using their own PKCE
   login and read/offline consent.
4. As each user, list collections, search their own fixture and retrieve its
   exact source. Verify the other fixture is absent from listing. Supply the
   other fixture's known collection ID to search and known document/chunk IDs
   to source retrieval: both must refuse without returning the other marker
   or source text. Repeat symmetrically; an empty user library is not sufficient
   positive-path evidence.
5. After the five-minute access lifetime, repeat each user's own and cross-user
   checks without signing in again. Record actual client behavior separately
   from server refresh tests. No token values belong in the report.
6. Delete both synthetic documents using their owning Chartroom sessions and
   confirm terminal deletion. Record any retained empty fixture collections
   because collection deletion is not supported.

To disable offline access, remove the refresh grant and managed offline scope
mapping, restore the authorization-code-only policy, and revoke existing refresh
records for this provider. Keep the corrected trailing-slash callback.
Rolling back only the advertised scopes does not revoke already-issued refresh
tokens.

Rollback: revert Fleet activation commit `ae4c7e6` to restore Harbormaster to
stdio/health-only defaults, remove the public
MCP route, and unset the three Lighthouse MCP OIDC settings together. Browser
and stdio credentials remain independent. Disable the new Authentik application
and revoke its grants; allow outstanding short-lived JWTs to expire.
Removing Fleet's `lighthouse-mcp-oauth` reference (commit `e6628bc`) disables
the additional verifier without changing the existing `lighthouse-runtime`
ConfigMap, credentials or originals PVC. Do not delete the `marina-mcp`
Namespace during rollback.

References:

- [Authentik OAuth provider](https://docs.goauthentik.io/add-secure-apps/providers/oauth2/)
- [MCP HTTP authorization](https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization)
