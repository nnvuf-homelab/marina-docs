# MARINA expansion plan

This plan covers the work that follows the MARINA Starter Pack v1.0 baseline.
It expands [section 5 of the development plan](DEVELOPMENT-PLAN.md) into
milestones with explicit ownership, acceptance criteria, and deferrals.

The starter pack is accepted and deployed. Treat every milestone here as a
production-bound change: the same admission gates that accepted Milestone 6 must
pass again before any milestone is called complete.

## 1. Current baseline

| Component | State |
| --- | --- |
| Lighthouse | Deployed in `marina-core`; API, worker, PostgreSQL, originals PVC |
| Chartroom | Deployed in `marina-ui` behind Authentik forward auth |
| Harbormaster | Scaffold only in `marina-mcp`; tool schemas and policy boundary |
| Trawler | Scaffold only in `marina-fetch`; target validation only |
| Fleet | Promotes all four releases; drift detection enabled |
| Shipyard | Builds, tests, and opens promotion pull requests |

Inactive by design: `agentic-rag` and `knowledge-learning` run at zero replicas.
Do not scale them until a milestone below requires them.

## 2. Gate 0 - establish the v1.0 baseline

Complete before starting Milestone 7. This is release hygiene, not new features.
Milestone 6's historical acceptance does not establish current Gate 0 readiness.

### 0.1 Recover and verify runtime dependencies

