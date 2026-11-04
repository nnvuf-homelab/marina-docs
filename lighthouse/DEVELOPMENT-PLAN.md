# MARINA Development Plan

## Implementation status

Architecture decisions are approved. The app-side implementation now includes
durable document APIs, leased ingestion/deletion jobs, PostgreSQL migrations,
private S3 originals, Qdrant retrieval, Ollama answers, OIDC token validation,
middleware-only Authentik session authentication (`forwardauth`: the browser
cookie is re-verified against the existing `homelab-forward-auth` outpost per
request, owner = fixed namespace + Authentik UID, exact-Origin CSRF checks, no
OAuth callback or pasted production tokens), and Chartroom's document/question
workflow. Earlier `oidc` owners are not auto-mapped (see the runbook).

See [Lighthouse startup and configuration](README.md),
[Chartroom integration](../chartroom/README.md), and the
[real-model acceptance evaluation](evaluation/README.md).

Local Docker validation now confirms startup of PostgreSQL, MinIO, Qdrant,
Ollama, the API/RAG/worker roles, and Chartroom's authenticated Nginx proxy.
Real PostgreSQL concurrency tests pass. Original documents and metadata have
been backed up and restored into separate validation resources.
Full-stack restarts preserve data, and deletion removes originals and vectors.

Earlier, superseded free-form trials: the initial 3B real-model evaluation retrieved evidence for 20/20 answerable
questions, returned cited answers for 19/20, and rejected all 10 unsupported
questions. The subsequent 7B evaluation achieved 20/20 retrieval hits, 20/20
cited answers, and 10/10 unsupported rejections. Automatic gates pass, but
assistant evidence review still found an incorrect Flux group list and an
unsupported qualification about external Ollama access. A subsequent bounded
draft/audit/correction flow withholds both bad answers, but returns only 16/20
answerable responses, below the unchanged 18/20 coverage diagnostic. Generation
quality in those trials was not approved; citation validity and answer coverage
are not factual acceptance.

The newer source-first implementation replaces those free-form trials and
passes local measured gates: 19/20 source-backed answers, 10/10 unsupported
rejections, and all four independent probes. On 2026-10-09, the user confirmed
their testing was satisfactory and approved committing the implementation;
see the current Milestone 4 section before interpreting the historical results.

This is not a completed production deployment. Real-model acceptance is tracked
separately; production Authentik/public reachability, image security scanning,
and production backup procedures still require verification.

Production promotion is blocked until Infrastructure provisions dependencies
and Fleet wires runtime configuration, named secrets, and the migration gate.
Staged Fleet runtime references and suspension now guard that handoff, and
Shipyard preserves release overrides during regeneration. Do not resume the
production releases until prerequisites pass; their initialization hook must
complete before rollout, followed by live acceptance checks.

## Goal and scope

Deliver a single-user, authenticated first release that lets a user upload
homelab documentation, track ingestion, and ask questions answered with
verifiable source citations.

Start with Markdown and plain text. Defer PDFs, web crawling, knowledge graphs,
autonomous research, and learning workflows until the core workflow is reliable.

At the start of this plan, Lighthouse exposed only health and about endpoints,
Compose ran four role scaffolds without backing stores, and Chartroom used
representative local data. The implementation status above records progress
beyond that baseline.

## 1. Architecture decisions

These are proposed dependencies, not assertions that every service is already
provisioned.

| Concern | MVP choice | Reason |
| --- | --- | --- |
| Metadata and ingestion jobs | PostgreSQL | Transactions, migrations, and durable job state |
| Original documents | Private filesystem storage on a dedicated volume (S3-compatible storage remains an opt-in mode) | Separates originals from metadata and vectors without another service; node-local volumes need off-node backups |
| Retrieval index | Qdrant | Matches the existing homelab vector-service direction |
| Embeddings and answer generation | Ollama | Uses existing local AI infrastructure |
| Background execution | PostgreSQL-backed job queue | Avoids introducing Redis solely for the first workflow |
| User interface | Chartroom | Builds on the existing application interface |
| Authentication | Authentik-backed access with API-side authorization | Protects APIs as well as the UI |

