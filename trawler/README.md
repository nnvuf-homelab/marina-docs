# Trawler

MARINA's single-URL, operator-controlled HTTPS acquisition service. It fetches
and normalizes public text, retains provenance and durable outcomes, and hands
documents to Lighthouse through the existing upload API.

```bash
python -m venv .venv
python -m pip install -e ".[dev]"
pytest
export TRAWLER_STATE_PATH="$PWD/.local/acquisitions.sqlite3"
trawler-service
```

Local service binding defaults to `127.0.0.1:8080`. `docker compose up --build`
exposes `127.0.0.1:8090` and retains the acquisition journal in a named volume.
The container uses a non-root UID. It needs no model, database, object-store,
Kubernetes or Lighthouse credentials.

### Docker-only local setup (PowerShell)

```powershell
Set-Location D:\Workspace\homelab\trawler
docker compose up --build -d --wait fetch-worker
Invoke-RestMethod http://127.0.0.1:8090/ready
Start-Process http://127.0.0.1:8090/docs
```

The Swagger UI supports acquisition, outcome inspection and retries without
installing Python locally. For example:

```powershell
Invoke-RestMethod http://127.0.0.1:8090/v1/acquisitions `
  -Method Post -ContentType application/json `
  -Body '{"url":"https://example.com/"}'
```

To upload to an existing Lighthouse deployment, set the two operator environment
variables described below, then use the optional helper container:

```powershell
docker compose --profile tools run --rm ingest https://example.com/ `
  --trawler http://127.0.0.1:8080 `
  --collection <existing-collection-uuid> `
  --lighthouse https://marina.nnvuf.uk
```

The helper shares the fetch container's network namespace, so its local Trawler
port is **8080**, not host port 8090. Only this one-shot helper receives the
Lighthouse session cookie. The fetch service receives no credentials.
Do not share `docker compose config` output when operator credentials are set:
Compose expands the helper's environment values.

Use `docker compose logs fetch-worker` for diagnostics and
`docker compose down` to stop. Do not add `-v` unless you intend to delete the
retained acquisition history. This stack runs Trawler only; it does not start
Lighthouse, Authentik, Ollama or Qdrant.

## Fetch and ingest

