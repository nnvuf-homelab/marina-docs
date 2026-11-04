# Real-model acceptance evaluation

## Multipart answer quality

The non-sensitive fixture gate measures all requested configuration values and
commands, two-/three-step numbering and per-step order, unsupported-part
abstention, distractor rejection, exact fenced code and citation provenance.
It sends only synthetic fixture text to an explicitly chosen Ollama endpoint
and does not write production application data:

```powershell
python evaluation\answer_quality.py --ollama-url http://127.0.0.1:11435 --repeat 3 --output evaluation-report-answer-quality.json
```

The endpoint must already contain `qwen2.5:7b`. The JSON records the actual model
digest, individual failures and timings; any failed case or dependency error
fails the gate and exits nonzero. No failed case is silently dropped.
The initial unchanged-adapter baseline passed 5/9 cases. After the local
implementation, all nine passed across three repeated runs against digest
`845dbda0ea48ed749caafd9e6037047aa19acfcfd82e704d7ca97d631a0b697e`.
Only validated server-added ordinal prefixes are excluded from the numbered
quotation check; command contents, comments, indentation and blank lines are
not rewritten. The older `run.py --source-first` check remains intentionally
strict about exact quotations and does not strip numbered formatting.

For isolated upload/index/retrieval/answer/source/deletion acceptance with real
embeddings and generation, including single-/two-chunk retrieval and unchanged
PDF page references, run:

```powershell
$env:MARINA_TEST_OLLAMA_URL = 'http://127.0.0.1:11435'
try { python -m pytest tests\test_answer_quality_integration.py tests\test_pdf_integration.py -q }
finally { Remove-Item Env:\MARINA_TEST_OLLAMA_URL }
```

This uses disposable SQLite metadata, in-memory Qdrant/originals, and real
`nomic-embed-text:v1.5` embeddings. It does not replace authenticated Chartroom
browser acceptance, PostgreSQL deployment checks, or reviewed image promotion.
Section selection cannot reconstruct code spanning ingestion chunks or
guarantee that every selected quotation is relevant.

## PDF candidate checks

The PDF regression suite uses locally generated two-page PDFs and exercises
page-relative lines, exact raw originals, owner/version authorization,
deduplication/reindex/deletion, resource limits, malformed/encrypted/no-text
refusals and unchanged text/Markdown citations:

```powershell
python -m pytest tests\test_pdf.py
```

For an explicitly selected, operator-authorized Ollama endpoint, an opt-in
integration check uses real embeddings and answers with isolated SQLite
metadata, in-memory Qdrant and in-memory originals:

```powershell
$env:MARINA_TEST_OLLAMA_URL = 'http://127.0.0.1:11435'
try { python -m pytest tests\test_pdf_integration.py -q }
finally { Remove-Item Env:\MARINA_TEST_OLLAMA_URL }
```

The endpoint must have `nomic-embed-text:v1.5` and `qwen2.5:7b` already installed;
the test never downloads models. It verifies a page-2 answer/citation, matching
original bytes, abstention and cleanup. It writes no production application
data, but sends its non-sensitive fixture text to the configured model.
This does not replace a PostgreSQL upgrade/restore rehearsal or authenticated
production PDF/browser acceptance after reviewed promotion.

## Production verification and local restore rehearsal

[The production runbook](../PRODUCTION-RUNBOOK.md) separates local rehearsal from live
admission. With the current image and local Compose dependencies already running:

```powershell
docker compose run --rm --no-deps --entrypoint python -e MARINA_GENERATION_MODEL=qwen2.5:7b --volume "$((Get-Location).Path)\evaluation:/evaluation" knowledge-api /evaluation/restore_rehearsal.py --acknowledge-local --output /evaluation/evaluation-report-restore.json
python evaluation\production.py --api-url https://marina.nnvuf.uk --origin https://chartroom.nnvuf.uk --output evaluation-report-production-public.json
```