Preserve Lighthouse's role boundaries:

- `knowledge-api`: collections, documents, ingestion status, and source access.
- `knowledge-worker`: ingestion, indexing, and deletion jobs.
- `agentic-rag`: retrieval and answer generation, without an autonomous agent loop.
- `knowledge-learning`: retain the scaffold but exclude it from the MVP's required
  execution path.

Before implementation, record decisions for storage, authentication, model
selection, API routing, and repository ownership. Select and pin embedding and
generation models after checking available hardware and a representative dataset.

## 2. Implementation milestones

### Milestone 1: Persist documents

**Status: complete - implemented and locally accepted on 2026-10-08.**
Production deployment is not part of this acceptance.

Delivered in Lighthouse:

- Database migrations.
- Collections, documents, document versions, and ingestion jobs.
- Markdown and plain-text upload into private original storage.
- File-size limits, encoding validation, and content hashes.
- Document listing, ingestion status, and deletion requests.
- Structured errors and request IDs.
- Local backing services and configuration in Compose.

Implemented API surface:

| Endpoint | Purpose |
| --- | --- |
| `POST /v1/collections/{id}/documents` | Upload a document |
| `GET /v1/collections/{id}/documents` | List documents |
| `GET /v1/documents/{id}` | Inspect metadata and ingestion state |
| `POST /v1/documents/{id}/retry` | Retry failed ingestion |
| `DELETE /v1/documents/{id}` | Request durable cleanup |

Acceptance evidence:

| Criterion | Result | Evidence |
| --- | --- | --- |
| Originals and metadata survive restarts | Passed locally | Real Docker app/backing-service restart preserved retrieval and exact source passages; separate PostgreSQL and object-store restore verified nine originals and matching metadata |
| Invalid uploads produce actionable errors | Passed | Unsupported extension, invalid UTF-8, empty/whitespace/NUL text, oversized file, and excessive line length return explicit errors with matching request IDs |
| Duplicate policy is explicit and tested | Passed | Same collection/content hash returns the existing document; deleting duplicates return conflict; upload is allowed again after cleanup |
| Collection ownership is enforced | Passed | Another owner's resources return 404 at upload, listing, status, retry, delete, source, search, and question boundaries |

Closeout validation reran the applicable tests against both SQLite and real
PostgreSQL: **20 passed, 0 skipped**. Test selection:

```powershell
.\.venv\Scripts\python.exe -m pytest -q tests\test_workflow.py -k "invalid_upload or collection_owner_on_every_boundary or full_workflow_duplicate_source_and_deletion or upload_outage_retains_cleanup_address"
```

The PostgreSQL run requires `MARINA_TEST_DATABASE_URL` pointing to a disposable
local database, as documented in [the validation instructions](README.md).
The real lifecycle/restart procedure is in [evaluation](evaluation/README.md).

Separate acceptance is tracked below: worker-interruption acceptance is recorded
under Milestone 2; answer-quality approval belongs to Milestone 4;
Infrastructure/Fleet configuration and production Authentik/public reachability
belong to Milestone 6. None is implied complete by this Milestone 1 closeout.

### Milestone 2: Make ingestion reliable

**Status: complete - implemented and locally accepted on 2026-10-08.**
This covers process-crash recovery, not Kubernetes node-loss recovery.

Delivered worker pipeline:

1. Claim a durable job.
2. Read and validate the original.
3. Extract text while retaining headings and line references.
4. Create deterministic chunks.
5. Generate embeddings.
6. Index chunks in Qdrant.
7. Publish the completed document version.

Use explicit states such as `queued`, `processing`, `ready`, `failed`, and
`deleting`.

Include:

- Job leases and recovery after worker crashes.
- Bounded retries with visible failure reasons.
- Idempotent indexing.
- Stored embedding model/version and vector dimensions.
- Deletion cleanup across originals, metadata, and vectors.
- Version-aware retrieval so partially indexed replacements are not exposed.