- Vault must be initialized, unsealed, and Ready. After a Vault restart, follow
  the [existing-instance recovery procedure](https://github.com/nnvuf-homelab/infrastructure/blob/main/docs/runbooks/vault.md#recover-an-existing-sealed-instance).
  The operator enters existing unseal shares privately; never reinitialize,
  replace its PVC, or disclose shares in chat.
- Require `vault-homelab` to be Ready and every ExternalSecret using it to be
  Ready with a successful refresh after recovery. Existing application Secrets
  keep pods running during an outage; healthy pods are not evidence of working
  secret synchronization.
- Confirm Flux sources, Kustomizations, and application HelmReleases are Ready
  and unsuspended. Match API/worker Pod `spec.containers[].image` to Fleet's
  pinned Lighthouse digest; do not compare an image index to its platform
  `imageID`.
- Require API and worker `/ready` to return 200 with every reported check true,
  and PostgreSQL/originals PVCs to remain Bound.

### 0.2 Repeat authenticated acceptance

- Run the [production smoke](PRODUCTION-RUNBOOK.md#5-public-admission-gates)
  with a current, non-sensitive test user's Authentik session. Require exit
  code 0 and a report with `passed: true`; an anonymous 302 is only a challenge,
  not authenticated acceptance.
- Repeat upload, indexing, questioning, citation inspection, and deletion
  through Chartroom's public browser session.
- Supply the cookie only through the operator's process environment and clear
  it after testing. Never store it in reports or commit it.
- The API smoke writes a fixture, deletes its document, and leaves an empty
  collection. Record that expected residue rather than claiming zero changes.

### 0.3 Establish recoverable off-host backup custody

**Deferred to Milestone 10 by the operator on 2026-11-01.** This is no longer
a Gate 0 prerequisite. The following requirements remain planned, not passed:
loss of this Windows host can still make application data and DPAPI-protected
recovery material unavailable. Retain the existing local encrypted backups;
do not describe this deferral as verified disaster recovery.

- Retain a verified metadata/originals backup made before Milestone 7 changes.
  If production data has changed since the accepted backup, take and verify a
  newer backup using the [production backup procedure](PRODUCTION-RUNBOOK.md#4-consistent-backup-and-isolated-restore).
- Store an encrypted copy outside the Windows machine hosting the WSL2
  control plane, with its hash and restore report. A copy in this workstation's
  session directory is not protection against host loss.
- Keep key recovery material separately in an approved external password
  manager or offline store. The October 18 key file uses Windows DPAPI
  CurrentUser: copying that file alone does not prove it is decryptable on
  another machine. Require a verified independent recovery method.
- Verify decryption and isolated restore from the retained copy, and record
  the verification date and a nonsecret custody reference. Do not restore
  over production or put keys/plaintext archives into Git.

### 0.4 Freeze the verified release baseline

- After runtime and authenticated acceptance gates pass, record source commits, full image digests,
  Fleet revision, and Infrastructure revision together. A deployment tag is a
  code reference, not a substitute for database/original backup recovery.
- Tag accepted Lighthouse and Chartroom source commits as `v1.0.0`, and record
  the corresponding Fleet/Infrastructure commits as baseline references.
  Harbormaster and Trawler remain scaffolds, not accepted v1.0 features;
  record their commits without marking them accepted.
- Inspect tag-triggered workflows before publishing tags. The current
  Lighthouse and Chartroom delivery workflows run for `v*` tags; publishing a
  tag can build an image and open a Fleet promotion. Do not silently merge
  that promotion or assume its digest matches the existing deployment.
  Revalidate any resulting promoted image.

Acceptance criteria:

- Runtime dependencies, secret synchronization, and authenticated public
  workflow all pass with current evidence.
- Off-host recovery is explicitly deferred to Milestone 10 with the host-loss
  risk acknowledged; the retained local backup remains readable.
- Accepted app source tags and recorded image/Fleet/Infrastructure references
  identify the tested baseline without claiming scaffold acceptance.

Latest check: **2026-11-02 - Gate 0 accepted (off-host backup deferred).** Fleet `0111186d` is Ready;
API and worker run Lighthouse digest
`sha256:415557940285558473f17c59b65bf6eebc22dc2da2e973dc189d69e97c028e5d`,
and API readiness is healthy. Vault was recovered on November 1 using its
retained encrypted initialization backup without saving plaintext. Vault and
`vault-homelab` are Ready, and all 21 ExternalSecrets successfully refreshed
at or after 2026-11-01T16:08:19Z following a requested controller refresh.
The authenticated production API smoke passed on November 1; its local report
confirms authentication challenges, forged-identity rejection, CSRF rejection,
and grounding passed with fixture document cleanup. The operator confirmed
the public Chartroom browser workflow passed on November 1. Independent
backup/key recovery is deferred to Milestone 10. Both `v1.0.0` release pipelines
passed and their Fleet promotions are merged. After the new API/worker pods
(2026-11-02 00:03:25 +07:00) and Chartroom pod (00:07:46 +07:00) started,
the authenticated production API smoke passed again at 00:14:47 +07:00. That
report covers challenges, forged-identity rejection, direct API check, CSRF
rejection, readiness, operations, and grounding. The operator then confirmed
the public Chartroom browser workflow passed on these images. An earlier
attempt at 00:11:44 failed only because a stale session cookie was rejected
by forward-auth before reaching the API. The October 18 encrypted archive's hash matches
its recorded metadata; that check is not a new restore rehearsal.

Additional checks on 2026-11-01:

- The isolated Chartroom browser-session test passed, covering cookie/Origin
  handling and 401/503 recovery with mocked API responses. This is not the
  authenticated public browser workflow.
- The retained October 18 backup passed encrypted-file SHA-256, HMAC,
  current-user DPAPI decryption, database-dump hash, and all original hashes
  (one original), entirely in memory without writing a plaintext archive.
  This proves local readability, not independent host recovery or a new restore.
- Published `v1.0.0` tags and successful release runs map to the deployed images:
  Lighthouse commit `4aaadbc4272f0495053d3f2da9694508f97557ff`,
  [run 41328616720](https://github.com/nnvuf-homelab/lighthouse/actions/runs/41328616720),
  [Fleet PR 13](https://github.com/nnvuf-homelab/fleet/pull/13);
  Chartroom commit `c028cb5d50975fbb3af66e75dd4002e055cf0aad`,
  [run 41329406283](https://github.com/nnvuf-homelab/chartroom/actions/runs/41329406283),
  [Fleet PR 15](https://github.com/nnvuf-homelab/fleet/pull/15).
  Chartroom's deployed digest is
  `sha256:f1a58073412f2674a382c15bbfd05975f4042bf95ec6b54c59bed319e980b9a1`.
  Infrastructure's applied revision remains `27cc3bdb`.
- No independent backup destination was supplied. Docker Desktop's local
  daemon is unavailable, so a new disposable local database restore was not
  run. Production was not interrupted for a restore drill.

### Gate 0 checklist

| Check | Latest status |
| --- | --- |
| Vault, store, and 21 ExternalSecrets | Reverified healthy on 2026-11-02 |
| API/worker readiness and promoted image | Passed on 2026-11-02 for new release images |
| Authenticated API smoke | Passed on 2026-11-02 against v1.0.0 images |
| Public Chartroom browser workflow | Passed on 2026-11-02 against v1.0.0 images |
| Independently recoverable off-host backup | Deferred to Milestone 10; host-loss risk remains |
| Accepted app tags and release references | v1.0.0 tags, CI, and deployed source/image mapping verified |

### Check runtime dependencies (WSL Bash)

Run from the workspace root. This is read-only; it does not unseal Vault,
refresh Secrets, restart pods, or reconcile Flux. A nonzero exit means the
runtime gate failed. The script checks the currently observed Fleet artifact
revision; it does not prove that the source has fetched the newest remote commit.

```bash
set -euo pipefail
export PATH=/usr/local/bin:$PATH
context=homelab

kubectl --context "$context" exec -n vault vault-0 -- vault status -format=json |
  python3 -c 'import json,sys; x=json.load(sys.stdin); assert x.get("initialized") is True and x.get("sealed") is False, "Vault is not initialized and unsealed"; print("Vault: unsealed")'
kubectl --context "$context" get pod vault-0 -n vault -o json |
  python3 -c 'import json,sys; x=json.load(sys.stdin); assert any(c.get("type")=="Ready" and c.get("status")=="True" for c in x.get("status",{}).get("conditions",[])), "Vault pod is not Ready"'
kubectl --context "$context" get clustersecretstore vault-homelab -o json |
  python3 -c 'import json,sys; x=json.load(sys.stdin); assert any(c.get("type")=="Ready" and c.get("status")=="True" for c in x.get("status",{}).get("conditions",[])), "Vault store is not Ready"; print("Vault store: Ready")'
kubectl --context "$context" get externalsecret -A -o json |
  python3 -c '
import datetime as dt,json,sys
items=[x for x in json.load(sys.stdin)["items"] if x.get("spec",{}).get("secretStoreRef",{}).get("name")=="vault-homelab"]
assert items, "No Vault ExternalSecrets found"
cutoff=dt.datetime.fromisoformat("2026-11-01T16:08:19+00:00")
for x in items:
    name=x["metadata"]["namespace"]+"/"+x["metadata"]["name"]
    status=x.get("status",{})
    assert any(c.get("type")=="Ready" and c.get("status")=="True" for c in status.get("conditions",[])), name+" is not Ready"
    refreshed=status.get("refreshTime")
    assert refreshed, name+" has no successful refresh"
    assert dt.datetime.fromisoformat(refreshed.replace("Z","+00:00"))>=cutoff, name+" has not refreshed after recovery"
print("Vault ExternalSecrets: %d/%d Ready and refreshed" % (len(items),len(items)))'
kubectl --context "$context" get gitrepositories,kustomizations,helmreleases -n flux-system -o json |
  python3 -c '
import json,sys
items=json.load(sys.stdin)["items"]
assert items, "No Flux resources found"
for x in items:
    name=x["kind"]+"/"+x["metadata"]["name"]
    assert not x.get("spec",{}).get("suspend",False), name+" is suspended"
    assert any(c.get("type")=="Ready" and c.get("status")=="True" for c in x.get("status",{}).get("conditions",[])), name+" is not Ready"
source=next(x for x in items if x["kind"]=="GitRepository" and x["metadata"]["name"]=="fleet")
group=next(x for x in items if x["kind"]=="Kustomization" and x["metadata"]["name"]=="fleet")
revision=source["status"]["artifact"]["revision"]
assert group["status"]["lastAppliedRevision"]==revision, "Fleet has not applied its fetched revision"
print("Flux: Ready; Fleet applied "+revision)'
expected=ghcr.io/nnvuf-homelab/lighthouse@sha256:415557940285558473f17c59b65bf6eebc22dc2da2e973dc189d69e97c028e5d
for role in knowledge-api knowledge-worker; do
  kubectl --context "$context" get pods -n marina-core -l "app.kubernetes.io/name=$role" -o json |
    python3 -c '
import json,sys
items=json.load(sys.stdin)["items"]
assert items, "No role pods found"
for x in items:
    assert not x["metadata"].get("deletionTimestamp"), "Role pod is terminating"
    assert x["spec"]["containers"][0]["image"]==sys.argv[1], "Image differs from recorded baseline"
    assert any(c.get("type")=="Ready" and c.get("status")=="True" for c in x.get("status",{}).get("conditions",[])), "Role pod is not Ready"
print("Pod image and readiness: passed")' "$expected"
  kubectl --context "$context" exec -n marina-core "deploy/$role" -- python -c '
import json,urllib.request
with urllib.request.urlopen("http://127.0.0.1:8080/ready",timeout=20) as r:
    assert r.status==200, "Readiness HTTP status failed"
    x=json.load(r)
assert x.get("status")=="ready" and x.get("checks") and all(v is True for v in x["checks"].values()), "Readiness checks failed"
print(x["role"]+": all readiness checks passed")'
done
kubectl --context "$context" get pvc -n marina-core data-lighthouse-postgres-0 lighthouse-originals -o json |
  python3 -c 'import json,sys; a=json.load(sys.stdin)["items"]; assert len(a)==2 and all(x.get("status",{}).get("phase")=="Bound" for x in a), "Required PVCs are not Bound"; print("PVCs: Bound")'
echo "Runtime checks passed; remaining Gate 0 checks are still required."
```

The pinned digest deliberately fails if another image is deployed. Review the
new promotion and update the accepted reference only after testing it.

### Check authenticated acceptance (Windows PowerShell)

Sign in to Chartroom, then copy only the complete `Cookie` request-header value
from a successful Marina `/v1/session` request. Do not paste it into this
document or chat. Run from the workspace root:

```powershell
Set-Location D:\Workspace\homelab\lighthouse
$env:MARINA_EVALUATION_COOKIE = (Get-Clipboard -Raw).Trim()
Set-Clipboard -Value " " -ErrorAction Stop
try {
    if ([string]::IsNullOrWhiteSpace($env:MARINA_EVALUATION_COOKIE)) {
        throw "No session cookie was supplied."
    }
    & .\.venv\Scripts\python.exe evaluation\production.py `
        --api-url https://marina.nnvuf.uk `
        --origin https://chartroom.nnvuf.uk `
        --login-prefix https://auth.nnvuf.uk/ `
        --output evaluation-report-production-public.json
    if ($LASTEXITCODE -ne 0) { throw "Production smoke failed." }
    $report = Get-Content -Raw evaluation-report-production-public.json | ConvertFrom-Json
    if ($report.passed -ne $true) { throw "Production report did not pass." }
    Write-Output "Authenticated API smoke passed; now repeat the Chartroom browser workflow."
} finally {
    Remove-Item Env:\MARINA_EVALUATION_COOKIE -ErrorAction SilentlyContinue
}
```

This runs the existing evaluator, writes only its gitignored report, and changes
production only through the documented fixture upload/deletion. A failed smoke
is not acceptance; investigate its error rather than increasing redirect limits.
The clipboard command replaces the cookie with a single space because some
Windows PowerShell versions reject an empty string. It does not clear clipboard
history or previously synchronized clipboard copies; remove those separately
if clipboard history or synchronization is enabled.

### Check baseline tags (Windows PowerShell)

This only lists published tags. Empty output means no matching tag exists; it
does not create tags, trigger CI, or prove a tag's commit matches an image.

```powershell
foreach ($repo in @("lighthouse", "chartroom")) {
    Write-Output "--- $repo published baseline tags ---"
    git -C "D:\Workspace\homelab\$repo" ls-remote --tags origin "refs/tags/v1.0.0"
    if ($LASTEXITCODE -ne 0) { throw "Cannot read tags for $repo." }
}
```

Off-host backup custody and independent key recovery require an actual retained
copy and a tested recovery environment. Do not mark that gate passed from a
local file-existence/hash check alone. Section 0.3 is deferred to Milestone 10;
record the nonsecret custody reference and restore date when implementing it.

## 3. Milestone 7 - Harbormaster read-only MCP

Owner: [Harbormaster](../harbormaster). Estimated 1-2 weeks.

**Closed by the operator on 2026-11-03 for the accepted read-only scope.**
Real-client list/search/source retrieval and write refusal are accepted.
Two-real-user remote OAuth isolation and client-side silent renewal remain
explicit follow-up checks, not claims made by this closure. No write-capable
MCP tools are authorized.

Expose MARINA knowledge to MCP clients without granting any write path.

Scope:

- Implement `marina_list_collections`, `marina_search`, and `marina_get_source`
  against the existing Lighthouse API. Add `marina_ask` only after the first
  three are stable.
- Ship stdio transport first. Keep the server a thin, auditable client of
  Lighthouse; it must hold no database, vector, or object-store credentials.
- Enforce the policy boundary in code: reject any tool call that would create,
  update, or delete Lighthouse state, and cap result counts and payload sizes.
- Log every tool invocation with the requesting identity and the collection
  touched.

Open decision - remote transport authentication:

Remote MCP needs a dedicated Authentik OAuth application with delegated,
per-user identity. A shared service identity is not acceptable because it would
let any MCP client read every collection. Resolve this decision before writing
remote transport code; until then, stdio with an operator-supplied credential is
the supported mode.

Acceptance criteria:

- A real MCP client lists collections, searches, and retrieves a source.
- Every write-shaped request is refused and the refusal is tested.
- Results stay scoped to collections the calling identity may read.
- No backing-store credential is mounted into the Harbormaster workload.

Status (2026-11-02): **Milestone 7 stdio scope accepted; release promotion pending.**
Harbormaster 0.2.0 replaces the scaffold's mismatched tools with
`marina_list_collections`, `marina_search`, and `marina_get_source` over stdio.
A code-level route allow-list refuses every other Lighthouse method and path
before any network call. Results are capped per item and at 64 KB per call, and
each call writes a stderr audit line (credential fingerprint, collection or
document, outcome) without logging queries or credentials. The operator
credential is the user's own Authentik session cookie, which VS Code's user-level MCP configuration
prompts for as a password input, so Lighthouse's per-owner scoping applies
unchanged. The default in-cluster URL now targets `marina-core`. The cluster
workload still carries no credential and serves health only. Unit and
in-memory MCP protocol tests (13) and a built-server stdio check pass.
Production acceptance on November 2 passed through a real JSON-RPC stdio
client launched with the Windows encrypted-cookie launcher: four collections
were listed, a collection search returned a citation, and source retrieval
returned matching document/chunk IDs and nonempty text. A delete-shaped tool
call was refused without any mutation. Existing Lighthouse owner-scoped routes
remain the authorization boundary; this check used one operator identity,
not a new multi-user isolation test.

Local VS Code chat access is confirmed by the operator. The encrypted-cookie
launcher also passes production acceptance independently of chat. Access from
the existing CLI-backed chat remains unverified and is not a prerequisite for
the accepted stdio scope. Further chat-host troubleshooting and public remote OAuth deployment are deferred.

Historical remaining release steps (completed by the rollout recorded below): commit the accepted implementation, tag `v0.2.0`,
and review the resulting Fleet promotion. No tag, push, or live rollout is
claimed by this acceptance record.

Streamable HTTP follow-up (2026-11-02): implemented as an opt-in stateless
`/mcp` endpoint, restricted to loopback. Each request supplies the user's own
Authentik session cookie; shared server credentials are refused. Host/Origin
checks and a 32 KB request limit apply. Tests cover all three tools, distinct
request credentials, write refusals, malformed exposure configuration,
oversized requests, and a real Node HTTP initialization exchange. This is
local HTTP transport support, not public remote deployment or delegated OAuth
acceptance. Stdio and the cluster's health-only default remain unchanged.

OAuth follow-up: Harbormaster's opt-in HTTP OAuth mode validates signed
per-user Authentik JWTs and publishes protected-resource discovery metadata.
Lighthouse's optional MCP OIDC verifier preserves browser authentication and
maps the signed `marina_uid` claim to existing owner keys, accepting bearer
tokens only on the three read routes. Stdio is unchanged. The
[Authentik integration runbook](../Infrastructure/docs/runbooks/marina-mcp-oauth.md)
documents PKCE client registration, claim/audience mapping, activation order,
acceptance and rollback. On November 2 the dedicated Authentik application
was provisioned with public client ID `marina-mcp-vscode`, exact VS Code
callbacks, authorization-code-only access, S256 policy and five-minute signed
tokens. Public discovery/JWKS and in-memory token/policy checks passed.
The operator explicitly enrolled `akadmin` in `marina-mcp-users`. On November 2,
Lighthouse `v1.1.2` and Harbormaster `v0.2.0` were deployed through Fleet, with
Infrastructure-owned runtime ConfigMaps, TLS-secret distribution and ingress.
Public discovery, OAuth 401 challenges, a nonpersisted synthetic signed-token
read, claim refusals and backend write refusal passed. Both workloads are Ready.
The existing stdio transport initializes and lists its tools; its saved local
cookie was rejected at rollout and needed renewal. At that point real PKCE login
and two-user list/search/source isolation were still pending; synthetic checks
do not constitute interactive acceptance. Image scans retain report-only base/dependency findings,
documented in the integration runbook; no security acceptance is claimed.

Acceptance update (2026-11-03): the operator reported successful native
Authentik login after correcting the exact loopback callback. A real call
through `marina-mcp-vscode` listed four owner-visible collections (one `test`
and three `Grounding adversarial acceptance` collections). The original
Milestone 7 stdio scope remains complete. Public HTTP listing is verified,
and two-real-user isolation subsequently passed. On November 3, the real
`marina-mcp-vscode` connection also searched the synthetic PDF beacon fixture
and retrieved its exact page-2 source with matching document/chunk/version/lines.
The operator subsequently approved creating `nnvu.fit` as the second human
account. It is active, non-admin and enrolled only in the MCP access group;
the operator completed private password setup. Both actual browser identities
passed own reads and symmetric cross-user 404 refusals. Native MCP checks also
passed for both identities after an operator account switch: own collection,
search and exact source were readable, the foreign collection was absent,
and known foreign search/source IDs were refused without source disclosure.
The two synthetic documents still require owner-authenticated cleanup.
The same connection repeated search/source calls after a 310-second wait past
the observed five-minute token expiry, without sign-in or a manual token
exchange. Read-only Authentik metadata confirmed new access expiry and refresh
rotation with the old refresh record revoked. Client-side silent renewal is
accepted for the tested `akadmin` VS Code connection; renewal as `nnvu.fit`
and other clients is not implied by the separate isolation acceptance.
Rotating refresh grants and offline-scope discovery are deployed; server-side
refresh, replay refusal and refreshed-token public reads passed controlled
tests. These checks do not substitute for client-side silent-renewal evidence.

## 4. Milestone 8 - Trawler controlled URL ingestion

Owner: [Trawler](../trawler). Estimated 2 weeks.

Status (2026-11-03): **closed by the operator; controlled ingestion accepted.** Private production acquisition,
dedicated PVC persistence, authenticated Lighthouse handoff, indexing, retry,
deduplication and cited retrieval passed. Chartroom returned source-grounded
answers with working citation inspection; completeness and requested numbering
are tracked separately as Lighthouse answer-presentation work below.
No public Trawler ingress is authorized.
The accepted operating model remains manual, internal operator use through
`trawler-ingest`; no automated application consumer or scheduled crawler is
required. Here "internal" describes the tool's exposure, not fetch destinations:
private/LAN URL ingestion remains unsupported and no private-address bypass or
authentication expansion is part of this milestone.

Implementation boundary:

- Trawler owns public HTTPS acquisition, normalization and a local SQLite
  acquisition/outcome journal. It never receives backing-store credentials,
  user cookies or Lighthouse bearer tokens.
- An operator-side `trawler-ingest` command fetches through Trawler, then sends
  the normalized UTF-8 document and provenance through Lighthouse's existing
  authenticated multipart document-upload route. The user's credential is sent
  only to Lighthouse; the MCP read-only OAuth scope cannot authorize uploads.
- Acquisition and handoff outcomes are queryable by acquisition ID. Failed
  fetches can be retried; failed uploads can reuse the retained acquisition
  without refetching. This is single-URL operator ingestion, not autonomous
  crawling, browser rendering or a new Lighthouse ingestion route.
- The acquisition API is private/operator-only. Local execution binds loopback;
  cluster rollout needs Infrastructure-owned storage/runtime wiring, a retained
  state volume, no public ingress and a no-service-account-token workload check.

Fetch a single operator-supplied URL and hand its content to Lighthouse
ingestion. This is the highest-risk milestone in the plan because it makes the
cluster issue outbound requests on request.

Required SSRF boundary, enforced in code and covered by tests:

- HTTPS only. Reject every other scheme.
- Resolve DNS explicitly and reject private, loopback, link-local, multicast,
  and cloud metadata addresses.
- Protect against DNS rebinding by connecting to the validated address rather
  than re-resolving the hostname.
- Revalidate every redirect hop against the same rules; cap the hop count.
- Enforce response size, total timeout, and an allowed MIME type list.
- Run with no service account token and no backing-store credentials.
- No unrestricted headless browser. If JavaScript rendering is ever needed, it
  becomes its own milestone with its own review.

Scope:

- Accept one URL per request, validate it, fetch it, and return normalized text
  plus provenance metadata.
- Record the fetch outcome so a failed acquisition is visible and retryable.
- Hand content to Lighthouse through the existing ingestion path; do not add a
  second ingestion route.

Local implementation evidence:

- Trawler's full standalone suite passes **69 tests**, including HTTPS-only
  targets, nonpublic/mixed DNS, pinned-IP TLS hostname behavior, redirect
  revalidation/hop count, response MIME/encoding/byte limits, measured slow-stream
  deadline enforcement, two-request concurrency cap, failed-fetch visibility,
  interrupted-fetch recovery, and explicit failed-upload reporting.
- A real verified public HTTPS fetch of `https://example.com/` passed under
  Linux. Its captured normalized text/provenance passed the cross-repo
  Lighthouse integration test: existing authenticated multipart upload,
  migrations, worker processing, cited search, answer response, retained-result
  upload retry and exact-content deduplication. Model/vector/object-store
  dependencies use Lighthouse's test doubles; this is not live-model acceptance.
- Python compilation and the application wheel build passed. Shipyard v1 schema
  validation passed. The existing Fleet workload renders with
  `automountServiceAccountToken: false` and no backing-store secret injection.
- The new API journal and local Compose volume retain acquisition results and
  fetch/handoff errors. These local checks preceded the separately authorized
  production promotion and storage migration below.
- Windows Python's CA store refused the test site's certificate; acquisition
  passed with Linux CA trust instead. TLS verification remains mandatory.

Production evidence:

- Source `f91e7c0` passed Shipyard pipeline `41495660050`; Fleet promotion
  PR #21 merged as `1ab09c7`. The pinned image digest is
  `sha256:98f6fa6fa46f002f772282975535f9d1980607369254bfe55e6ea9edd091ea7b`.
- Infrastructure `c2faf8a` provisioned the retained 2 GiB, local-path RWO
  acquisition PVC. Fleet `1ed6e58` and `cad6dce` activated its mount, non-root
  group access and single-replica `Recreate` rollout.
- A consistent SQLite backup preserved both existing records. Following restore,
  a public acquisition returned 201 and a loopback target returned 422. All four
  complete records survived pod replacement unchanged; SQLite integrity, the
  unchanged PVC identity, Helm readiness and Service readiness passed.
- No public Ingress, service-account token or backing-store credential injection
  was added. Node-local storage is not replicated, the request is not a quota,
  and automatic retention remains unimplemented.

- The Foundry guide exposed unreachable IPv6 selection. Trawler `2bea1f7`
  added fallback across validated addresses while preserving the total deadline,
  TLS verification and mixed-DNS refusal. All **77 tests passed**. Pipeline
  `41499448171` succeeded; Fleet PR #22 merged as `7451129`, promoting
  `sha256:13f6b3e77a682b241eb1f0067a9d2c3a63f1eb9d8bd46dae9dcab8fa54871e1f`.
- The promoted API fetched the official Foundry SDK quickstart with HTTP 201
  (26,447 normalized characters); a loopback target still returned 422.
  Acquisition `e53a08ff-1f84-4606-a5c3-672bc8c8058f` retained the result.
- Operator-side authenticated handoff produced document
  `34e762e2-6c9e-4d47-aeb0-18bc10c1ebe3` in the existing `test` collection.
  Chartroom showed Ready. Repeating the handoff returned the same document ID,
  Ready, version 1. The preceding manual upload remains a separate document;
  no deduplication claim is made between the differently submitted files.
- A deliberately invalid session produced HTTP 302 without following the login
  redirect. The journal recorded `failed` / `Lighthouse HTTP 302` while keeping
  the acquisition. Retrying with a valid operator session restored the handoff
  to `queued`, cleared the error and returned the same Ready/version-1 document.
  `queued` is submission status, not indexing status. Credentials were cleared
  locally after use; no credential is part of this evidence.
- MARINA MCP search and source access returned the automatically ingested
  document, version 1, lines 81-120, including the Python SDK and CLI login
  instructions. User screenshots confirmed Chartroom answers and matching
  source inspection. The deployed source-first adapter uses the real model to
  select quotations, not to generate free-form summaries.

Lighthouse follow-up: the broad question omitted setup steps; a focused question
returned unrelated code; an explicit two-numbered-step request returned correct
commands but no numbering or per-step references. Source validity passed, but
presentation and coverage did not. Preserve the accepted source-first grounding
boundary while improving this behavior; do not describe formatting as fixed in
production before a separate promotion and browser recheck.

Investigation result: this is deliberate extractive assembly in
`OllamaModels.answer`, not a fallback or a Chartroom rendering defect. Sentence
selection can bundle newline-separated commands with unrelated HTML text.
An exact-line/numbered-output candidate was tested in memory against the live
`qwen2.5:7b` model with actual top-five MARINA retrieval, without changing deployed
files or accessing private credentials. Its explicit two-step answer passed,
but the broad question still returned only prerequisites and a section label;
the focused question could include unrelated navigation text. The candidate
was removed from the working implementation and not promoted. Complete coverage,
relevance and formatting remain open; the ingestion acceptance above is separate.

### Clean-document comparison and extraction follow-up (2026-11-03)

The operator uploaded a manually curated Python-only Markdown guide into a
separate collection, leaving the original test collection intact. Browser
comparison showed:

- The broad prerequisites/setup question still returned prerequisites only.
- The focused installation/authentication question returned both commands,
  without other languages or application code.
- The explicit two-numbered-step question included the right material but
  did not number the steps. Chartroom displayed code fences as literal text.

Thus extraction quality contributes to relevance but does not explain the
entire answer-quality gap. A further bounded multipart-selection candidate
was tested in memory with the live model against the clean guide. Focused
questions improved, but broad setup still omitted commands; unrestricted
planning also invented commands in intermediate questions. Requiring exact
user-text parts correctly rejected those invalid plans. That candidate was
removed; no Lighthouse runtime change is being promoted.

Promoted extraction/presentation follow-up:

- Trawler now prefers article/main content over page chrome, preserves Markdown
  headings, list boundaries, blank lines and preformatted code indentation,
  and refuses malformed/oversized code instead of silently flattening it.
  A production fetch of the official guide retained SDK/login instructions,
  headings and fenced code while removing browser/navigation notices.
- Chartroom renders fenced code as escaped React code blocks instead of literal
  delimiters. It does not interpret source HTML or execute code.

Fleet promotions for Trawler `a06ebee` and Chartroom `9521a80` were verified Ready
at Fleet `b60f38a`. New production acquisition
`d6953cc2-63a7-4fb9-a993-dc51a739ba63` contains 7,496 normalized characters,
23 headings and eight fence delimiter lines, with code indentation preserved
and navigation removed. Trawler's existing PVC identity and acquisition history
were preserved; SQLite integrity passed.

The operator's browser screenshot of the separate clean-guide collection
confirmed that both installation and login commands display as code blocks
without literal fences. It still did not produce the requested two numbered
steps. These are accepted extraction/presentation changes, not proof of complete
multipart answers. Reindexing the new automated acquisition in a separate
collection and comparing browser answers remain pending. Existing acquisitions
and documents are unchanged.

### Answer-quality follow-up (2026-11-03)

A concrete selector bug was reproduced: sentence splitting could break a code
string at a period, split a fenced block at blank lines, and treat its comments
as section headings. The local adapter now supplies a complete fenced block
within a retrieved chunk as one exact-text excerpt. It preserves comments,
blank lines and indentation without generating or repairing code. Source chunk
IDs, citation mapping, response shape and the selection prompt are unchanged.
Code spanning ingestion chunks is not reconstructed by this fix.

Both backtick/tilde regression cases fail on the original implementation.
With the fix, the targeted adapter/API/evaluation suite passes 103 tests with
one expected SQLite concurrency skip. Five adapter-only checks against the
deployed `qwen2.5:7b` digest also pass: complete code, beacon color, unsupported
installation date, ordered stages and sensor routing. These supplied-fixture
checks do not establish end-to-end retrieval or browser acceptance.

Separate grouped-selection candidates were tested against the clean and
automatically extracted guide. Direct grouping, section scoping and internal
requirement labels still produced omitted configuration/commands, misplaced
steps or grounding failures. Those candidates remain outside the implementation.
Complete multipart coverage and requested numbering remain open.

The code-preservation fix was committed as Lighthouse `a9e5341` and promoted
through Fleet `fced55e22e985e953112ea5e1d16e617eae6a6f6` with image
`sha256:19c228e77cb6936a756deff8129d19105ff9ddf4ed6495bd41ea7f14a031e271`.
Deployed source hashes match the commit. API and worker readiness returned 200
with all dependency checks true, retained PVCs remained Bound, and all six
in-pod real-model checks passed (backtick/tilde code, beacon color, abstention,
ordered stages and sensor routing). Those adapter checks wrote no documents;
they are not authenticated browser or full-retrieval acceptance.

Storage custody, rollout recovery and rollback restrictions are documented in the
[Trawler storage runbook](../Infrastructure/docs/runbooks/trawler-storage.md).

Operator commands and exact retry behavior are documented in
[Trawler's README](../trawler/README.md).

Acceptance criteria:

- A public URL ingests end to end and produces answerable, cited content.
- Every boundary rule above has a test that proves the request is refused.
- The workload runs without a service account token.
- Failed fetches surface in operational visibility, not only in logs.

## 5. Milestone 9 - PDF ingestion with page citations

Owner: [Lighthouse](.). Estimated 1-2 weeks.

**Accepted and complete (2026-11-03).** Functional and current recovery checks
passed. The operator approved closure with the historical pre-promotion
backup/rehearsal timing exception recorded below.
Lighthouse owns isolated extraction, unchanged raw originals, additive migration
`0002` and page-aware citations. Chartroom accepts PDF uploads and shows physical
pages and authenticated original links; Harbormaster carries optional page
metadata through its existing read-only tools. Trawler remains unchanged.

Implementation decisions:

- Only text-layer PDFs, up to 5 MiB, 100 pages and 1 MiB extracted UTF-8 text.
- A separate parser process has a 20-second wall-time limit and a 256 MiB memory
  limit; it does not inherit application credentials. Missing memory isolation
  is an explicit dependency failure. One parser runs per worker process.
- Encrypted, malformed, scanned/image-only or blank-page PDFs fail with a
  visible, actionable job/document error on the first attempt. OCR and inferred
  text/layout are not included. Blank pages must be removed before upload.
- Each chunk belongs to one physical page; citation lines are relative to that
  page's extracted text. Existing non-PDF response fields and chunk IDs remain
  unchanged. Originals are retrieved only by their owner and ready version;
  MCP tokens do not gain original-download access.
- Migration/rollout and rollback compatibility are gated separately in the
  [production runbook](PRODUCTION-RUNBOOK.md#6-rollback). Previous images require
  schema `0001` for readiness; an image-only rollback after `0002` is insufficient.

Local validation (2026-11-03):

- Lighthouse: 241 tests passed, eight skips (seven platform/PostgreSQL-dependent
  and the opt-in real-model test, run separately below);
  compile and wheel build passed. Upgrade from `0001` preserves existing text
  chunks. Tests cover PDF page boundaries/IDs, owner/version/hash checks,
  unchanged text citations, original-byte backup, deduplication, reindex,
  deletion, terminal parse errors and transient dependency retries.
- Actual timeout and memory-allocation probes pass; Windows Job Object and
  Linux `RLIMIT_AS` both refuse a 300 MiB allocation under the 256 MiB limit.
  Exactly 100 pages and the exact 1 MiB text threshold have boundary tests.
- An additional opt-in integration test passes using real `nomic-embed-text:v1.5`
  embeddings, in-memory Qdrant and `qwen2.5:7b` digest
  `845dbda0ea48ed749caafd9e6037047aa19acfcfd82e704d7ca97d631a0b697e`.
  A local two-page PDF answers from page 2, the citation resolves exactly,
  original bytes match, an unsupported question abstains, and deletion cleans
  the isolated document. Only model requests reach the cluster; no production
  document, database or vector writes occur.
- Chartroom: 49 unit tests, production build and lint pass. Both Chromium
  session tests pass, including PDF upload, page display, authenticated
  versioned original fetching and clearing the PDF link. Browser API responses
  in those tests are fixtures, not a production end-to-end acceptance.
- Harbormaster: 22 tests and TypeScript build pass, including PDF page metadata
  and unchanged non-PDF output/read-only policy.

Production acceptance and recovery (2026-11-03):

- Reviewed promotions were merged: Lighthouse `34dfe03`, Chartroom `ebe158f`
  and Harbormaster `8d724d7`, deployed through Fleet `7e134f1d1332`.
  The initialization hook applied `0002`; API/worker readiness returns 200
  with every check true, and the original PostgreSQL/originals PVCs remain Bound.

  | Component | Promoted image digest |
  | --- | --- |
  | Lighthouse | `sha256:e56901495ff289aabedeb975840e8c81bc8527280c3573d6770579020a02e726` |
  | Chartroom | `sha256:98401d07bb24a16c5508843f1980cd3b3791c06d07326bee4e4fcd6e71ba3317` |
  | Harbormaster | `sha256:fac6c9ee9d304df39c3b5f34afa2adff117267c92843b045886ac97c80a32be1` |

- Production backend checks passed: ingestion, page-2 answer/source, unchanged
  original bytes, deduplication, unsupported-question abstention, actionable
  no-text failure, existing text citations and fixture deletion. The isolated
  acceptance collection is empty; deleted document/job tombstones remain.
- The operator confirmed authenticated Chartroom upload/Ready, the correct
  beacon answer and page-2 reference, and opening the original at page 2.
  A separate parcel-stage question opened page 3.
- The first assistant-generated fixture had invalid direct PDF content streams;
  parser acceptance alone did not establish browser compatibility. The test
  factory now uses indirect content streams, with a regression assertion.
  The corrected three-page fixture rendered independently before browser use.
  The targeted PDF/backup suite passes 36 tests. Production does not generate
  PDFs; no production parser or original-storage change was needed.
- The real authenticated VS Code MCP connection found the corrected PDF and
  returned exact source chunks with `page_number: 2` for the beacon and
  `page_number: 3` for parcel stages. Source retrieval still works after the
  backup maintenance. This does not close deferred two-user OAuth isolation
  or silent-renewal checks from Milestone 7.
- A disposable PostgreSQL 17.6 instance rehearsed a fresh `0001` backup,
  restore into an empty database and upgrade to `0002`, preserving existing
  text chunks and null page metadata. PDF ingestion, exact originals,
  page-aware citations, a fresh `0002` backup/restore/reindex, real-model
  answering, terminal invalid-PDF failure and PDF deletion all passed.
  Only synthetic documents were used in this upgrade rehearsal.
- A fresh consistent production backup was then taken with API/worker writers
  stopped and Fleet/Lighthouse reconciliation temporarily suspended. Service
  and reconciliation were restored afterward. The archive is encrypted on the
  operator workstation with an authenticated encryption envelope and a
  current-user DPAPI-protected key; decrypt/read-back SHA-256 matched.
- That archive restored into another empty disposable PostgreSQL database
  and original directory. Reindex preserved document metadata, semantic
  version metadata, exact chunk IDs/text/lines/pages and every original hash:
  12 document records (including tombstones), 3 originals, 7 chunks and
  7 active vectors; all 3 ready documents rebuilt and jobs drained.
  The operator's PDF retained its page-2 source and exact original bytes.
  Recovery did not write to the live database, originals or vector collection.
  Disposable databases/vector collections and temporary pods were removed;
  no production PVC was replaced.

Historical release-gate exception: the fresh pre-upgrade production backup and
isolated PostgreSQL rehearsal were not established before the merged rollout.
The successful post-rollout backup and rehearsals prove current recovery and
upgrade behavior, not recovery of the exact production state immediately before
`0002`. The operator acknowledged that timing exception and approved Phase 9
closure on 2026-11-03. Independently recoverable off-host key/backup custody and
independent-host recovery remain Milestone 10 work; workstation DPAPI custody
alone does not satisfy them. Image-only rollback remains unsafe as documented
in the [production runbook](PRODUCTION-RUNBOOK.md#6-rollback).

Scope:

- Extract text per page and carry the page number through chunking into
  citations.
- Keep the original PDF in object storage so a citation can link back to it.
- Decide and document behavior for scanned PDFs with no text layer. Refusing
  them with a clear error is acceptable for this milestone; OCR is deferred.
- Bound extraction time and memory so a large or malformed PDF cannot exhaust
  the worker.

Acceptance criteria:

- A multi-page PDF ingests and answers cite the correct page.
- A PDF with no extractable text fails with an actionable error, not a stuck job.
- Chartroom renders page-level citations.
- Existing non-PDF citations are unchanged.

## 6. Milestone 10 - operational hardening

Owner: [Infrastructure](../Infrastructure) with Fleet and Lighthouse handoffs.
Estimated 1-2 weeks and partly continuous.

**Deployed monitoring verified (2026-11-03); backup automation deferred.**
Phase 9 is closed with the operator-approved timing exception above. Phase 10
keeps runtime orchestration in Infrastructure and telemetry implementation in
Lighthouse. The original full operational-hardening acceptance criteria remain
partially deferred; this is not a claim of unattended backups or disaster recovery.

Production verification: Infrastructure `d20a5177c4fd7c7e079f63875af810510538c672`
and Fleet `a62a93a2018009fe093171c2d5e728d342f3ab95` reconciled.
Lighthouse `3d4b319` runs image
`sha256:441dd1c47b6c2d32ac7a9cd68b7347086e452bdb0427d3b69c4ff1210c691496`.
API/worker readiness returned 200 with all checks true after enabling the
internal 9090 listener. Both Prometheus targets are UP without scrape errors;
all 12 MARINA rules evaluate healthily and are inactive. Application dashboard
panels are present, shared job gauges are API-only, and public port 8080
`/metrics` returns 404. Authenticated MCP still returns the exact PDF page-2
source. Original PVCs remain Bound and inactive roles remain at zero replicas.

Implemented and validated:

- Lighthouse provides an opt-in dedicated internal metrics listener, default
  disabled (`MARINA_METRICS_PORT=0`; production handoff uses `9090`), with bounded
  labels and no owner/document/query/credential contents. API-only queue gauges
  avoid duplicate counts; committed job outcomes, answer latency/outcomes and
  fresh dependency readiness are exported. No public API `/metrics` is added.
  Targeted telemetry/API/worker/grounding tests: 187 passed, eight skips;
  package build passed. The PDF/backup compatibility suite separately passes
  36 tests after telemetry changes.
- Infrastructure adds ClusterIP metrics Services/ServiceMonitor, queue age,
  terminal-job, dependency and missing-telemetry alerts, and application-level
  ingestion/answer-latency/failure panels in the existing MARINA dashboard.
  Real `promtool` 2.54.1 validates all 12 MARINA rules and their firing,
  hold-time, recovery, missing-target and strict-threshold fixtures.
- The explicitly gated Linux/WSL backup command and daily systemd templates
  support an approved mounted off-host destination, public-recipient age
  encryption, ciphertext/hash read-back and 30-day verified-pair retention.
  The scheduler never needs the private recovery identity. A local fallback
  mount is refused. Admission stops before worker drain, and restoration of
  service/reconciliation is attempted on failure or SIGTERM. Real age 1.2.1
  streaming/decryption and tampered-ciphertext rejection pass.
- The isolated restore command refuses production-shaped target names, restores
  metadata/originals, migrates, reindexes with actual dependencies and verifies
  exact original bytes, semantic version metadata, chunk IDs/lines/pages and
  vector counts. It passed against disposable PostgreSQL 17.6 with two
  synthetic text/PDF originals and four chunks/vectors.
- A pinned-image upgrade/rollback rehearsal confirmed previous-image readiness
  on `0001`, current-image readiness after `0002`, previous-image refusal on
  `0002` with only the database check failing, and successful previous-image
  recovery/reindex with identical text citations after restoring a pre-upgrade
  archive into fresh `0001` targets. This was an isolated application/schema
  rehearsal, not a live Helm rollback. Production images/data were unchanged.
- The [MARINA operations runbook](../Infrastructure/docs/runbooks/marina-operations.md)
  documents per-alert responses, daily backup/30-day retention, weekly decryption
  checks, monthly independent-host full restore, release-time upgrade/rollback
  checks and short Harbormaster/Trawler operational procedures.

Deferred work and completed deployment gates:

**Operator scope decision (2026-11-03):** retain encrypted backups only on the
Windows cluster host; no external alert destination. Off-host backup/key
custody, independent-host disaster recovery and notification delivery below
are deferred, not blockers to the reduced local operational scope. Host/disk
loss can still destroy both production data and recovery material; dashboards
and local logs require manual inspection. The prepared backup command currently
rejects local destinations: an explicit local mode, encryption/key-custody
configuration, schedule activation and restoration from an unattended local
backup remain outstanding. At 22:15 +07:00 the operator explicitly deferred
that local implementation and activation as unnecessary for now; do not start
it without a new request. Retain the existing encrypted backups. Do not bypass the off-host guard or claim local
automation is already active. The original broader goals remain below for
future work.

- Operator-approved physically independent backup host/mount and public
  recipient; independently recoverable identity, Vault snapshot/TLS/unseal
  custody and a real independent-host recovery report. Existing DPAPI archives
  are retained, not relabeled as host-loss protection.
- Install/enable the timer only after a successful manual off-host run and
  configure/test operator notification for failed/stale backups. Verify the
  first unattended backup and restore **that retained scheduled copy**.
- Completed on 2026-11-03: promote the telemetry-capable Lighthouse image through Shipyard/Fleet, then
  provision the runtime port and restart through the reviewed rollout path,
  reconcile metrics discovery/rules, and verify real scrape targets/dashboard.
  Configuration changes alone do not restart existing pods.
- Verify Alertmanager notification delivery to an approved receiver. Synthetic
  firing tests do not establish receiver delivery or live scrape health.
- Trawler's retained SQLite journal needs its own verified off-host custody;
  Lighthouse archives do not include it. Do not remove its retained backup.

Scope:

- Complete deferred section 0.3: off-host encrypted backup custody,
  independently recoverable application keys and Vault unseal shares, and a
  tested recovery on an independent host. Local DPAPI decryption is not enough.
- Schedule encrypted backups instead of running them by hand, with a stated
  retention period and a documented key custody model.
- Set a restore rehearsal cadence using the isolated drill procedure proven
  during production acceptance.
- Alert on queue age, failed jobs, PVC utilization, and dependency health.
- Add a dashboard that shows ingestion throughput, answer latency, and failure
  counts.
- Rehearse one upgrade and one rollback, and record what the rollback does not
  undo: database migrations and storage initialization are not reversed by a
  Helm rollback.
- Write a short per-service runbook for Harbormaster and Trawler once they carry
  production traffic.

Acceptance criteria:

- A backup runs unattended and a restore from it is verified.
- Each alert has a documented response and a tested firing condition.
- An upgrade and a rollback are both rehearsed and recorded.

## 7. Future work and deferred scope

### MARINA v1 closure (2026-11-03)

The operator accepted MARINA v1 as implemented and complete for the agreed
scope: controlled ingestion, PDF citations, source-grounded answers,
authenticated Chartroom, read-only MCP, two-real-user isolation and deployed
monitoring. The items below are follow-up housekeeping, additional validation
or explicitly deferred capabilities, not claims that those capabilities already
exist. Phase 10's original unattended-backup and independent-host recovery goals
remain deferred.

| Priority | Future work | Completion evidence / condition |
| --- | --- | --- |
| Closeout | Remove the two synthetic `isolation.md` documents through their owning `akadmin` and `nnvu.fit` Chartroom sessions | Both documents reach `deleted`; empty collections remain because collection deletion is unsupported. Fixture IDs are in the [OAuth runbook](../Infrastructure/docs/runbooks/marina-mcp-oauth.md#real-client-acceptance-follow-up-2026-11-03). |
| Closeout | Commit the remaining isolation harness and acceptance records | Reviewed changes published in their owning repositories; do not overwrite local MCP/account configuration. |
| Optional validation | Repeat post-expiry silent renewal as `nnvu.fit` and for any additional MCP clients | Real own/cross-user tool calls after expiry without manual login; existing renewal acceptance covers the tested `akadmin` VS Code connection only. |
| Optional validation | Compare a newly indexed automated Trawler extraction with the retained guide in Chartroom | Separate collection, exact commands/citations and complete multipart answers; synthetic/curated-guide acceptance does not prove arbitrary extracted-page quality. |
| Deferred operations | Implement and activate encrypted Windows-host backup scheduling with retention and restore cadence | New operator request, reviewed local key custody and a successful restore from an actual unattended retained backup. Existing off-host tooling rejects local destinations; do not bypass it. |
| Deferred resilience | Independent-host backup/key custody and disaster recovery, including Trawler SQLite and Vault recovery material | Operator-approved separate host/custody and a verified independent-host restoration; local DPAPI archives do not protect against Windows host/disk loss. |
| Intentionally unconfigured | External alert delivery | Only on request: approved destination and verified delivery; dashboards currently require manual inspection. |
| Usage-driven | Document-library search/filtering or an explicit internal-source ingestion contract | Choose a concrete user workflow first; preserve ownership and current private/LAN URL protections. |
| Maintenance | Review current dependency/base-image scan findings and prioritize remediation | Fresh scans and regression-tested fixes; historical report-only findings are not a clean security assessment or proof of current exploitability. |
| Optional compatibility | Additional MCP hosts, browser-session expiry/renewal and seven-day refresh expiry | Separate real-client evidence; do not infer acceptance from the existing VS Code five-minute renewal test. |

Keep the account `nnvu.fit` as the operator-requested non-admin account; fixture
cleanup does not authorize deleting that account or revoking its access.
Normal usage and feedback should guide further application work.

Backup scheduling/local-storage adaptation is deferred by the operator on
2026-11-03. No scheduled backup or external notification receiver is enabled.

Do not start these without a concrete, stated use case:

- Knowledge graphs and entity extraction.
- Autonomous or recursive crawling.
- Write-capable MCP tools.
- Learning agents and the `knowledge-learning` role.
- OCR for scanned documents.

### Recommended next application work

Prioritize Lighthouse answer completeness and Chartroom presentation before
new ingestion formats or autonomous behavior. The operator approved implementation
of this follow-up. Local implementation, validation, reviewed image promotion
and authenticated production browser acceptance are complete.

1. Establish a source-grounded regression set for complete multipart answers,
   requested step numbering, exact fenced commands and citations. Include the
   SDK installation/authentication guide, PDF page references, distractor lists
   and unsupported questions. Record the current production baseline first.
2. Fix omitted requested parts and ordering without inventing steps or repairing
   source text. Preserve abstention, owner isolation, exact citations and the
   existing whole-code-block behavior. Earlier grouped-selection candidates
   failed and must not be promoted merely because they appear more complete.
3. Gate promotion on measurable real-model results and authenticated Chartroom
   acceptance; retain negative grounding and non-PDF/PDF compatibility tests.
   If grounding and completeness cannot both be met, keep the current release
   and report the limitation rather than weakening verification.

Local implementation evidence:

- The unchanged adapter passed 5/9 synthetic real-model cases. Numbered requests
  failed, an unsupported requested operation was not rejected, and an exact-code
  question abstained.
- The new bounded selector retains complete section excerpts for numbered
  steps, anchors each operation to verbatim question text, validates order/count
  and IDs, and adds only server-owned numbering. Unnumbered selection explicitly
  covers all requested parts and permits quoting, never executing, source commands.
- All nine cases passed across three repeated real-model runs. Isolated
  upload/index/retrieval tests cover single-/two-chunk evidence, exact citation
  ownership and cleanup; the PDF integration still returns physical page 2 and
  the exact original bytes. The retained SDK guide returned both exact commands
  in the requested two steps.
- Full local regression run: 282 passed, 7 skipped, with real-model integration
  enabled. Source/wheel builds and `git diff --check` passed.
- This is not universal completeness or injection resistance. Selected sections
  can contain unrelated text; code spanning ingestion chunks is not reconstructed.
  These limits remain after acceptance.
- Published Lighthouse commit `2e03452` and Chartroom acceptance commit `06e8a80`;
  the operator merged their Fleet promotions at `aa6fa1a08bf059ff1c8f1a7779882c35dc3cbeba`.
  Lighthouse runs digest
  `sha256:3bc726c451d955b86f85a5da0ff1758c38640f0178f2bbf3ee15e6c0b2c77872`;
  Chartroom's deployment pins
  `sha256:9e4d19d7112773486c9912d3037cbd2a885bec2afdf5e888d3159ea47e18cb42`.
  Both HelmReleases are Ready; API and worker readiness checks all passed.
- On 2026-11-03, both live Chromium acceptance tests passed with manual operator
  Authentik login and no mocked responses: anonymous challenge, session mode,
  no bearer headers, correct write Origin, missing/foreign-Origin CSRF rejection,
  upload/indexing, multipart completeness, ordered numbered steps, rendered exact
  commands, exact source retrieval, unsupported-part abstention, deletion and
  workspace clearing. The synthetic document reached `deleted`; empty collection
  `25a6892a-dc99-4b10-9e6c-ecceffc364dc` remains because collection deletion is
  not supported. No browser session state was exported. Two-user isolation,
  real expiry/renewal and PDF viewer behavior were not retested by this suite.

After that, choose a concrete user workflow before expanding scope: document
library search/filtering or an explicit internal-source ingestion contract.
Private/LAN fetching is not currently supported by Trawler and must not be
enabled by bypassing its existing URL protections.

## 8. Sequencing

Milestones 7 and 8 are independent and could overlap, but a single developer
should finish Milestone 7 first because it is lower risk and validates the
read-only API surface that Trawler content will be consumed through.

| Period | Outcome |
| --- | --- |
| Week 0 | Gate 0 dependency recovery, authenticated smoke, backup custody, baseline references |
| Weeks 1-2 | Milestone 7 Harbormaster read-only MCP |
| Weeks 3-4 | Milestone 8 Trawler controlled URL ingestion |
| Weeks 5-6 | Milestone 9 PDF ingestion and page citations |
| Week 7 | Milestone 10 operational hardening |

## 9. Change rules carried forward

- Application source stays in its own repository; Infrastructure owns cluster
  runtime configuration only.
- Promote images through Shipyard and Fleet; never edit a running release by
  hand.
- Secrets stay imperative and local-only. No plaintext secret enters git.
- Static validation does not establish live readiness. Verify the running digest
  and `/ready` after every promotion.