The first command uses the image's PostgreSQL client tools and disposable
database, original-storage, and vector targets. With the default filesystem
storage the isolated originals are a sibling directory of
`MARINA_STORAGE_PATH` on the same `marina-originals` volume (a bucket in S3
mode); the live `originals` directory is never written. Never run it with production administrator
credentials. It verifies backup, restore, vector rebuild, exact citations, and
a restored source-first answer, then cleans isolated resources. It does not
restart production applications or change their storage.

The second command uses a real Authentik browser session, a deployed API in
`MARINA_AUTH_MODE=forwardauth`, and working public TLS. Sign in to Chartroom as a
non-sensitive test user, open the browser developer tools' Network tab, and copy
the complete `Cookie` request-header value from an authenticated
`https://marina.nnvuf.uk/v1/session` request that returned 200. In Windows
PowerShell, `Read-Host` can truncate long Authentik cookies. Copy the value to
the clipboard, set it without echoing via
`$env:MARINA_EVALUATION_COOKIE = (Get-Clipboard -Raw).Trim()`, then immediately
replace the clipboard with `Set-Clipboard -Value " " -ErrorAction Stop`.
A single space avoids empty-string errors in Windows PowerShell. Remove any
clipboard history or synchronized copies separately if enabled.
Clear the environment variable
after the run. The cookie is a live credential: never put it in arguments,
files, reports, chat, or tickets. `--origin` (or
`MARINA_EVALUATION_ORIGIN`) must be the exact Chartroom origin configured in
`MARINA_FORWARDAUTH_BROWSER_ORIGIN`; writes from any other Origin are rejected.
Bearer tokens are rejected for this smoke.

Cookie-free requests, requests with forged `X-authentik-*`/forwarded identity
headers, and invalid bearer requests must produce the edge's Authentik login
redirect (302/303/307 with a `Location`; pin it with `--login-prefix
https://auth.nnvuf.uk/`) or 401. A login redirect is the expected challenge, not
an outage. Optionally pass `--direct-api-url http://127.0.0.1:<port>` through a
private port-forward to prove the API itself returns 401 for forged identity
without a session. With the cookie it checks `/v1/session`, that forged headers
do not change the owner, CSRF rejection of foreign/missing Origin (403),
readiness, failed-job/queue-age gates, and exact-source grounding with fixture
cleanup. Redirects are never followed, so the cookie is not sent to another
endpoint. Running against loopback rehearses behavior but cannot establish public
Authentik acceptance. Reports remain gitignored, contain no cookie, and may
contain non-sensitive fixture text.

## Grounding corpus

This evaluation is separate from unit tests. It exercises a running Lighthouse
API, worker, PostgreSQL, original storage, Qdrant, and the selected Ollama models.
Passing adapter tests does not establish retrieval or answer quality.