Acceptance criteria:

- Restarting a worker during ingestion neither loses the job nor duplicates chunks.
- Failed jobs can be retried safely.
- Deleted documents immediately disappear from retrieval.
- Cleanup remains durably tracked until complete.

Acceptance evidence:

- A real child worker was killed after actual Ollama embeddings and Qdrant
  writes, before PostgreSQL publication committed. Partial vectors were hidden.
- Natural lease expiry allowed a replacement worker to reclaim the same job
  with a new token. A stale worker could not publish or access backing services.
- Recovery produced exactly the expected deterministic PostgreSQL chunk IDs and
  Qdrant point count, with stored embedding digest and dimensions.
- Retry-budget exhaustion and manual retry, partial-index retries, concurrent
  claims, deletion during ingestion, and failed cleanup retries pass on real
  PostgreSQL as well as applicable SQLite cases.
- Recovered documents disappear from retrieval immediately on deletion;
  durable worker cleanup removes the actual vector points.

Validation: all workflow tests **63 passed, 1 skipped**. The skip is the
PostgreSQL-specific concurrent-claim test's SQLite variant; its PostgreSQL
variant passed. The [real interruption harness](https://github.com/nnvuf-homelab/lighthouse/blob/main/evaluation/reliability.py)
passed twice and removed its unique database schema, bucket, vector collection,
temporary marker, and child process without stopping the running app worker.
See [reproduction instructions](evaluation/README.md).

The MVP uses immutable uploads with published version 1. Editing/replacing an
existing document is not an implemented workflow; incomplete jobs remain hidden
behind authoritative document status and active-version checks.

### Milestone 3: Search before generating answers

**Status: complete - implemented and locally accepted on 2026-10-08.**
Retrieval acceptance is independent of generation quality.

Delivered:

- Collection-scoped semantic search.
- Results containing document ID, version, chunk ID, title, excerpt, and source
  location.
- A source endpoint that opens the cited passage.
- Authorization filtering before evidence reaches the model.

Create an evaluation set from homelab documentation with approximately 20
answerable questions and 10 unsupported questions. Maintain expected supporting
passages for the answerable questions.

Acceptance criteria:

- The expected supporting passage appears in the top five results for at least
  18 of the 20 answerable questions.
- Search results never cross a collection-access boundary.
- Every result resolves to the correct stored source version.

This milestone isolates retrieval quality from model behavior.

Acceptance evidence:

- The real retrieval-only evaluation through Chartroom's Nginx proxy achieved
  **20/20 supporting-passage hits in the top five**, above the required 18/20.
  Each result resolved to its exact stored document, published version, title,
  passage text, and one-based line references. No question-generation endpoint
  was called.
- Ownership checks and authoritative collection filtering reject foreign
  resources even if the vector backend supplies another collection's IDs.
- Ranked results are deduplicated. Unpublished versions cannot be retrieved
  through the source endpoint. Embedding digest and dimension mismatches fail
  explicitly rather than returning misleading results.
- Workflow and evaluator regressions: **80 passed, 1 skipped** across SQLite and
  real PostgreSQL. The skip is the PostgreSQL-only concurrency test's SQLite
  variant, not a skipped PostgreSQL check.

Reproduce with `evaluation/run.py --retrieval-only` as described in
[evaluation instructions](evaluation/README.md). The local report is stored in
ignored `evaluation-report-retrieval.json`. Milestone 4's known generation
quality limitations and Milestone 6's production rollout gates remain open.

### Milestone 4: Add grounded question-answering

**Status: complete - source-first implementation locally accepted on 2026-10-09.**

The user confirmed satisfactory testing and approved committing this implementation.
This accepts the local milestone, not production deployment.

On 2026-10-09, the approved source-first UX replaces free-form answers and the
same-model auditor. The model selects numbered complete source sentences using
section headings as context; the server returns exact stored excerpts and maps
their IDs back to authorized chunk citations. It cannot invent names or append
qualifications to answer text. Chartroom labels these **What your sources say**,
not a generated summary. Public API shapes remain unchanged.

Current built-image validation with `qwen2.5:7b`:

- **20/20** supporting passages retrieved.
- **19/20** answerable questions answered (required 18/20).
- **10/10** unsupported questions rejected (required 9/10).
- **4/4** independent grounding probes passed; fixture cleanup verified.
- Every excerpt is exact text from its cited source; assistant review of all
  returned answers found relevant support and complete requested paths/lists.
- Flux groups abstained rather than returning a wrong list. No corpus-specific
  answer rules or relaxed thresholds were introduced.
- **140 tests passed**, one expected SQLite-only concurrency skip; Chartroom
  **25 tests passed**, lint and frontend image build passed. Backend image build passed.
- Corpus request latency: median **27.9 seconds**, maximum **30.6 seconds**
  on the local CPU-only Ollama runtime.
- Fresh Chartroom Nginx proxy validated the full ingress-path answer and exact
  citations; all four probes also passed through the authenticated proxy with
  `--source-first`.

Ignored `evaluation-report-source-first-final.json` records the full run and
model digest. Earlier paragraph-selection trials are retained separately and
are not accepted results. Source selection now precedes status classification
in structured output, and uses simple request-local IDs rather than long chunk
identifiers; these changes improved model coverage.
Ignored `evaluation-report-source-first-proxy-grounding.json` records the
additional proxy probes.

Copying proves provenance, not truth or relevance. Sentence splitting is
conservative, not a universal linguistic parser, and cannot recover missing
context across chunks. User testing and approval close this local milestone;
production handoffs remain gated. Normal containers have not been
replaced. The results below describe the superseded free-form approach.

The current work adds explicit truncation rejection, a bounded model context,
preservation instructions for complete names/lists/paths/negations, and tests
that keep document commands in quoted user evidence rather than system messages.
Evaluation reports answer coverage separately from retrieval/abstention and
includes expected supporting passages for evidence review. A harmless adversarial
document fixture checks supported answers and abstention beside embedded commands,
plus independent entity/list and routing examples, then deletes its uploaded document.

The previous answer adapter implemented a bounded draft -> audit -> one correction ->
final audit flow. Audits compare each factual clause with only the cited passages,
analyzing evidence before issuing a verdict. Rejected corrections abstain with
no citations; malformed output, unknown citations, truncation, and model outages
remain explicit dependency errors. Drafts, evidence, and review reasons never
become system instructions or ordinary log contents. There are at most four
generation calls; Chartroom's proxy and the evaluator allow 600 seconds for the
default per-call limits plus retrieval/embedding work. Public API shapes are unchanged.

Local validation on 2026-10-08:

| Check | Result |
| --- | --- |
| Adapter, workflow, and evaluation regressions, SQLite and real PostgreSQL | 128 passed, 1 SQLite-only concurrency skip |
| Chartroom proxy contract regressions | 8 passed |
| Python compilation and dependency consistency | Passed |
| `qwen2.5:7b` supporting-passage retrieval | 20/20 |
| Strictly audited `qwen2.5:7b` answered responses with exact resolvable citations | 16/20; coverage gate fails (required 18/20) |
| Unsupported questions rejected | 10/10 |
| Harmless document-instruction and independent fixtures | Color and missing date pass; repeated runs falsely reject ordered stages and/or the supported sensor route; document cleanup completed |
| Assistant review of the 16 returned factual answers | Supported by their cited passages; human acceptance still pending |
| Known incorrect Flux and Mac Ollama answers | Both withheld, not corrected successfully |
| Additional answerable corpus questions rejected | Ingress path and deployment order; no coverage threshold was relaxed |
| Fresh backend and Chartroom image builds | Passed without bypassing dependency integrity checks |

The candidate digest is
`845dbda0ea48ed749caafd9e6037047aa19acfcfd82e704d7ca97d631a0b697e`.
Full quality tests used a temporary current-source API with real local dependencies.
Separate fresh-image readiness and authenticated Nginx answer/source checks pass.
The final corpus report is ignored `evaluation-report-verified-strict.json`;
`evaluation-report-verified-grounding.json` explicitly records a failed independent
probe rather than retaining a stale success;
earlier `evaluation-report-milestone4.json` records unverified generation, and
`evaluation-report-verified.json` records the superseded less-strict audit.
The existing local stack retains its prior 3B configuration.

Acceptance rerun on 2026-10-08 (19:42-20:09 local time for the corpus):
the built strict-verifier image, using the same 7B digest and real dependencies,
again returned **20/20 retrieval hits, 16/20 cited answers, and 10/10 unsupported
rejections**. Every citation resolved exactly; assistant review found all 16
returned answers supported. The same four answerable cases abstained: ingress
path, Flux groups, Mac Ollama access, and deployment order. The independent
probe failed on the supported sensor route and cleaned its document. Regression
checks again passed: **128 backend/evaluation tests**, one expected SQLite-only
skip, and **8 proxy tests**. Human approval and the 18/20 coverage diagnostic
remain unmet. No model, prompt, or acceptance threshold changed for this rerun.

Fresh ignored reports: `evaluation-report-milestone4-ac-rerun.json` (including
run timestamps and model digest) and
`evaluation-report-milestone4-ac-rerun-grounding.json` (explicit failed probe).
The isolated acceptance container was stopped after validation; the normal
stack was not replaced.

Larger-model comparison (started 2026-10-08; unchanged prompts, context, and
90-second per-call timeout): `qwen2.5:14b`, digest
`7cdf5a0187d5c58cc5d369b255592f7841d1c4696d45a8c8a9489440385b22f6`,
does **not** resolve acceptance. Both 7B and 14B passed 5/6 independent verifier
calibration cases, wrongly accepting an unsupported direct-access qualification.
The 14B corpus run completed 18 answerable cases (17 answered, Flux groups
abstained), then returned HTTP 503 on Mac Ollama access. A focused reproduction
confirmed the correction exceeded the unchanged 90-second model timeout after
the auditor wrongly rejected the supported route. Remaining corpus cases were
not evaluated; no full-set success metrics are claimed. The independent
grounding probe also returned HTTP 503 and cleaned its fixture.

Completed 14B questions had a median latency of 113.61 seconds and a maximum
of 299.64 seconds on this CPU runtime. Observed Ollama container memory was
approximately 14 GiB of the available 31 GiB. This candidate improved the ingress
answer but did not improve the unsupported-qualification calibration result.
The explicit failed/partial comparison, model inventory, calibration judgments,
and timings are in ignored `evaluation-report-model-comparison.json`.
Normal model configuration remains unchanged; increasing timeout alone would
not fix the demonstrated factual judgment failures.

Different-family comparison on 2026-10-09: Meta `llama3.1:8b`, digest
`46e0c10c039e019119339687c3c1757cc81b9da49709a3b3924863ba87ca666e`,
completed the full built-image evaluation with unchanged prompts and limits:
**20/20 retrieval hits, 19/20 cited answers, 10/10 unsupported rejections**,
and all four independent grounding cases passed. Verifier calibration remained
**5/6**, accepting the same unsupported direct-access qualification. Assistant
review of every returned factual answer failed: Flux groups were answered as
`Flux, Helm, Kustomizations`, and the ingress path omitted ClusterIP Service and
ready workload. The same-model auditor accepted both defects. The control-plane
location question unnecessarily abstained.

Corpus-question median latency was 44.87 seconds, maximum 95.81 seconds.
Ollama reported approximately 6.01 GiB loaded for Llama and zero GPU allocation;
total Ollama container memory also included other loaded models and is not a
Llama-only footprint. Results, exact citations, calibration verdicts, timings,
model inventory, and explicit failed assistant review are in ignored
`evaluation-report-llama-comparison.json`. Passing coverage and these four
independent probes did not establish factual acceptance. The temporary candidate
was stopped; probe documents were cleaned; normal runtime configuration remains
unchanged. This model family is not approved for Milestone 4.

The 7B candidate fixed the earlier incomplete ingress path and unnecessary
optional-worker abstention, but still answered the Flux-group question with
repository names rather than `platform`, `applications`, and `observability`.
The less-strict auditor withheld the incorrect Flux answer but still accepted
the unsupported Mac qualification. The final per-clause auditor withholds both,
at the cost of excessive abstention. An independent sensor-routing example
reproduces this false rejection, so the limitation is not confined to the corpus.
Repeated independent runs also rejected the directly supported processing-stage
list: the same-model audit is not reliably calibrated.
The available 3B auditor also failed a focused comparison; no automatic verifier
model switch or dataset-specific answer rules were added.
At the time of these superseded trials, human acceptance remained pending;
assistant review is not human approval.
A model must pass factual evidence review before this milestone can be accepted.
Valid citation IDs alone are insufficient. Further prompt-only experiments are
not an acceptance substitute: a better-calibrated model/auditor must improve
coverage without reintroducing unsupported claims.

Implement in `agentic-rag`:

- Retrieval of authorized evidence.
- Answer generation grounded in that evidence.
- Structured answers with citations and an explicit insufficient-evidence outcome.
- Citation identifier validation against retrieved passages.
- Model timeouts, concurrency limits, and context-size limits.
- Handling of uploaded text as untrusted evidence, not instructions.
- Model version, retrieval parameters, and timing records, without logging private
  document contents by default.

Acceptance criteria:

- Every returned citation resolves to evidence actually supplied to the model.
- Human review confirms factual answers are supported by cited passages.
- At least 9 of the 10 unsupported evaluation questions produce an
  insufficient-evidence response.
- Model outages produce clear errors rather than fabricated answers or
  success-shaped responses.

Calibrate evidence sufficiency against the evaluation set rather than hard-coding
an arbitrary similarity threshold.

### Milestone 5: Connect Chartroom

**Status: complete - implemented and locally verified on 2026-10-09.**
Production deployment remains part of Milestone 6, not this acceptance.

Chartroom already implements the required library, upload, ingestion/retry,
deletion, question, and source-inspection surfaces. This closeout adds
repeatable Chromium acceptance in `chartroom/e2e/workspace.e2e.ts`, using the
current source-first backend image with `qwen2.5:7b`, the current Chartroom
Nginx image, and real local PostgreSQL, MinIO, Qdrant, Ollama, and worker.
The normal running containers were not replaced.

Verified:

- **2/2 browser scenarios passed**. The real workflow creates a collection,
  uploads Markdown, waits for ready ingestion, reloads/reconnects, asks a
  question, displays the exact supported answer, opens the versioned source,
  cancels then confirms deletion, and waits for completed cleanup entirely
  through the UI. An API check additionally verifies the deleted tombstone.
- The second scenario injects 503 and 401 responses to verify visible errors,
  recovery, insufficient evidence with zero citations, and removal of private
  UI after unauthorized access. These injected responses prove UI behavior,
  not a real model outage.
- **27/27 UI/client/proxy tests passed**, including ingestion and cleanup retry,
  polling errors and explicit-refresh recovery, source-load errors and retry,
  loading/empty states, upload limits, cancellation, and stale-response isolation.
- Lint, TypeScript checking (including browser tests), frontend build, container
  build, and Compose configuration validation passed.
- Regenerated API types match the committed Lighthouse OpenAPI contract.
- Uploaded acceptance documents were deleted, including the first failed
  test's fixture; empty test collections remain because no collection-delete
  API exists. Temporary test containers were stopped after verification.

See [browser acceptance instructions](../chartroom/README.md#browser-acceptance-milestone-5).
Browser tokens are supplied at runtime; no traces, screenshots, or videos are
recorded. Test output is gitignored. Adding the browser runner also reported
an existing high-severity `source-map-js` development-dependency advisory;
that dependency was not introduced or changed by this work and is not resolved
by Milestone 5.

Replace representative local data with Lighthouse APIs.

Build:

- Document library.
- Upload form.
- Ingestion status and retry controls.
- Delete confirmation and cleanup status.
- Question interface.
- Citation panel that opens the source passage.
- Loading, empty, unauthorized, and dependency-error states.

Use Lighthouse's versioned OpenAPI contract for typed integration instead of
maintaining independent DTO definitions.

Acceptance criterion: a user can complete upload, ingestion, question-answering,
source inspection, and deletion entirely through the UI.

### Milestone 6: Production deployment

**Status: complete - production deployed and accepted on 2026-10-18.**

The [production runbook](PRODUCTION-RUNBOOK.md) defines the explicit rollout/rollback gates.
Local development and production are the only environments.
Delivered foundations:

- Infrastructure-owned PostgreSQL with a retained local-path PVC, a separate
  non-superuser app role, and gated local-only runtime/credential provisioning.
  Production originals use filesystem storage on Infrastructure's retained
  `lighthouse-originals` local-path RWO PVC (`MARINA_STORAGE_BACKEND=filesystem`,
  `MARINA_STORAGE_PATH=/var/lib/marina/originals`), mounted by the Fleet
  initialization hook, API, and worker with fsGroup 65532 and `Recreate`.
  Existing S3-backed data migrates only through the explicit, verified offline
  `copy-originals` step; initialization refuses metadata whose originals are
  missing from the selected storage.
- Fleet runtime ConfigMap/Secret references and staged suspension for Lighthouse
  and Chartroom. Production runs only API/worker; no mutable/new unverified image
  digest was inserted. Shipyard retains Fleet release policy during promotion.
- Release-controlled pre-install/pre-upgrade migration/storage initialization
  Job using the same immutable image and runtime references as the application.
  Failure blocks rollout; manual provisioning does not run migrations.
- Owner-scoped `/v1/operations` counts and queue age alongside role readiness
  and sanitized worker/dependency logs.
- Credential-safe metadata/original backup, empty-target restore, and
  hash-checked vector rebuild commands; no destructive in-place restore.
- Real isolated database/bucket/vector restore rehearsal passed, preserving
  exact citations, deterministic point count, and a source-first answer.
- Real disposable PostgreSQL initialization and migration passed with an app
  role that cannot act as superuser or create roles/databases.
- Local authenticated smoke and both Chromium UI scenarios passed.
  Local development authentication is not evidence of public Authentik acceptance.
- Candidate API process restart preserved originals/metadata/vectors and exact
  sources. An isolated unavailable S3 endpoint and missing model produced visible
  503 errors while liveness stayed healthy; restoring valid configuration
  recovered exact answers. Shared backing services were not interrupted.
- **169 backend/evaluation tests passed**, one expected SQLite-only concurrency
  skip; **27 frontend tests**, **2 Chromium scenarios**, **36 Shipyard tests**
  and **4 Infrastructure bootstrap guard tests** passed. Both app contracts,
  Helm lint/rendering, Kustomize rendering, Bash syntax, and frontend/backend
  builds passed. Tests ran against SQLite and real local PostgreSQL where applicable.

Live production evidence recorded on 2026-10-18:

- Shipyard-promoted immutable Lighthouse and Chartroom images are deployed through
  Fleet; Flux Kustomizations and HelmReleases are Ready and unsuspended.
- PostgreSQL and the retained originals PVC are Bound. API and worker pod
  recreation preserved collection/document/version/chunk/job counts, exact
  document and original hashes, and returned all readiness checks to healthy.
- A quiesced production backup restored into UUID-named temporary PostgreSQL,
  filesystem, and Qdrant targets. Metadata, original hashes, chunks, citations,
  and vector counts matched; no jobs failed, and all temporary targets were
  removed. The exported archive is encrypted off-node, its key is protected by
  Windows DPAPI, and HMAC, archive hashes, and a decryption round trip passed.
- Idle API/worker usage was 18m/112MiB and 5m/112MiB. A representative
  source-first request completed in 9.035 seconds with the API at 54m/214MiB;
  GPU memory was 7,775/12,227 MiB. Originals storage was 7% used with 898GiB
  available. Failed jobs, pending jobs, and queue age were all zero.
- Anonymous public Marina and Chartroom requests receive the expected Authentik
  challenge. The operator confirmed the authenticated Chartroom workflow and
  final public-session validation, then accepted the milestone.

Milestone 6 is accepted. Future releases still repeat the relevant deployment,
public-session, backup, recovery, and resource gates; this acceptance is not a
permanent waiver for later images or configuration changes.

Deploy a small collection of non-sensitive homelab documents first.

Before admission:

- Provision backing services and local-only credentials.
- Configure authentication and private object access.
- Verify backup and restore for metadata and originals.
- Document vector-index rebuild from authoritative stored data.
- Test role-specific readiness and resource sizing.
- Add ingestion-failure, queue-age, and dependency-health visibility.
- Run an end-to-end smoke test through the public authenticated route.
- Document rollback, including database migration compatibility and data retention.

Acceptance criteria:

- The full workflow survives application restarts and a tested restore.
- Unauthorized requests are rejected.
- Model and backing-service failures remain visible and recoverable.

## 3. Repository ownership and deployment handoffs

| Repository | Responsibility |
| --- | --- |
| [Lighthouse](https://github.com/nnvuf-homelab/lighthouse) | Backend, migrations, workers, retrieval, API contracts, and local development |
| [Chartroom](../chartroom) | UI and typed API integration |
| [Infrastructure](https://github.com/nnvuf-homelab/infrastructure) | Backing services, Authentik integration, storage, local-only secrets, ingress, and monitoring |
| [Fleet](https://github.com/nnvuf-homelab/fleet) | Application releases, runtime settings, and admission |
| [Shipyard](https://github.com/nnvuf-homelab/shipyard) | Build, test, and promotion pipeline; change only for concrete contract gaps |

The existing [Lighthouse deployment contract](https://github.com/nnvuf-homelab/lighthouse/blob/main/shipyard.yaml) declares multiple
roles. Check whether generated deployments support the configuration, named
secret references, and routing needed by this workflow before expanding tooling.

Keep application, Fleet, and Infrastructure changes as explicit handoffs.
Image reference, namespace, port, hostname, and health endpoints must agree.
Validate contract changes against Shipyard's authoritative schema.

Never put application source into Infrastructure or commit secret values.
Provision cluster dependencies through Infrastructure and promote app releases
through Shipyard and Fleet. Treat live mutations as separately authorized
operations; static validation does not establish live workload readiness.

## 4. Suggested schedule

For one developer, this is a sequencing estimate, not a delivery commitment.
Advance only when each milestone's acceptance criteria pass.

| Period | Outcome |
| --- | --- |
| Week 1 | Architecture decisions, local dependencies, and persistent upload API |
| Week 2 | Reliable ingestion, indexing, and deletion |
| Week 3 | Search evaluation and grounded answers |
| Week 4 | Chartroom integration |
| Week 5 | Authentication hardening, restore test, and production deployment |

## 5. Follow-on work

Production acceptance is complete. The follow-on milestones are planned in
detail in the [expansion plan](EXPANSION-PLAN.md):

1. Harbormaster: expose read-only search and source retrieval.
2. Trawler: add controlled URL ingestion behind a tested SSRF boundary.
3. PDF support: add extraction and page citations.
4. Operational hardening: scheduled backups, alerting, and rehearsed rollback.

Knowledge graphs, autonomous crawling, write-capable MCP tools, and learning
agents stay deferred until a concrete use case justifies them.

## First implementation task

Start with Milestone 1: a tested Lighthouse upload, list, status, and delete
workflow running locally with durable storage. Build this foundation before
adding a chat endpoint backed by temporary data.