Set `LIGHTHOUSE_SESSION_COOKIE` and `LIGHTHOUSE_ORIGIN` in the operator's local
environment (never in Trawler's service/container). Use a current user's browser
session and the existing Chartroom origin; the read-only MCP OAuth token cannot
authorize document uploads.

```bash
trawler-ingest https://example.com/ \
  --collection <existing-collection-uuid> \
  --lighthouse https://marina.nnvuf.uk
```

For Compose, add `--trawler http://127.0.0.1:8090`. Credentials are sent only to
the configured Lighthouse origin, with its CSRF Origin header. Service origins
must use HTTPS, except local loopback HTTP; redirects and proxy environment
inheritance are disabled. Lighthouse still enforces collection ownership.

The command uploads a `.txt` document through
`POST /v1/collections/{id}/documents`, not a second ingestion route. The document
contains requested/final URLs, redirect chain, UTC acquisition time, MIME type,
original-response SHA-256 and normalized text. Lighthouse's existing worker
creates line-based citations. The returned document ID is initially queued;
track its processing status through Lighthouse's existing document endpoint.
HTML extraction prefers nonempty article content, then main content, then the
remaining document. Navigation, headers/footers, sidebars, forms/buttons,
explicitly hidden elements and scripts/styles are excluded. Headings become
Markdown headings, list items retain boundaries, and preformatted code becomes
fenced code with indentation and blank lines intact. No JavaScript or browser
rendering runs; CSS-only visibility and client-rendered content cannot be inferred.
The extractor does not rewrite an article into a preferred language or invent
missing provisioning steps.

Markdown fences and blank lines are retained during normalization. Code lines
over Lighthouse's 4,000-character ingestion limit, excessive HTML nesting and
unclosed HTML code blocks fail explicitly rather than silently corrupting code.
The original response SHA-256 remains provenance for the fetched bytes, not a
hash of the normalized Markdown. Existing stored acquisitions are not rewritten;
acquire a new result to use the updated extractor.

If upload fails, its outcome is recorded and the acquisition ID is reported.
After correcting authentication/connectivity, reuse exactly the retained
document without issuing another public fetch:

```bash
trawler-ingest --acquisition <acquisition-uuid> \
  --collection <existing-collection-uuid> \
  --lighthouse https://marina.nnvuf.uk
```

Retrying the same retained content in the same collection uses Lighthouse's
existing hash deduplication. Reacquiring a URL creates a new timestamped source.

## Private operational API

| Route | Purpose |
| --- | --- |
| `POST /v1/targets/validate` | HTTPS URL + DNS snapshot validation; no fetch |
| `POST /v1/acquisitions` | Fetch one URL: `{"url":"https://example.com/"}` |
| `GET /v1/acquisitions?limit=20` | Recent acquisition and handoff outcomes, including failures |
| `GET /v1/acquisitions/{id}` | Retained normalized result and error |
| `POST /v1/acquisitions/{id}/retry` | New attempt for a failed acquisition; preserve prior failure |
| `POST /v1/acquisitions/{id}/handoff` | Operator reports `queued` + document ID, or `failed` + error |
| `GET /healthz`, `GET /ready` | Process health and writable state-store readiness |

Expected failures return explicit non-success HTTP statuses and retain an
acquisition ID/error. Interrupted fetches become failed/retryable after 60
seconds. Upload retries are separate from fetch retries. Handoff status is
operator-reported submission status, not proof that Lighthouse finished indexing.
No background crawling or automatic recursive retries run.

The API has **no public authentication layer**: keep it loopback-only or private
behind an explicit operator port-forward. Do not publish an ingress or expose it
to untrusted clients. Stored URLs/content may be sensitive; avoid URLs containing
credentials in query parameters and keep the state directory private.

## Enforced acquisition boundary

- HTTPS, standard port 443, no embedded credentials/fragments/control characters.
- Resolve all A/AAAA answers with a bounded subprocess; refuse the entire set
  if any address is private, loopback, link-local, multicast, reserved,
  IPv4-mapped/transition-range or a known metadata service.
- Connect directly to a validated numeric IP. TLS SNI and certificate hostname
  verification retain the original hostname; there is no second DNS lookup.
- If connection establishment fails, try the next address from the validated
  DNS snapshot, sharing the remaining total deadline across candidates. This
  supports IPv4-only pods receiving public IPv6 answers. TLS errors and
  response/content refusals do not trigger fallback.
- Revalidate each redirect, including relative redirects; maximum three hops.
- Maximum 1 MiB response/normalized document, 20-second total acquisition
  deadline including DNS and redirects, two concurrent acquisitions per process,
  and 8192-byte incoming request bodies. A socket shutdown enforces the deadline
  even on a slow-drip response.
- Accept only UTF-8 `text/plain`, `text/markdown`, `text/html`. Refuse compressed
  transfer content, oversized declared/streamed bodies and empty/invalid text.
- No ambient proxies, cookies, authorization headers or backing-store credentials
  go to public origins. No headless browser or service-account token is needed.

Production uses Infrastructure's retained `trawler-acquisitions` PVC in
`marina-fetch`: 2 GiB, `local-path`, `ReadWriteOnce`, mounted by Fleet at
`/var/lib/trawler`. The single replica uses `Recreate` and `fsGroup: 65532`.
The journal survived a verified pod replacement with both migrated and newly
created success/failure records. Storage is node-local, not replicated; the
capacity request is not a hard quota and automatic retention is not implemented.
See the [storage and recovery runbook](../Infrastructure/docs/runbooks/trawler-storage.md).
The workload disables service-account token mounting and receives no backing-store
credentials. Keep the acquisition API private; do not add a public ingress.

## Validation

`python -m pytest` runs the isolated boundary, operational API and handoff tests.
`tests/integration_lighthouse.py` is an explicit sibling-repository integration
check (not collected by the standalone suite). From this repo, using the
Lighthouse development environment:

```powershell
@'
import sys, pytest
sys.path[:0] = [r"..\lighthouse\src", r"..\lighthouse\tests", r"src"]
raise SystemExit(pytest.main(["-p", "conftest", "tests/integration_lighthouse.py", "-q"]))
'@ | ..\lighthouse\.venv\Scripts\python.exe -
```

This tests the real Lighthouse multipart route, migrations, worker, cited search,
answer contract and exact-content upload retry with repository model/vector/
object-store test doubles. Set `TRAWLER_PUBLIC_CAPTURE` to a JSON serialization
of `dataclasses.asdict(fetch_url(...))` to run it with captured real public
content. A verified Linux HTTPS fetch of `https://example.com/` and its subsequent
Lighthouse ingestion/citation check passed locally. Windows Python's CA store
rejected that site's certificate on this workstation; verification was not
disabled. The promoted production workload passed public acquisition, private-target
refusal, no-service-account-token and retained-storage checks. The official Foundry
guide also passed authenticated operator handoff, indexing, repeat-upload
deduplication and MARINA cited retrieval. A deliberately rejected session recorded
a failed handoff; retry with a valid session reused the acquisition and returned
the same Ready/version-1 document. Chartroom source inspection passed. Broader
answer completeness and numbered-step presentation are Lighthouse follow-ups;
the accepted answer path selects source quotations rather than writing summaries.