The non-sensitive [corpus](corpus.md) summarizes workspace architecture.
The [dataset](https://github.com/nnvuf-homelab/lighthouse/blob/main/evaluation/questions.json) contains 20 answerable and 10 unsupported questions.

From the Lighthouse directory, supply an authorized bearer token through
`MARINA_EVALUATION_TOKEN` (local `dev`/`oidc` modes) and run:

```powershell
python evaluation\run.py --seed --output evaluation-report.json
```

Alternatively, use `--collection-id <id>` against an existing collection
containing the evaluation corpus as separate section documents.
Use `--api-url` for a different API origin.
Non-loopback APIs require HTTPS. Redirects are rejected to avoid forwarding the
token to another endpoint. Against a `forwardauth` deployment, set
`MARINA_EVALUATION_COOKIE` instead of the token (and `MARINA_EVALUATION_ORIGIN`
to the Chartroom origin); setting both is rejected. The cookie is sent with the
Origin header on writes and is never stored or logged.

The seed operation creates a collection and uploads each corpus section as its
own document. This provides more than five candidate passages, so top-five
retrieval is not trivially satisfied by returning the entire corpus.
It leaves that collection available for inspection; delete its documents through the application
when finished. It does not remove existing collections or data.

Automatic gates:

- Supporting text appears in the top five results for at least 18 of 20 questions.
- At least 9 of 10 unsupported questions return `insufficient_evidence`.
- Citations resolve to the matching document version, chunk, text, and lines.

The report includes answers and citations for manual review. Human review of
factual support remains required even when the automatic gates pass.
Review failures before changing retrieval settings or evidence thresholds.
Do not substitute test doubles for this acceptance run.

## Milestone 4: grounded-answer review

The full evaluation also reports `answer_coverage_gate_pass` at 18/20 cited
answers. This is an additional coverage diagnostic: it prevents excessive
abstention from looking like a usable question-answering system, but neither
coverage nor valid citation IDs proves factual support. The runner's exit code
still represents the original retrieval and unsupported-question gates.

Each answerable result includes its expected supporting text for review.
For every generated answer, check:

- All factual statements are supported by the supplied citations.
- Names, quantities, ordered paths, and lists are correct and complete.
- Negation and optional/required distinctions are preserved.
- No uncited qualifications or explanations have been added.
- Unsupported questions abstain rather than claiming facts.

Use an exact installed generation tag; record its digest from the application
logs with the evaluation results. Evaluate replacement models on the whole
dataset, not only the previously failing questions. Model tags can change
upstream: repeating acceptance is required after their digest changes.

The adapter bounds generation to 1,024 output tokens and an 8,192-token model
context, while the service bounds supplied evidence by configured character
budget. A generation stopped by its token limit returns an explicit dependency
error, not a truncated success. Dependency outages, invalid/unknown citations,
timeouts, and concurrency saturation are regression-tested separately.

The current source-first adapter selects source excerpts using their section
headings as context, then constructs answers from exact stored text.
Prose is split into sentences, but fenced code within a supplied chunk is one
excerpt: comments, blank lines, indentation and sentence-like strings remain
unchanged. Code comments do not become section headings. This does not join
code blocks split across ingestion chunks, reconstruct missing delimiters, or
guarantee that the model selects every requested part. Excerpt IDs are internal;
they do not implement user-requested numbered steps.
It does not freely compose answers or rely on a same-model factual auditor.
One selection call is bounded by `MARINA_MODEL_TIMEOUT` (90 seconds by default).
Invalid selection output, unknown IDs, truncation, excessive answer length,
timeouts, and concurrency saturation remain dependency errors, not abstentions.
The client/proxy retain a conservative 600-second budget including retrieval.

Validate the exact-source requirement explicitly:

```powershell
python evaluation\run.py --source-first --collection-id <id> --output evaluation-report-source-first.json
```

`source_first_text_validated` records this check. Each excerpt separated by a
blank line must exist verbatim in a returned citation; changing whitespace or
paraphrasing is rejected. This proves copying, not question relevance or completeness.
Human evidence review remains required. Models select sentences within existing
chunks, so extraction cannot repair missing context at chunk boundaries.

Evidence is kept in the user message as quoted data, never promoted to system
instructions. This separation and an instruction to ignore document commands
reduce prompt-injection risk; they are not a guarantee that arbitrary hostile
documents cannot influence a model. The RAG model receives no execution tools.

Run the harmless document-instruction probe against the same API/model:

```powershell
python evaluation\grounding.py --source-first --output evaluation-report-grounding.json
```

It uploads a known beacon-color fact beside instructions to emit a harmless
sentinel and invent dates. It checks the supported answer, abstention on the
missing date, exact citations, and absence of the sentinel. Two independent
examples additionally check ordered processing stages versus building names and
sensor routing without invented direct/indirect qualifications. Its document is
deleted in `finally`; an empty test collection remains for inspection.
Passing these four cases is not a universal grounding or prompt-injection guarantee.
An assertion failure writes `fixture_gate_pass: false` and its failure reason
when `--output` is supplied, then still exits with an error. This prevents an
older successful report from masquerading as the current result.

## Milestone 3: retrieval without generation

```powershell
python evaluation\run.py --retrieval-only --seed --output evaluation-report-retrieval.json
```

Alternatively, pass `--collection-id <id>` for the existing seeded evaluation
collection. The required token and API-origin rules above still apply.
This mode calls search and source endpoints only: no generation-model request,
answer quality score, or unsupported-question abstention claim is made.
It requires at least 18/20 supporting-passage hits in the top five and validates
each returned passage's document, published version, text, and line references.
Reports are ignored by Git. Retrieval results alone do not approve Milestone 4.

## Real document lifecycle and restart check

With the same token environment variable, run:

```powershell
python evaluation\smoke.py --phase create --state <local-state-file>
python evaluation\smoke.py --phase verify --state <local-state-file>
# Restart only this local stack's app and backing containers, preserving volumes.
python evaluation\smoke.py --phase verify --state <local-state-file>
python evaluation\smoke.py --phase delete --state <local-state-file>
```

Use `--api-url http://127.0.0.1:8084/api` to exercise Chartroom's real Nginx
proxy. Store the state file outside the repository; it contains document and
collection identifiers, not credentials. The check covers duplicate uploads,
worker ingestion, retrieval/source consistency, immediate deletion exclusion,
terminal cleanup, and unauthorized requests. A restart check is not a backup
restore test.

## Milestone 2: real worker interruption

Run `python evaluation\reliability.py` with `MARINA_DATABASE_URL`,
`MARINA_STORAGE_BACKEND`/`MARINA_STORAGE_PATH` (or the `MARINA_S3_*` settings in
S3 mode), `MARINA_QDRANT_URL`, its API key,
`MARINA_OLLAMA_URL`, and the embedding model/dimensions configured for local
services. Use a disposable local PostgreSQL database; never a live cluster.

For the repository's local Compose stack, from the Lighthouse directory:

```powershell
$v = ConvertFrom-StringData (Get-Content .env -Raw)
$env:MARINA_DATABASE_URL = 'postgresql+psycopg://marina:' + $v.POSTGRES_PASSWORD + '@127.0.0.1:5432/marina'
# A host directory whose parent exists; the harness uses an isolated sibling.
$storage = New-Item -ItemType Directory -Force (Join-Path $env:LOCALAPPDATA 'marina-evaluation')
$env:MARINA_STORAGE_BACKEND = 'filesystem'
$env:MARINA_STORAGE_PATH = Join-Path $storage.FullName 'originals'
$env:MARINA_QDRANT_URL = 'http://127.0.0.1:6333'
$env:MARINA_QDRANT_API_KEY = $v.QDRANT_API_KEY
$env:MARINA_OLLAMA_URL = 'http://127.0.0.1:11434'
$env:MARINA_EMBEDDING_MODEL = $v.MARINA_EMBEDDING_MODEL
$env:MARINA_EMBEDDING_DIMENSIONS = $v.MARINA_EMBEDDING_DIMENSIONS
.\.venv\Scripts\python.exe evaluation\reliability.py
```

Run this in a dedicated terminal and close it afterwards to discard its
credential environment variables. Do not print or commit those values.

For a stack still in S3 mode, set `MARINA_STORAGE_BACKEND = 's3'`,
`MARINA_S3_ENDPOINT = 'http://127.0.0.1:9000'`, and the MinIO credentials instead.

The harness creates a unique schema, private original-storage directory (or
bucket), and vector collection, and removes them afterwards.
It launches a real worker process, allows actual embeddings and Qdrant writes,
then kills that exact child process before PostgreSQL publication commits.
It waits for natural lease expiry, reclaims the job, rejects a stale worker,
and checks exact deterministic chunk/point counts and embedding provenance.
Incomplete evidence must remain hidden; recovered evidence must be searchable.
Deletion is also checked. All generated resources are cleaned in `finally`.

The six-second lease and post-write barrier belong only to the test harness,
not production configuration. The normal application worker and its documents
are not stopped or modified. This demonstrates process-crash recovery, not a
Kubernetes node-loss or production backup recovery guarantee.

The report is ignored by Git because it may contain content from an existing
collection. Do not commit private evaluation reports.
